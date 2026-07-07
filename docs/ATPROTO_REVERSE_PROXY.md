# ATProto Reverse Proxy / fedproxy Tunnel Topology

How fedproxy.com and xrpc.fedproxy.com expose services at subdomains through
the did-key-relay dispatcher. These are transport-level details — not part of
the core compute-contract protocol, but they are how guests reach the network
and how requesters reach guests in the reference implementation.

## SSH Tunnel Topology

Guest VM/container never opens a public port. All traffic flows through the
did-key-relay dispatcher. The requester reaches the guest exclusively through
this relay — SSH `ProxyCommand` over a WebSocket tunnel.

```
requester SSH client
  ProxyCommand websocat --binary wss://<service>--did-plc-<key>.fedproxy.com
    → fedproxy dispatcher (did-key-relay, routes by SNI subdomain)
      → fedproxy-client (guest side, dialed outbound through relay)
        → websocat ws-l:127.0.0.1:8080 → sshd 127.0.0.1:22
```

## FQDN convention

```
<SERVICE>--did-plc-<DID_PLC_KEY>.fedproxy.com
```

- `SERVICE` — the VM's role/name (from `compute.vm.role`), flattened (no `.` or `:`)
- `DID_PLC_KEY` — the requester's DID PLC key (bare, without `did:plc:` prefix), flattened

Example: `my-cool-role--did-plc-5svqtrhheairglgiiyvutzik.fedproxy.com`

## Guest cloud-init (what Bob injects)

The `compute.vm.user_data` is a `#cloud-config` YAML document. Bob's
compute provider enriches it with OIDC provisioning (nonce + prove
script) and RBAC grants, then hands it to the container/VM backend. The
default transport (fedproxy) installs:

- **sshd** — key-only root login, loopback-only (`ListenAddress 127.0.0.1`)
- **websocat** — bridges `ws-l:127.0.0.1:8080 → tcp:127.0.0.1:22`
- **fedproxy-client** — fronts websocat, dials the relay outbound, registers the guest's FQDN
- Alternative: **tunnel-subscriber** (did-key-relay xrpc subscriber) replaces
  fedproxy-client + websocat with a single Deno process that speaks the relay
  tunnel protocol directly

### Default cloud-config (fedproxy transport)

```yaml
#cloud-config
packages:
  - openssh-server
  - jq
  - curl

disable_root: false
ssh_pwauth: false

write_files:
  - path: /root/.ssh/authorized_keys
    owner: root:root
    permissions: '0600'
    content: |
      <requester's ed25519 public key>

  - path: /etc/ssh/sshd_config.d/10-websocat.conf
    owner: root:root
    permissions: '0644'
    content: |
      ListenAddress 127.0.0.1
      PermitRootLogin prohibit-password
      PasswordAuthentication no

  - path: /etc/systemd/system/websocat.service
    owner: root:root
    permissions: '0644'
    content: |
      [Unit]
      Description=websocat ws→sshd bridge (fronted by fedproxy-client)
      After=network-online.target sshd.service ssh.service
      Wants=network-online.target

      [Service]
      Type=simple
      User=root
      ExecStart=/usr/local/bin/websocat --binary ws-l:127.0.0.1:8080 tcp:127.0.0.1:22
      Restart=always
      RestartSec=5

      [Install]
      WantedBy=multi-user.target

  - path: /etc/systemd/system/fedproxy-client.service
    owner: root:root
    permissions: '0644'
    content: |
      [Unit]
      Description=FedProxy Client Service
      After=network-online.target
      Wants=network-online.target

      [Service]
      Type=simple
      User=root
      WorkingDirectory=/root
      Environment="SERVICE=<vmName>"
      Environment="HANDLE=<didPlc>"
      Environment="PORT=8080"
      Environment="ATPRP_URL=https://<xrpcRelaySubdomain>.<relayHost>"
      Environment="AUTH_PLUGIN=oidc"
      Environment="MARKET_ACCEPT_JSON_PATH=/root/secrets/publicdomainrelay.com/market/accept.json"
      ExecStart=/usr/local/bin/fedproxy-client
      Restart=always
      RestartSec=5

      [Install]
      WantedBy=multi-user.target

runcmd:
  - systemctl daemon-reload
  - systemctl enable --now ssh || systemctl enable --now sshd
  - systemctl enable setup-websocat.path
```

### Alternative: tunnel-subscriber transport

Replaces fedproxy-client + websocat with a single Deno process:

```yaml
#cloud-config
packages:
  - openssh-server
  - jq
  - curl
  - unzip

write_files:
  - path: /etc/systemd/system/tunnel-subscriber.service
    owner: root:root
    permissions: '0644'
    content: |
      [Unit]
      Description=did-key-relay xrpc tunnel subscriber (ssh-over-relay)
      After=network-online.target sshd.service ssh.service
      Wants=network-online.target

      [Service]
      Type=simple
      User=root
      Environment="JSR_URL=http://<jsrUrl>/"
      Environment="DENO_DIR=/var/lib/deno"
      ExecStart=deno run -A jsr:@publicdomainrelay/hono-did-key-relay-tunnel-subscriber \
        --dispatcher-host <dispatcherHost> \
        --aud-host <audHost> \
        --private-key-hex <privateKeyHex> \
        --target-host 127.0.0.1 --target-port 22
      Restart=always
      RestartSec=5

      [Install]
      WantedBy=multi-user.target

runcmd:
  - systemctl daemon-reload
  - systemctl enable --now ssh || systemctl enable --now sshd
  - systemctl enable --now tunnel-subscriber.service
```

## Service endpoints on bidder

The bidder exposes these XRPC service endpoints (advertised via its
`did:web` document, proxied through the relay):

| NSID | Method | Purpose |
|------|--------|---------|
| `com.publicdomainrelay.temp.market.submitRfp` | XRPC procedure | Requester pushes RFP to bidder |
| `com.publicdomainrelay.temp.market.submitAccept` | XRPC procedure | Requester pushes Accept to winning bidder |
| `com.publicdomainrelay.temp.market.submitEvent` | XRPC procedure | Requester sends lifecycle events (vm.delete, etc.) |

## Bidder discovery (how requester finds bidders)

1. **Relay index** (`listReposByCollection` on `market.offering`) — primary
2. **Firehose watch** (subscribeRepos / Jetstream filtered to `market.offering`) — live complement
3. **Vouch graph** (`sh.tangled.graph.vouch`) — social-trust allowlist
4. **Manual** (`extraBidderDids` / `denyBidderDids` in contract options)

## Record signatures

All requester-authored records (`market.rfp`, `market.accept`) carry inline
badge.blue attestations (`network.attested.signature`). The attestation
key is published in the author's DID document. The bidder verifies
signatures before dispatching to callbacks. The requester verifies the
receipt's signature + remote proof (receipt binds to the accept record)
before trusting the provisioned guest.

## References

- https://github.com/publicdomainrelay/atproto-reverse-proxy — fedproxy-client (guest-side tunnel agent)
- https://github.com/publicdomainrelay/publicdomainrelay — monorepo: did-key-relay, hono-bidder, request-vm-ssh, cloud-init-common
