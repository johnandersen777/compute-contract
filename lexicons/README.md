# Compute Contract Lexicons

[atproto lexicon](https://atproto.com/specs/lexicon) schemas for the compute
contract record types.

The schemas are pre-stable and use the `temp` infix convention:
`com.publicdomainrelay.temp.<name>`. The path on disk mirrors the NSID
(each segment is a directory, the last segment is the filename).

Once a schema stabilizes, it is promoted to `com.publicdomainrelay.<name>`
and evolved additively over time. Genuinely incompatible breaks bump the
short name (`<name>V2`, `<name>V3`, ...).

| NSID                                       | Path                                                |
| ------------------------------------------ | --------------------------------------------------- |
| `com.publicdomainrelay.temp.ccrfp`         | `com/publicdomainrelay/temp/ccrfp.json`             |
| `com.publicdomainrelay.temp.ccb`           | `com/publicdomainrelay/temp/ccb.json`               |
| `com.publicdomainrelay.temp.ccbap`         | `com/publicdomainrelay/temp/ccbap.json`             |
| `com.publicdomainrelay.temp.ccba`          | `com/publicdomainrelay/temp/ccba.json`              |
| `com.publicdomainrelay.temp.ccr`           | `com/publicdomainrelay/temp/ccr.json`               |
