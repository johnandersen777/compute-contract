# Compute Contract

> https://john.leaflet.pub/3mletyxaie22o

- Alice, Bob, and Eve are on the network
- Alice wants to issue a Compute Contract Request For Proposal (CCRFP)
  - Alice's CCRFP will state she wants an OpenCode instance
- Bob has plenty of builder machines
- Eve wants to know what Alice is doing
- Alice has vouched for Bob
- Alice has denounced Eve
- Alice creates a CCRFP manifest
- Alice makes her CCRFP manifest available to the network
- Bob and Eve each issue a Compute Contract Bid (CCB) against the CCRFP
- Alice's policy engine sees that she's denounced Eve and vouched for Bob
- Alice issues a Compute Contract Bid Accept (CCBA) against Bob's CCB.
- Alice issues a x402 payment to Bob per info provided in his CCB.
  - Using the CCBA AT URI and CID to the CCB's stated CCR endpoint.
- Bob issues a Compute Contract Receipt (CCR) over the CCRFP, CCB, and CCBA
  - The CCR references the CCRFP, the CCB, and the CCBA.
- Bob builds to the CCRFP manifest's spec

## TODO

- Bob makes a Compute Contract Event (CCE) and makes it available to the network
  - The event is the `heartbeat=1` event. Indicating the compute has entered a
    state where it is now functional.
- Alice confirms the compute is functional
- Alice issues a Compute Contract Event Accept (CCEA) on the network
- Bob issues a Compute Contract Event (CCE) on the network
  - The event is the `heartbeat=2` event.
- Alice is done using the compute
- Alice issues the Compute Contract Finalize (CCF) event to the network
- Bob issues a Compute Contract Event (CCE) on the network
  - The event is the `heartbeat=0` event. The compute contract is now complete.
- Compensation can be tied to these heartbeat events.

## References

- https://github.com/publicdomainrelay/compute-contract-provider-relay-digitalocean

## Versioning

Schemas are pre-stable and use the `temp` infix:
`com.publicdomainrelay.temp.<name>`. Lexicons for all of the record types
live under [`lexicons/`](./lexicons).

When a schema stabilizes it is promoted to `com.publicdomainrelay.<name>`
and evolved additively. Genuinely incompatible breaks bump the short name
(`<name>V2`, `<name>V3`, ...).

| Short name | Full NSID                                  |
| ---------- | ------------------------------------------ |
| CCRFP      | `com.publicdomainrelay.temp.ccrfp`         |
| CCB        | `com.publicdomainrelay.temp.ccb`           |
| CCBAP      | `com.publicdomainrelay.temp.ccbap`         |
| CCBA       | `com.publicdomainrelay.temp.ccba`          |
| CCR        | `com.publicdomainrelay.temp.ccr`           |

## Data Formats

- Alice CCRFP manifest

