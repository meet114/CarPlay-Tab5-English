# Ed25519 implementation provenance

This component vendors the portable Ed25519 implementation from
[`orlp/ed25519`](https://github.com/orlp/ed25519) at revision
`b1f19fab4aebe607805620d25a5e42566ce46a0e`.

Upstream C sources are kept as-is under the zlib license; see
[LICENSE.txt](LICENSE.txt). The component uses only key generation, sign and
verify sources. Seeds are produced through the platform adapter using the
ESP-IDF hardware RNG. The original upstream description is retained in
[UPSTREAM_README.md](UPSTREAM_README.md).
