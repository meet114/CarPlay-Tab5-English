# Experimental software authentication materials

These three files are committed with the repository for optional iAP2 software-auth builds:

| File | Purpose |
|---|---|
| [identity.pk8](identity.pk8) | PKCS#8 P-256 private key; this is a publicly shared experimental identity, not a confidential secret |
| [certificate.p7b](certificate.p7b) | PKCS#7 certificate container used for phone authentication |
| [certificate.der](certificate.der) | DER certificate extracted from that container, used for key matching and signature self-checks |

Clone the current repository to obtain them. If downloading separately, keep the filenames and the `secrets/mfi/` directory layout. Do not save web-page HTML as key files. File sizes and SHA-256 digests are in [SHA256SUMS](SHA256SUMS).

From the repository root:

```sh
(cd secrets/mfi && shasum -a 256 -c SHA256SUMS)
source <esp-idf-v5.5.3>/export.sh
python3 scripts/fetch_openh264.py
./scripts/build.sh --software-auth
```

The materials come from the DiPlay v0.2.21 official Release APK `assets/offline-mfi` tree. APK SHA-256, provenance boundaries, build steps and device evidence are in [software authentication setup](../../docs/software-authentication-setup.md). The DER certificate was extracted from the PKCS#7 container. These materials are not relicensed under this project's GPL source license; bundling them does not imply Apple/MFi authorization or certification, and does not guarantee continued acceptance on every device or iOS version.

This directory name follows the existing build path. The identity listed here is already public and must not be used where private-key confidentiality or a device-unique identity is required. Other personal/device credentials, NVS pairing records and logs are outside these files and must still not be committed.