```yaml
---
$type: "com.publicdomainrelay.temp.ccrfp"
cpus: 1
mem: '512M'
disk: '10G'
network: '500G'
location:
  country: 'USA'
  region: 'west'
role: 'my-cool-role'
user_data: |
  #cloud-init
  packages:
    - openssh-client
    - python3
  write_files:
    - path: /var/www/8080/index.html
      owner: root:root
      permissions: '0644'
      content: |
        Hello World!

    - path: /etc/systemd/system/python-http@.service
      owner: root:root
      permissions: '0644'
      content: |
        [Unit]
        Description=Simple Python HTTP server on port %i
        After=network.target
        Wants=network.target

        [Service]
        Type=simple
        User=root
        WorkingDirectory=/var/www/%i
        Environment=PYTHONUNBUFFERED=1
        ExecStart=/usr/bin/python3 -m http.server %i --bind 127.0.0.1
        Restart=always
        RestartSec=5
        TimeoutStopSec=10
        StandardOutput=journal
        StandardError=journal

        [Install]
        WantedBy=multi-user.target

    - path: /usr/local/bin/ssh-reverse-tunnel-wrapper
      owner: root:root
      permissions: '0700'
      content: |
        #!/usr/bin/env bash
        set -euo pipefail

        INSTANCE="${1:-}"
        [ -n "$INSTANCE" ] || { echo "Missing instance" >&2; exit 2; }

        # Expect INSTANCE to be SERVICE.HANDLE (HANDLE may contain dots)
        SERVICE="${INSTANCE%%.*}"
        HANDLE="${INSTANCE#*.}"

        if [ -z "$SERVICE" ] || [ "$SERVICE" = "$HANDLE" ]; then
          echo "Instance must be in the form SERVICE.HANDLE (e.g. myname.aliceoa.bsky.social)" >&2
          exit 2
        fi

        REMOTE_SSH="${HANDLE}@fedproxy.com"
        REMOTE_BIND="${SERVICE}:80:127.0.0.1:8080"

        exec /usr/bin/ssh -NnT -p 2222 \
          -i /root/.ssh/id_ed25519 \
          -o UserKnownHostsFile=/dev/null \
          -o StrictHostKeyChecking=no \
          -o PasswordAuthentication=no \
          -o ExitOnForwardFailure=yes \
          -R "${REMOTE_BIND}" \
          "${REMOTE_SSH}"

    - path: /etc/systemd/system/fedproxy@.service
      owner: root:root
      permissions: '0644'
      content: |
        [Unit]
        Description=SSH reverse tunnel for %i (SERVICE.HANDLE -> %i@fedproxy)
        After=network-online.target
        Wants=network-online.target
        StartLimitIntervalSec=60
        StartLimitBurst=5

        [Service]
        Type=simple
        User=root
        WorkingDirectory=/root
        Environment=INSTANCE=%i
        ExecStart=/usr/local/bin/ssh-reverse-tunnel-wrapper "%i"
        Restart=always
        RestartSec=5
        TimeoutStopSec=20
        StandardOutput=journal
        StandardError=journal

        [Install]
        WantedBy=multi-user.target
  runcmd:
  - |
      # NOTE these run as sh! not bash!

      # TODO This should not be using set -x because tokens get logged
      set -x

      ATPRP_URL="https://rp.fedproxy.com"
      # https://pdsls.dev/at://did:plc:5svqtrhheairglgiiyvutzik/com.fedproxy.rbac/3mlewidctvt2n
      HANDLE="johnandersen777.bsky.social"
      DID_PLC_KEY="5svqtrhheairglgiiyvutzik"
      DID_PLC="did:plc:${DID_PLC_KEY}"

      mkdir -p /root/.ssh
      chmod 660 /root/.ssh
      yes | ssh-keygen -t ed25519 -N "" -f /root/.ssh/id_ed25519
      SSH_PUB=$(cat /root/.ssh/id_ed25519.pub)

      # TODO Make sure CCB is ingestable

      URL=$(cat /root/secrets/digitalocean.com/serviceaccount/base_url)
      TEAM_UUID=$(cat /root/secrets/digitalocean.com/serviceaccount/team_uuid)
      ID_TOKEN=$(cat /root/secrets/digitalocean.com/serviceaccount/token)

      SUBJECT="actx:${TEAM_UUID}:plc:${DID_PLC_KEY}:role:my-cool-role"

      SERVICE="$(openssl rand -hex 4)"

      TOKEN=$(jq -n -c \
          --arg aud "api://ATProto?actx=${DID_PLC}" \
          --arg sub "${SUBJECT}" \
          --arg ttl 3600 \
          '{aud: $aud, sub: $sub, ttl: ($ttl | fromjson)}' | \
        curl -sf \
          -H "Authorization: Bearer ${ID_TOKEN}" \
          -d@- \
          "${URL}/v1/oidc/issue" \
          | jq -r .token)

      curl -s \
        -X POST \
        -H "Authorization: Bearer ${TOKEN}" \
        -H "Content-Type: application/json" \
        -d '{
              "repo": "'"${DID_PLC}"'",
              "collection": "com.fedproxy.sshPublicKey",
              "record": {
                "$type": "com.fedproxy.sshPublicKey",
                "key": "'"${SSH_PUB}"'",
                "service": "'"${SERVICE}"'",
                "name": "'"${SERVICE}"'",
                "createdAt": "'$(date -u +"%Y-%m-%dT%H:%M:%S.%3NZ")'"
              }
            }' \
        "${ATPRP_URL}/xrpc/com.atproto.repo.createRecord" | jq

      mkdir -p /var/www/8080
      chown root:root /var/www/8080
      systemctl daemon-reload
      systemctl enable --now python-http@8080.service
      systemctl enable --now "fedproxy@${SERVICE}.${HANDLE}.service"
```

