# Third-party source

`main/usbmux.cpp`, `main/usbmux.h`, `main/lockdown_reply.h`, `main/mux_channels.cpp`, `main/mux_channels.h`,
`main/lockdown_client.cpp`, `main/lockdown_client.h`, and `main/lockdown_xml.h` are adapted from
[xmz28/Carplay-S3](https://github.com/xmz28/Carplay-S3/tree/79a9f169f275b283f2909c5a326ed03c716c0d02),
commit 79a9f169f275b283f2909c5a326ed03c716c0d02. Source files specify GPL-3.0-or-later;
license text is retained at `licenses/Carplay-S3-GPL-3.0.txt`.

Tab5 modifications: accept 64/512-byte bulk endpoints, expose a C ABI and periodic
status, rename the client label, add single-phone pairing persistence in the existing
NVS namespace `tab5_pair` without erase/migration, and retain TLS through the CarKit/iAP2 session. Multi-stream transport is included
for the pairing worker. CarKit/iAP2 progression uses an explicit auth provider. XML helper routines are extracted for host tests.
No external authentication identities or release firmware are imported. The pairing
build has been flashed with explicit user authorization; pairing, TLS QueryType and
reboot identity reuse have been verified on the connected phone.

The resulting firmware includes GPL-covered code; redistribution must comply with
applicable GPL source/license obligations. This notice does not relicense third-party
BSP/SDK libraries, whose own notices remain in their packages.


## iAP2 link diagnostic

`main/vendor/iap2_core.c` and `.h`, `main/iap2_client.h`, and
`tests/iap2_core_test.c` are copied from the same pinned Carplay-S3 commit.
The core retains the DiPlay contributors and ESP32 CarPlay Web Relay contributors
copyright notices and GPL-3.0-or-later terms. Upstream provenance is retained in
`main/vendor/UPSTREAM.md`. `main/iap2_client.cpp` now adapts the upstream authentication, identification and
media-session control flow through a provider API; no real credentials are included. The CarKit StartService block is adapted from the
upstream Lockdown client, with validation of the optional SSL flag.

## Offline protocol core (2026-11-06)

`main/core/ntb16.h`, `main/core/airplay_rtsp_codec.c/h` and
`tests/test_ntb16.cpp` are ported from the same pinned Carplay-S3 commit
`79a9f169f275b283f2909c5a326ed03c716c0d02` and retained under GPL-3.0-or-later.
See `main/core/UPSTREAM.md`.

## OpenH264 and synthetic video tests (2026-11-06)

Cisco OpenH264 v2.6.0 (commit `652bdb7719f30b52b08e506645a7322ff1b2cc6f`)
is used under BSD-2-Clause; the license is in `components/openh264_sw/LICENSE`.
The fetch tooling, component CMake adaptation and `tests/assets/high_fixture.bin`,
`high_ltr.bin`, `high_ltr_crc.h` come from the same pinned Carplay-S3 commit and
retain that project's GPL-3.0-or-later provenance. Test assets are synthetic
bitstreams with no user media. Reference values come from upstream FFmpeg
checks; this project also compares OpenH264 decode frame-by-frame and does not
treat the reference values alone as pass evidence.

## NCM and AirPlay protocol components (integration in progress)

`main/ncm_host.cpp/h`, `components/airplay_*` (control, timing, screen,
media_bridge, session, pairing, rtsp), `components/net_port`, `time_port`,
`media_bridge` and `auth_port` come from the same pinned Carplay-S3 commit
`79a9f169f275b283f2909c5a326ed03c716c0d02` and retain GPL-3.0-or-later notices.
The protocol port described in this section does not itself include auth
credentials; the later experimental auth materials are covered under the
software-auth provider note below. When no provider is registered, auth_port
explicitly returns unsupported. Local NCM uses a CDC Union association, supports
high-speed endpoints, and fixes allocation-failure reclaim and data-endpoint
stop behavior. Copying and compiling protocol components does not mean services
are running; runtime paths and validation status are recorded in the execution
log.

## AirPlay local identity and Ed25519

`airplay_identity_port`, `airplay_pairing_store` and `airplay_software_provider`
come from the same pinned Carplay-S3 commit. Template plists contain no auth
private keys. `components/ed25519` retains Orson Peters' zlib-style license
(`components/ed25519/LICENSE`) with original copyright and provenance files.
Local changes place AirPlay receiver identity and controller records in the
existing NVS namespaces `tab5_ap_id` and `tab5_ap_pair`, separate from Lockdown
`tab5_pair`.

`tests/test_airplay_bplist.c` and `tests/test_airplay_screen_codec.c` are also
ported from the pinned Carplay-S3 commit for host boundary tests against this
project's compiled protocol sources. `configure_tab5_info.py` is a local script
that normalizes and checks capability templates.

The full `main/iap2_client.cpp` control flow plus `main/core/iap2_params.h` and
`tests/test_iap2_params.cpp` come from the same pinned upstream commit. Local
changes use Tab5 identity, explicit NCM interface setup, a generic auth provider
and local service lifecycle without copying auth credentials into source by
default.

## Optional iAP2 software-auth provider

`components/auth_software_provider` comes from the same pinned Carplay-S3 commit
`src/vendor/auth_software_provider.c/h` and retains GPL-3.0-or-later. Local
additions include optional builds, a signing mutex, clearing of sensitive
temporary buffers and a real raw-signature verification self-check. Experimental
identity materials are kept with the project under `secrets/mfi/`, sourced from
DiPlay v0.2.21 APK `assets/offline-mfi`; the DER certificate is extracted from
the PKCS#7 container. The materials are not relicensed as GPL source. This
project has not independently confirmed earlier provenance or redistribution
authorization scope. Provenance, checksums and build boundaries are in
[software authentication setup](docs/software-authentication-setup.md).

`components/openh264_sw/cabac_decoder_esp.cpp` and `cabac_bin_inline.h` are modified copies of OpenH264 v2.6.0 `cabac_decoder.cpp` (BSD-2-Clause), changed to use 32-bit arithmetic.
