# Tab5 CarPlay Audio Input/Output: Current Analysis and Implementation Plan

2026-11-08. Symptom: tapping QQ Music play on Tab5 routes sound to the phone; Siri and other voice input also use the phone microphone.

## 0. Progress

- 2026-11-08 phases 0–1 complete: after declaring microphone input, the phone immediately established a media stream and QQ Music played from Tab5; Siri established a 24 kHz speechRecognition stream, and silence uplink worked. Root cause confirmed. See `docs/execution-log.md`.

## 1. Conclusion

Most likely cause: the audio capabilities Tab5 declares to the iPhone include output only, not microphone input. The iPhone therefore does not treat Tab5 as a complete CarPlay audio device; all sound stays on the phone, and CarPlay does not appear in the output list.

Evidence:
- **On-device logs**: across 142 sessions in `artifacts/device-*`, the phone only ever established a video stream (`stream_ready type=110`) and never issued an audio stream SETUP (type 100/101) to Tab5, so there is not a single `AUDIO` log line. Receive and playback code has always been present but was never called.
- **DiPlay issue #288**: after a user disabled DiPlay's microphone permission, music and navigation all switched to the phone speaker, and CarPlay disappeared from the output list. Maintainer judgment: without providing audio input, the iPhone does not treat the receiver as a complete audio device; the approach is "still declare input, and uplink silence." This matches our symptom.
- **Declaration comparison**:

| Item | Tab5 now | DiPlay (can play sound) | Carplay-S3 |
|---|---|---|---|
| `features` | `0x615653aee2` | same | same |
| Audio stream kinds | 3 kinds: type 100 default, media, alert | 9 kinds | 8 kinds |
| `audioInputFormats` (microphone) | **none** | compatibility, default, telephony, speechRecognition all present | none (also never received audio) |
| telephony, speechRecognition | none | present | present |
| type 101 secondary audio (nav prompts) | none | present | present |
| type 102/103 (AAC music) | none | 102 AAC-LC, 103 optional | none |
| `extendedFeatures` | enhancedRequestCarUI | also vocoderInfo | — |

Carplay-S3 also does not declare input, and its docs record "no actual music PCM receive obtained," which corroborates the judgment above.

## 2. Protocol Notes (DiPlay as reference)

**Stream types**: 100 main audio, distinguished by `audioType` as default, media, alert, telephony, speechRecognition, compatibility; 101 secondary audio for navigation prompts etc.; 102 main high-quality audio (AAC); 103 main buffered audio (AAC, requires `mainBuffered` feature).

**Downlink (phone → Tab5)**:
- RTP packets encrypted with ChaCha20-Poly1305. RTP header timestamp and SSRC are AAD; packet trailer is a 16-byte tag plus 8-byte nonce.
- Key derived with HKDF-SHA512: input is the Pair-Verify shared secret, salt is `DataStream-Salt<streamConnectionID>`, info is `DataStream-Output-Encryption-Key`.
- Payload is big-endian 16-bit PCM.
- Existing `airplay_audio.c` already implements this part.

**Uplink (Tab5 → phone, microphone)**:
- In audio stream SETUP the phone provides its receive port `dataPort`.
- Key derivation is the same except info becomes `DataStream-Input-Encryption-Key`.
- In the RTP header, payload type equals the stream type (100). Each packet increments sequence by 1, adds this packet's sample count to timestamp, and increments the nonce counter (little-endian) by 1.
- Payload is big-endian PCM. At 48 kHz mono, 10 ms per packet to stay under the 1452-byte MTU; at 16 kHz and 24 kHz, 20 ms per packet is fine, or follow the phone's `framesPerPacket`.

**When to open the mic**:
- telephony and speechRecognition always open the mic.
- default and compatibility open the mic only when the phone requests input: `input` is true, or `dataPort` is provided.
- Start capture on the first downlink packet. Siri-like cases may have no downlink audio; also start capture 500 ms after SETUP.
- During a call, uplink and downlink on the same stream share one sample rate; the phone picks a single `audioFormat` for that stream.

**Format bits**:
- PCM mono: 8k `0x4`, 16k `0x10`, 24k `0x40`, 32k `0x100`, 44.1k `0x400`, 48k `0x4000`; corresponding stereo is the mono bit × 2.
- Opus: `0x10000000` (16k), `0x20000000` (24k), `0x40000000` (48k).
- AAC-LC: `0x400000` (44.1k), `0x800000` (48k).

**Other commands**: `duckAudio` and `unduckAudio` mark the start and end of navigation prompts and require lowering music volume. `requestSiri` wakes Siri; siriAction 2 and 3 are press and release respectively.

## 3. Tab5 Hardware Constraints