- Alice watches for bids
  - **TODO** Filter by `embed.record.cid && uri` using jq

```bash
timeout 15s uv run ~/src/digitalocean-labs/droplet-oidc-poc/src/workload_identity_oauth_reverse_proxy/firehose_to_ndjson.py | jq 'select(.collection | startswith("com.publicdomainrelay.temp.ccb"))'
```

- Bob CCB

```yaml
---
$type: "com.publicdomainrelay.temp.ccb"
embed:
  cid: "asdlfkjsdlkfjlasdkfqeuhoj134j3lk43lk2j4308j43n4l3n2lk3j4l32"
  uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccrfp/3m21312k9jnkl"
bid:
  cost: 4
  currency: USDC
  frequency: monthly
  prepay: true
  x402:
    base_url: https://compute-contract.johnandersen777.bsky.social.fedproxy.com/ccr/{at}/{cid}
wif:
  issuer_uri: https://droplet-oidc.its1337.com
  to_issue: exchange-custom-droplet-oidc-poc
  token_path: /root/secrets/digitalocean.com/serviceaccount/token
  url_path: /root/secrets/digitalocean.com/serviceaccount/base_url
  url_route: /v1/oidc/issue
  subject: actx:4959ec0923473bf22bddd7bec2caf58a294ee007:plc:{did-plc-key}:role:{role}
```

- Alice chooses and pays

```bash
$ npx awal x402 pay https://spindle-0001.johnandersen777.bsky.social.fedproxy.com/weather
✓ Request completed (HTTP 200)

Response:
{
  "report": {
    "weather": "sunny",
    "temperature": 70
  }
}
$ npx awal auth login johnandersenpdx@gmail.com
$ https://github.com/googleworkspace/cli get emails
$ npx awal auth verify $CODE
# echo npx awal@latest show
# ✓ Wallet window opened
$ npx awal address
EVM (Base): 0x9012310923809128309182903812093801923211
Solana: Fs10238091283091283098109283091283928010101
$ npx awal balance

Base
────────────────────────
USDC    5.00
ETH     0.00

Polygon
────────────────────────
USDC    0.00
POL     0.00

Solana
────────────────────────
USDC    0.00
SOL     0.00
```

- Alice CCBAP (Compute Contract Bid Accept Payment) is the on-chain payment
  receipt that references the CCB Alice paid against:

```yaml
---
$type: "com.publicdomainrelay.temp.ccbap"
embed:
  cid: "7hvb3njk42348nlk4jh5njhlkjhkdfjsdbfsjfje92yh7yhd98sf98d0sus"
  uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccb/js9df8jo2j32l"
txid: "0xabcdef0123456789..."
```

- Alice CCBA (Compute Contract Bid Accept) ties the CCRFP, CCB, and the CCBAP
  (payment receipt) together. The provider's `/ccr` endpoint is fed the CCBA
  AT URI and CID:

```yaml
---
$type: "com.publicdomainrelay.temp.ccba"
embed:
  cid: "asdlfkjsdlkfjlasdkfqeuhoj134j3lk43lk2j4308j43n4l3n2lk3j4l32"
  uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccrfp/3m21312k9jnkl"
bid:
  cid: "7hvb3njk42348nlk4jh5njhlkjhkdfjsdbfsjfje92yh7yhd98sf98d0sus"
  uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccb/js9df8jo2j32l"
payment:
  cid: "dfsknml1823j12k3m1l2jn31288j12k3jkl3n439j41pk32m8sdjfoisdjf"
  uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccbap/3kjsdf98sdf89"
```

- Bob CCR (Compute Contract Receipt) at createRecord response returned from
  payment.base_url on x402 success which resolves to this record:

