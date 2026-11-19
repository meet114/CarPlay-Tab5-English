# AirPlay software identity notice

This component provides an AirPlay receiver identity for development and testing. It does not require an external MFi chip, and it does not skip or fake an MFi challenge success. iAP2 MFi challenge-response and AirPlay Pair-Setup/Pair-Verify are independent flows.

## Identity and storage

- On first boot, the component uses the ESP-IDF hardware RNG to generate a 32-byte Ed25519 seed and a random pairing UUID, then stores them in the `airplay_id` namespace of the `pairing` NVS partition. Later boots reuse the same identity so pairing relationships stay stable.
- Public key, device ID, `/info` binary plist and `_airplay._tcp` TXT data are derived or generated from that identity. Generating TXT data is not the same as verified mDNS advertisement. Current `/info` declares primary and secondary screens, default 480×288 / 6 FPS, with Touch HID on the primary screen. The four size/FPS profiles come from `assets/video_profiles.json`; the physical LCD is 800×480.
- The default Pair-Setup code is `3939`, a development default in source. Change it via Kconfig `Wired AirPlay → Pair-Setup code` (`CONFIG_AIRPLAY_PAIRING_CODE`).
- Current firmware does not enable NVS encryption or Flash encryption. Anyone with the device or a Flash image can read the NVS seed. This is an experimental implementation and is not suitable for public deployment or sensitive identity storage.

## Protocol behavior and limits

- Pair-Setup follows SRP-6a with an Ed25519 accessory signature. Pair-Verify validates the paired controller public key and derives session keys. Controller public-key records are stored separately in NVS, up to 8 entries.
- Ed25519 uses the `orlp/ed25519` upstream implementation at revision `b1f19fab4aebe607805620d25a5e42566ce46a0e`, retaining the zlib license and upstream copyright notices under `components/ed25519/`.
- The code follows DiPlay protocol behavior. It is not an Apple-issued or Apple-approved AirPlay receiver identity, and other iOS versions are not guaranteed to accept it. This project has historical device evidence of AirPlay sessions, real video, touch and local menus on the same board and iPhone; see [execution log](../../docs/execution-log.md). That is not proof of general compatibility, mDNS, real music PCM reception, I2S audio output or microphone uplink.

Do not commit non-default Pair-Setup codes in actual use, seeds, private keys or device images to public repositories, issue reports or logs. Before deploying on a shared network, change the default pairing code and evaluate NVS/Flash encryption for the target hardware. iAP2 software-auth materials come from the local `secrets/mfi/` tree and can be extracted from authenticated firmware after embedding. They are distinct from this component's AirPlay identity and from on-device Lockdown pairing identity.

Full Release firmware embeds the same experimental iAP2 identity used in device testing so visitors do not need to import auth files; source trees continue to treat raw materials carefully. That identity is not this component's Ed25519 seed. Materials match the DiPlay local inputs; see [software authentication setup](../../docs/software-authentication-setup.md) for the provenance chain.
