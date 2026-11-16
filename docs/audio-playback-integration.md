# Tab5 Audio Playback Integration (Phase One)

> Historical phase record. v1.0.0 retained pause/resume and packetized encryption regressions; output moved to dual-stream mix and settings-page volume; boot self-test was removed. Current features, on-device results, and limits are governed by the [v1.0.0 release notes](releases/v1.0.0.md) and the [execution log](execution-log.md).

Goal: play a single iPhone music PCM stream on Tab5 through the existing CarPlay/AirPlay path.
Currently there is only offline test and build evidence; that alone cannot establish that the phone has sent music to Tab5 or that the speaker has produced sound.
Known under-test combination for this round: Tab5 + iPhone X, iOS 16.7.16; board silicon revision still to be read.

## Changes in This Round

- After roughly 100 ms of continuous empty playback buffers, clear the old RTP timestamp anchor and re-prebuffer when packets resume.
  This covers the case where the sender's timestamp does not keep advancing during local silence playback after pause, avoiding permanent rejection of late packets.
  Short packet loss still fills silence on the original timeline; connection-generation checks and mono/stereo and endian conversion remain.
- Extract the existing RTP fixed header / ChaCha20-Poly1305 decrypt into a testable module; decrypt failures must not expose PCM.
  Protocol layout and key derivation stay as implemented; synthetic tests cannot prove real iOS protocol compatibility.
- On output open, explicitly set local volume and unmute; log codec open, first real PCM write, and periodic status.
- Schedule the optional boot audio self-test to finish before USB init so it does not operate codec/I2S at the same time as the playback task.

## Build and Offline Checks

```sh
./scripts/check_cli.sh
./scripts/test_host.sh
source <esp-idf-v5.5.3>/export.sh
./scripts/test_crypto.sh
python3 scripts/fetch_openh264.py
./scripts/build.sh
./scripts/test_decoder.sh
```

`test_host.sh` covers short packet loss at 44.1/48 kHz, sustained underrun, pause timestamp recovery, re-prebuffer,
and rejection of an old connection. `test_crypto.sh` adds synthetic encrypted RTP→decrypt→PCM buffer→byte-by-byte output regression,
covering mono/stereo, endianness, AAD/ciphertext/tag/nonce tampering, wrong key, truncation, and illegal formats.

## On-Device Steps

1. Follow the pre-flash flow in [Device Acceptance](device-acceptance.md) to confirm silicon revision, partitions, and backup. Do not assume a default
   v1.x build can flash onto any Tab5. Preserve NVS, authentication identity, and the partition table.
2. With no phone connected, enable `TAB5_AUDIO_TEST` via `idf.py menuconfig`; before USB init it
   plays a short tone and samples microphone peaks. Confirm sound by ear; record `Tone write complete` and `MIC` logs.
   Codec success and changing mic peaks do not equal CarPlay audio or microphone-uplink acceptance.
3. `TAB5_AUDIO_VOLUME` controls local output volume when the playback stream opens; default 30%, 0 is mute.
   This round does not implement phone volume commands. For normal integration, disable `TAB5_AUDIO_TEST`.
4. Connect iPhone X / iOS 16.7.16, keep the verified auth/USB configuration, and play ordinary music.
   Save the full serial log from before connect through play, pause, resume, and unplug; do not save voice or keys.
5. Use the table below to locate the earliest missing or failing stage. On format rejection, record the format value; do not work around errors by falsely declaring
   unimplemented audio capabilities.
6. Pause then resume, change tracks, and re-plug; confirm output recovers and does not play audio from the previous connection.
   Play continuously for 30 minutes; note stutter, pops, A/V drift, and queue/error counts.

| Log | What it proves / where to look |
|---|---|
| No `SETUP requested` | No supported audio SETUP seen yet; check session setup, capability negotiation, and phone output routing |
| `SETUP rejected` | Unsupported field/format, output busy, or codec open failed; debug by specific reason |
| `OUTPUT_OPEN` | Codec open, volume, and unmute calls succeeded; does not prove audio was received |
| `ready ... data=... control=...` | Audio ports established; does not prove the phone sends audio to those ports |
| `PCM_FIRST` | First authenticated PCM accepted by the playback buffer |
| `auth_errors` / `format_errors` | Auth decrypt failure vs packet structure or sample-alignment mismatch |
| `sink_drops` | Decrypt succeeded but playback layer rejected; combine with queue and rejected_packets |
| `OUTPUT_FIRST_PCM` | First real PCM write to codec returned success; still needs ear confirmation |
| `OUTPUT_STATUS` | Real frames written, queued frames, underruns, rejected packets, rebuffer count; about every 3 seconds |
| `OUTPUT_WRITE failed` | Codec/I2S write failed; current output stopped; stream must be re-established |

## Still Incomplete

- On-device sound, pause/resume, and long-term stability acceptance.
- Multi-stream mix, navigation ducking music, phone volume control, sample-clock drift compensation.
- Microphone uplink, Siri/call bidirectional audio, and AEC.

These capabilities remain unverified/unimplemented; do not expand `/info` input or audio format declarations.
