# Tab5 Software Authentication Setup

Updated: 2026-11-14. This repository already includes experimental iAP2 software authentication materials; after cloning you can choose an auth-enabled build directly. This identity is a publicly shared experimental identity. It does not represent Apple/MFi certification and does not guarantee ongoing compatibility with every phone or iOS version.

## Obtaining the in-repo materials

Inspect the [authentication materials directory](../secrets/mfi/README.md) directly. There is no need to download or extract an APK separately:

| File | Purpose | Size |
|---|---|---|
| [identity.pk8](../secrets/mfi/identity.pk8) | PKCS#8 P-256 private key | 67 bytes |
| [certificate.p7b](../secrets/mfi/certificate.p7b) | PKCS#7 certificate container | 607 bytes |
| [certificate.der](../secrets/mfi/certificate.der) | DER certificate for self-checks | 558 bytes |

Prefer cloning or updating the current `main` so the files land under the `secrets/mfi/` path the build already expects. You can also download the raw files from the links above; keep the filenames and directory layout. Do not save GitHub HTML pages in place of the binary files.

Verify from the repository root:

```sh
(cd secrets/mfi && shasum -a 256 -c SHA256SUMS)
```

All three entries should report `OK`. The [checksum list](../secrets/mfi/SHA256SUMS) confirms a complete download; it is not evidence of earlier provenance or authorization.

## Build and flash

With ESP-IDF v5.5.3, from the repository root:

```sh
source <esp-idf-v5.5.3>/export.sh
python3 scripts/fetch_openh264.py
./scripts/build.sh --software-auth
```

This command explicitly sets `TAB5_SOFTWARE_AUTH=ON`, and CMake embeds the three files above into `build/carplay_tab5.bin`. Builds without `--software-auth` explicitly set the flag OFF, so software authentication stays disabled even if the materials directory exists. A no-auth firmware cannot enter CarPlay directly.

The local release-format build command remains:

```sh
python scripts/release.py --auth local
```

Artifacts land under `artifacts/releases/v1.0.0/local-auth/`. `local-auth` is the existing build-variant name; it does not mean the identity inside is still secret. For upgrades on devices that already have firmware, write only the application region so pairing, volume, and brightness are preserved:

```sh
python -m esptool --chip esp32p4 -p <serial-port> write_flash 0x10000 build/carplay_tab5.bin
```

When switching phones: unplug → Clear pairing on the waiting page → confirm → reboot Tab5 → connect an unlocked phone and confirm Trust / Allow CarPlay. Clearing pairing does not remove the authentication materials embedded in the firmware; the authentication identity and the NVS phone-trust records are different data.

## Provenance and disclosure scope

The authentication provider was ported from Carplay-S3 `src/vendor/auth_software_provider.c/h` at fixed commit `79a9f169f275b283f2909c5a326ed03c716c0d02`, retaining GPL-3.0-or-later. The authentication materials themselves are not re-licensed under GPL.

Per this project's 2026-11-07 acquisition and verification notes, the materials came from the DiPlay v0.2.21 official Release APK under `assets/offline-mfi`. The APK SHA-256 checked at that time was `916c0708beb1022d4ea9678ff6a46dfb6d234f707837a2cbd80d615e8e2bb2e3`; the APK was not executed. The PKCS#7 container held a single certificate; after extracting DER, the certificate and private key were verified to match. This commit reuses those three files and does not include the APK, phone pairing records, or device logs.

Upstream traces the materials to public Carlinkit firmware; this project independently verified only the APK and its assets, and has no independent proof of earlier provenance or redistribution rights. Provenance notes: [Carplay-S3 authentication materials provenance](https://github.com/xmz28/Carplay-S3/blob/79a9f169f275b283f2909c5a326ed03c716c0d02/docs/认证材料来源.md), [DiPlay v0.2.21 Release](https://github.com/shihabal3amri/DiPlay/releases/tag/v0.2.21).

**The private key is public in the current repository and can be extracted from auth-enabled firmware. Do not treat it as secret or as a device-unique identity.** Publishing it in-repo changes how this project manages materials; it does not mean Apple/MFi authorization was obtained. Other private keys, personal identities, phone pairing records, and capture logs must stay local. Git only admits the three materials listed in this directory plus the accompanying notes and checksum list.

## Runtime checks and on-device evidence

After obtaining entropy, the Lockdown worker thread initializes the provider, checks certificate extensions, public/private key match, and a raw P-256 signature self-test; signing calls use a mutex. `IAP2_AUTHENTICATED` is recorded only when the phone actually returns `0xAA05` and the certificate, signature, and ACK sequence are valid. A passing software-signature self-test is not the same as a full successful CarPlay connection.

The first verification on 2026-11-07 only confirmed that the current phone accepted the identity, then stopped because NCM was not yet configured. Later, v1.0.0 completed short tests of real video, music, navigation, and settings; see the [release notes](releases/v1.0.0.md) and [execution log](execution-log.md). Those results do not prove compatibility with every phone/iOS version, and they are not official certification.

## Difference from the v1.0.0 first-release snapshot

The published `v1.0.0` tag, source attachment, and no-auth development firmware keep the first-release contents; that source snapshot does not include authentication materials. The materials were added on current `main` after the first release. To obtain the materials with the project, use current `main`, or download these three files from the links on this page into the same-named directory in older source trees. This materials-and-docs commit does not rewrite existing tags or firmware attachments.