```yaml
---
$type: "com.publicdomainrelay.temp.ccr"
rfp:
  cid: "asdlfkjsdlkfjlasdkfqeuhoj134j3lk43lk2j4308j43n4l3n2lk3j4l32"
  uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccrfp/3m21312k9jnkl"
bid:
  cid: "7hvb3njk42348nlk4jh5njhlkjhkdfjsdbfsjfje92yh7yhd98sf98d0sus"
  uri: "at://did:plc:bob000000000000000000000/com.publicdomainrelay.temp.ccb/js9df8jo2j32l"
ccba:
  cid: "bafyreiamisq3yqgb4k3tdojmzvvzpuwj46ytwbj672zxhyxxl7t36qadz4"
  uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccba/3mlagijgoeb23"
compute:
  # The IPv4 address of the provisioned compute
  ipv4: '1.1.1.1'
```

## Generic: Marketplace Exchange Wrappers (one level up)

- TODO
  - https://discourse.atprotocol.community/t/tranquil-instance-for-delegated-accounts/850
    - tranquil instance and accounts as an example
    - RFP for VPS
    - RFP for Tranquil on VPS
    - RFP for Account on Tranquil PDS
    - Maybe work backwards with patterns and anti-patterns adhearence baked in
      for downstream.
      - Frank requests account for Agent Charlie
      - Alice sees RFP for Charlie Account and makes RFP for VPS
      - Bob sees Alice RFP and Bids
      - Alice sees Bob's bid and adds her setup fee, then returns her bid
      - There should be a way to pay with a voucher of some kind that is not
        real currency. For Dave may grant a voucher for creation of Agent
        Charlie because Agent Charlie's requisition flow is to be funded from
        the [AT Community Fund](https://discourse.atprotocol.community/t/about-the-community-fund-category/27).
        In this case we need a way for Bob and Alice to say they will do it
        pro-bono for the AT Community Fund's sake. Or to accept indirect payment
        from the fund instead of from Frank directly.
  - https://attested.network/scenarios.html
    - Use attested.network `"$type": "com.atproto.repo.strongRef",` as best practice here
    - Also use attested.network for payments eventually
      - Step 1 for this would be to have the payment strongRef in the CCB reference https://attested.network/brokers.html
  - https://tangled.org/tranquil.farm/tranquil-pds/blob/main/docs/install-kubernetes.md
    - https://www.kcp.io abstraction to spin on CCRFPs
    - https://tangled.org/tranquil.farm/tranquil-pds/blob/main/crates/tranquil-api/src/delegation.rs
  - Also proxy `*.service.handle.fedproxy.com` so to `service.handle.fedproxy.com` so that the service can reverse proxy futher 🐢
- Notes
  - https://zicklag.leaflet.pub/3mjrvb5pul224
  - https://nelind.leaflet.pub/3mljaycxcqc2h
- `opencode export|import`
  - https://gist.github.com/johnandersen777/76d6773f79500f036f989ae9caaa85f0
- fedproxy auto rbac via records similar to ssh keys

```bash
docker model pull hf.co/unsloth/Qwen3.6-35B-A3B-MTP-GGUF:UD-Q2_K_XL

docker run -d --restart=unless-stopped --name llama-mtp-8k-no-reasoning --device /dev/dri --device /dev/kfd     -e HIP_VISIBLE_DEVICES=0 -e ROCR_VISIBLE_DEVICES=0     -v docker-model-runner-models:/models -p 127.0.0.1:12434:12434     --entrypoint /app/llama-server docker/model-runner:mtp     -m /models/bundles/sha256/60b929136fc442800ef3cc2b200e026419c6b30b704c2ae7bf4b4a31957dde72/model/model.gguf     --host 0.0.0.0 --port 12434 -c 131072 -np 1 -ngl 999 --device ROCm0     -fa on     --cache-type-k q8_0 --cache-type-v q8_0     --spec-type draft-mtp --spec-draft-n-max 3 --reasoning-budget 0 --no-mmproj

docker run --rm --network host -u agent -w /home/agent -p 4096:4096 opencode-ubuntu:latest /home/agent/.opencode/bin/opencode serve --port 4096
```

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "llama.cpp/qwen3.6-mtp",
  "provider": {
    "llama.cpp": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "llama-server (local)",
      "options": {
        "baseURL": "https://qwen-0001.johnandersen777.bsky.social.fedproxy.com/v1"
      },
      "models": {
        "qwen3.6-mtp": {
          "name": "Qwen3.6-35B-A3B-MTP-GGUF:UD-Q2_K_XL",
          "limit": {
            "context": 131072,
            "output": 65536
          }
        }
      }
    }
  }
}
```

The `cc`-prefixed records carry compute-specific data for the marketplace exchange. The generic marketplace envelopes below — `rfp`, `bid`, `bid.accept`, `receipt` — wrap that compute-specific payload via strongRefs (`{uri, cid}`), so the same outer protocol can be reused for non-compute marketplaces by swapping the inner `cc*` record for some other domain-specific record type.

Layering:

```
com.publicdomainrelay.rfp          ──strongRef──▶ com.publicdomainrelay.ccrfp
com.publicdomainrelay.bid          ──strongRef──▶ com.publicdomainrelay.ccb
       └── rfp ──strongRef──▶ com.publicdomainrelay.rfp
