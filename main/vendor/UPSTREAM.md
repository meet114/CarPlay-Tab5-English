# iAP2 source provenance

This tree is ported from the pinned
[xmz28/Carplay-S3 commit](https://github.com/xmz28/Carplay-S3/tree/79a9f169f275b283f2909c5a326ed03c716c0d02).
The upstream core retains DiPlay / ESP32 CarPlay Web Relay contributor copyright
and GPL-3.0-or-later notices.

The Tab5 adaptation targets ESP32-P4/ESP-IDF, keeps link-boundary, ACK,
retransmission and parse tests, and requests certificates/signatures through the
`components/auth_port` interface. This project does not register a production
auth provider in the no-auth path, does not copy or embed auth private
keys/certificates into source by default for that path, and documents release
firmware separately. Host mock providers are for tests only and must not be used
for real phone authentication. Local changes and licenses are summarized in
`../../THIRD_PARTY_NOTICES.md`.