- ES8388 (speaker and headphone) and ES7210 (dual-mic ADC) share **one full-duplex I2S**, so TX and RX use the same sample rate. BSP default is 48 kHz, 16-bit, mono.
- Therefore I2S stays fixed at 48 kHz; other rates convert in software: 16k and 24k upsample or downsample by integer factors; media streams declare only 48k to avoid fractional resampling for 44.1k.
- CPU: video decode saturates two cores. Estimate: 48k stereo PCM plus ChaCha decrypt is about 1–2% of one core; microphone uplink is less. Acoustic echo cancellation (AEC) costs much more—consider only for calls, and only after measurement.
- Speaker and microphone hardware have not yet been verified on device (`CONFIG_TAB5_AUDIO_TEST` self-test has no log record).

## 4. Phased Plan

Measure each step separately; roll back if ineffective. Flash only the factory partition; preserve NVS.

### Phase 0: Add logging first (~half day)
- Log every `/command` from the phone with type and parameter key names. Currently only reply 200; types are not recorded.
- Log each stream in SETUP: type, audioType, audioFormat, input, dataPort, framesPerPacket.
- Use this to confirm the effect of each later step.

### Phase 1: Declare input and uplink silence (1–2 days, highest leverage)
- Change `info_*.plist` to DiPlay's declaration set:
  - type 100: compatibility, default, alert, media, telephony, speechRecognition;
  - type 101: compatibility, default;
  - add `audioInputFormats` (PCM mono 16k, 24k, 48k), `audioLatencies`, and `vocoderInfo`.
  - Do not declare 102/103 yet; force media streams to PCM.
- Generate uniformly via `scripts/configure_tab5_info.py`, keeping HID descriptor validation.
- `airplay_audio_start`:
  - Accept 8–48 kHz PCM and type 101;
  - Derive Input key;
  - When the phone requests input, uplink **marked silence packets** to its `dataPort` (log `uplink=silence`).
- Acceptance:
  - CarPlay appears in iPhone Control Center output list;
  - When playing QQ Music, logs show `AUDIO ready type=100 usage=media` and `PCM_FIRST`;
  - Video frame rate and reject counts do not worsen.
- If after declaring input the phone still does not establish an audio stream, compare `/command` and SETUP logs item-by-item with DiPlay captures, then check `modes` and iAP2 identification fields.

### Phase 2: Speaker playback (1–2 days)
- Playback task above the two decode tasks; I2S fixed at 48 kHz.
- Software mix main and secondary audio; upsample 16k and 24k to 48k.
- Jitter buffer 100–150 ms to absorb decode CPU jitter. Follow the phone clock by dropping or inserting samples.
- Handle `duckAudio` and `unduckAudio`; duck music to about 30% during navigation prompts.
- Acceptance: music plays continuously for 10 minutes with near-zero measurable underruns; nav prompts duck music; video metrics do not worsen.

### Phase 3: Microphone uplink (1–2 days)
- Capture from ES7210 at 48 kHz, downsample to the negotiated rate, convert to big-endian, encrypt and packetize for uplink, replacing phase 1 silence.
- Siri: long-press the CarPlay home icon, or add a local button that sends `requestSiri`.
- Acceptance: Siri understands and responds; uplink packet counts are healthy with no over-MTU packets.

### Phase 4: Calls (resource-dependent)
- Echo cancellation can use esp-sr AFE (supports ESP32-P4). Confirm whether ES7210 has a loopback channel for the reference signal; M5 official demos use multi-channel TDM.
- Measure CPU usage first. If needed, lower video frame rate during calls.

## 5. Risks

- Whether the input declaration is the root cause can currently only be inferred from the DiPlay case. Phase 1 success or failure is the verification.
- Different iOS versions may prefer Opus or AAC. This plan declares PCM only; if the phone insists on a compressed format, a decoder must be added.
- Raising audio task priority above video decode may drop video frame rate; compare against phase 1 and 2 data.
- Declaring a microphone that does not actually work leaves Siri and calls silent. The phase 1 silence version is for testing only and must not ship.

## References

- DiPlay: `AirPlayInfoPlist.kt`, `AirPlaySession.kt`, `CarPlayMediaEngine.kt`, `MicrophonePacketizer.kt`, `MicrophoneStart.kt` (https://github.com/shihabal3amri/DiPlay)
- DiPlay issue #288: https://github.com/shihabal3amri/DiPlay/issues/288
- Carplay-S3: `scripts/generate_capabilities.py`, `docs/验证记录.md` (local `reference/Carplay-S3`)
- This project: `components/airplay_session/airplay_audio.c`, `main/audio_output.c`, `components/airplay_software_provider/assets/info_*.plist`