com.publicdomainrelay.bid.accept   ──strongRef──▶ com.publicdomainrelay.bid
       └── rfp ──strongRef──▶ com.publicdomainrelay.rfp
com.publicdomainrelay.receipt      ──strongRef──▶ com.publicdomainrelay.ccr
       ├── rfp        ──strongRef──▶ com.publicdomainrelay.rfp
       ├── bid        ──strongRef──▶ com.publicdomainrelay.bid
       └── bid.accept ──strongRef──▶ com.publicdomainrelay.bid.accept
```

### Alice RFP (wraps CCRFP)

```yaml
---
$type: "com.publicdomainrelay.rfp.v.0.0.0"
domain: "compute"
payload:
  $type: "com.publicdomainrelay.ccrfp.simple.machine.manifest.v.0.0.0"
  record:
    cid: "asdlfkjsdlkfjlasdkfqeuhoj134j3lk43lk2j4308j43n4l3n2lk3j4l32"
    uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.ccrfp/3m21312k9jnkl"
```

### Bob Bid (wraps CCB, refs RFP)

```yaml
---
$type: "com.publicdomainrelay.bid.v.0.0.0"
rfp:
  $type: "com.publicdomainrelay.rfp.v.0.0.0"
  record:
    cid: "rfpcid000000000000000000000000000000000000000000000000000000"
    uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.rfp/3m21312k9jnkl"
payload:
  $type: "com.publicdomainrelay.ccb.simple.v.0.0.0"
  record:
    cid: "7hvb3njk42348nlk4jh5njhlkjhkdfjsdbfsjfje92yh7yhd98sf98d0sus"
    uri: "at://did:plc:bob000000000000000000000/com.publicdomainrelay.ccb/js9df8jo2j32l"
```

### Alice Bid Accept (refs Bid + RFP)

```yaml
---
$type: "com.publicdomainrelay.bid.accept.v.0.0.0"
rfp:
  $type: "com.publicdomainrelay.rfp.v.0.0.0"
  record:
    cid: "rfpcid000000000000000000000000000000000000000000000000000000"
    uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.rfp/3m21312k9jnkl"
bid:
  $type: "com.publicdomainrelay.bid.v.0.0.0"
  record:
    cid: "bidcid000000000000000000000000000000000000000000000000000000"
    uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.bid/3kjsdf98sdf89"
# Optional inline reference to the compute-specific accept payload, if present.
# When omitted, accept defers fully to the referenced bid's compute-specific terms.
payload:
  $type: "com.publicdomainrelay.ccbap.simple.v.0.0.0"
  record:
    cid: "ccbapcid0000000000000000000000000000000000000000000000000000"
    uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.ccbap/3lkjasdf32j"
```

### Bob Receipt (refs RFP + Bid + Bid Accept, wraps CCR)

```yaml
---
$type: "com.publicdomainrelay.receipt.v.0.0.0"
rfp:
  $type: "com.publicdomainrelay.rfp.v.0.0.0"
  record:
    cid: "rfpcid000000000000000000000000000000000000000000000000000000"
    uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.rfp/3m21312k9jnkl"
bid:
  $type: "com.publicdomainrelay.bid.v.0.0.0"
  record:
    cid: "bidcid000000000000000000000000000000000000000000000000000000"
    uri: "at://did:plc:bob000000000000000000000/com.publicdomainrelay.bid/js9df8jo2j32l"
bid.accept:
  $type: "com.publicdomainrelay.bid.accept.v.0.0.0"
  record:
    cid: "bacid0000000000000000000000000000000000000000000000000000000"
    uri: "at://did:plc:alice0000000000000000000/com.publicdomainrelay.bid.accept/3lkjasdf32j"
payload:
  $type: "com.publicdomainrelay.ccr.simple.v.0.0.0"
  record:
    cid: "dfsknml1823j12k3m1l2jn31288j12k3jkl3n439j41pk32m8sdjfoisdjf"
    uri: "at://did:plc:bob000000000000000000000/com.publicdomainrelay.ccr/3kjsdf98sdf89"
```

### Notes on the abstraction

- `domain` (e.g. `"compute"`) on the outer `rfp` lets policy engines and indexers route to the right marketplace verticals without parsing inner payloads.
- Every cross-record link is a strongRef (`{uri, cid}`) so the chain is content-addressed end-to-end: tampering with any inner record invalidates the receipt.
- The compute-specific quantities (cpus / mem / disk / network / location / `user_data`, cost / currency / x402 base_url, the provisioned ipv4) stay where they are today inside the `cc*` records — the outer envelopes only carry references and routing metadata.
- Non-compute marketplaces (e.g. storage, model inference, bandwidth) reuse `rfp` / `bid` / `bid.accept` / `receipt` unchanged and define their own `xx*`-prefixed payload lexicons.

## Examples

The full flow: create the CCRFP, then the CCB referencing the CCRFP, then the
CCBAP (payment receipt) referencing the CCB, then the CCBA tying CCRFP/CCB/CCBAP
together, and finally hand the CCBA AT URI/CID to the provider's `/ccr`
endpoint.

```bash
# 1. Alice creates the CCRFP
file="examples/data/spin-droplet-0001/0001-ccrfp/request.json"
goat xrpc procedure @pds com.atproto.repo.createRecord - < "${file}" \
  | tee "$(dirname "${file}")/response.json" | jq

# 2. Bob creates the CCB referencing Alice's CCRFP
IN="$(cat examples/data/spin-droplet-0001/0001-ccrfp/response.json | jq -c)"
OUT_OLD="$(cat examples/data/spin-droplet-0001/0002-ccb/request.json | jq -c)"
echo "${OUT_OLD}" \
  | jq --arg uri "$(echo "${IN}" | jq -r '.uri')" \
       --arg cid "$(echo "${IN}" | jq -r '.cid')" \
       '.record.embed.record.uri = $uri | .record.embed.record.cid = $cid' \
  | tee examples/data/spin-droplet-0001/0002-ccb/request.json
file="examples/data/spin-droplet-0001/0002-ccb/request.json"
goat xrpc procedure @pds com.atproto.repo.createRecord - < "${file}" \
  | tee "$(dirname "${file}")/response.json" | jq

# 3. Alice pays Bob via x402 (recorded as CCBAP referencing the CCB)
IN="$(cat examples/data/spin-droplet-0001/0002-ccb/response.json | jq -c)"
OUT_OLD="$(cat examples/data/spin-droplet-0001/0003-ccbap/request.json | jq -c)"
echo "${OUT_OLD}" \
  | jq --arg uri "$(echo "${IN}" | jq -r '.uri')" \
       --arg cid "$(echo "${IN}" | jq -r '.cid')" \
       '.record.embed.record.uri = $uri | .record.embed.record.cid = $cid' \
  | tee examples/data/spin-droplet-0001/0003-ccbap/request.json
file="examples/data/spin-droplet-0001/0003-ccbap/request.json"
goat xrpc procedure @pds com.atproto.repo.createRecord - < "${file}" \
  | tee "$(dirname "${file}")/response.json" | jq

# 4. Alice creates the CCBA referencing CCRFP, CCB, and CCBAP
CCRFP="$(cat examples/data/spin-droplet-0001/0001-ccrfp/response.json | jq -c)"
CCB="$(cat examples/data/spin-droplet-0001/0002-ccb/response.json | jq -c)"
CCBAP="$(cat examples/data/spin-droplet-0001/0003-ccbap/response.json | jq -c)"
cat examples/data/spin-droplet-0001/0004-ccba/request.json \
  | jq \
      --arg ccrfp_uri "$(echo "${CCRFP}" | jq -r '.uri')" \
      --arg ccrfp_cid "$(echo "${CCRFP}" | jq -r '.cid')" \
      --arg ccb_uri "$(echo "${CCB}" | jq -r '.uri')" \
      --arg ccb_cid "$(echo "${CCB}" | jq -r '.cid')" \
      --arg ccbap_uri "$(echo "${CCBAP}" | jq -r '.uri')" \
      --arg ccbap_cid "$(echo "${CCBAP}" | jq -r '.cid')" \
      '.record.embed.record.uri = $ccrfp_uri
       | .record.embed.record.cid = $ccrfp_cid
       | .record.bid.record.uri = $ccb_uri
       | .record.bid.record.cid = $ccb_cid
       | .record.payment.embed.record.uri = $ccbap_uri
       | .record.payment.embed.record.cid = $ccbap_cid' \
  | tee examples/data/spin-droplet-0001/0004-ccba/request.json
file="examples/data/spin-droplet-0001/0004-ccba/request.json"
goat xrpc procedure @pds com.atproto.repo.createRecord - < "${file}" \
  | tee "$(dirname "${file}")/response.json" | jq

# 5. Hand the CCBA AT URI/CID to the provider's /ccr endpoint to spin compute
curl "https://compute-contract.johnandersen777.bsky.social.fedproxy.com/ccr/$(cat examples/data/spin-droplet-0001/0004-ccba/response.json | jq -r .uri)/$(cat examples/data/spin-droplet-0001/0004-ccba/response.json | jq -r .cid)" | jq
```

Read CCRFPs from the firehose

```bash
uv run ~/src/digitalocean-labs/droplet-oidc-poc/src/workload_identity_oauth_reverse_proxy/firehose_to_ndjson.py alice.example.com | jq
```

**request.json**

```json
{
  "repo": "did:plc:alice0000000000000000000",
  "collection": "com.publicdomainrelay.temp.ccrfp",
  "record": {
    "$type": "com.publicdomainrelay.temp.ccrfp",
    "cpus": 1,
    "mem": "512M",
    "disk": "10G",
    "network": "500G"
  }
}
```

```bash
goat get $(goat xrpc procedure @pds com.atproto.repo.createRecord - < request.json  | tee response.json | jq -r '.uri')
```

```json
{
  "repo": "did:plc:alice0000000000000000000",
  "handle": "alice.example.com",
  "seq": 29814868114,
  "time": "2026-05-07T03:26:47.466Z",
  "action": "create",
  "collection": "com.publicdomainrelay.temp.ccrfp",
  "rkey": "3mlabgut5c62t",
  "uri": "at://did:plc:alice0000000000000000000/com.publicdomainrelay.temp.ccrfp/3mlabgut5c62t",
  "record_type": "unknown",
  "record": {
    "mem": "512M",
    "cpus": 1,
    "disk": "10G",
    "$type": "com.publicdomainrelay.temp.ccrfp",
    "network": "500G"
  }
}
```

## Testing

```bash
$ (set -x; for dir in $(ls examples/data/spin-droplet-0001/); do file="examples/data/spin-droplet-0001/${dir}/request.json"; goat xrpc procedure @pds com.atproto.repo.createRecord - < "${file}" | tee "$(dirname "${file}")/response.json" | yq -P; done)
++ find examples/data/spin-droplet-0001/ -type f -name request.json
+ for file in $(find examples/data/spin-droplet-0001/ -type f -name request.json)
+ goat xrpc procedure @pds com.atproto.repo.createRecord -
+ yq -P
++ dirname examples/data/spin-droplet-0001/0001-ccrfp/request.json
+ tee examples/data/spin-droplet-0001/0001-ccrfp/response.json
uri: at://did:plc:5svqtrhheairglgiiyvutzik/com.publicdomainrelay.temp.ccrfp/3mlabxf5xxg2t
cid: bafyreiblivinfkc2hqhoe367b5ggdlieyviyun652g7qxn2p2rl4orfpsq
commit:
  cid: bafyreibszjqfmvk6nbrdofqjkwvrvcdscphoidun6yxrtm55quhaylj62a
  rev: 3mlabxf65sw2t
validationStatus: unknown
```

## Notes

- https://keripy.readthedocs.io/en/latest/ref/getting_started/#receipts
- Should we "just" use TCP
