# First Four Offline Development Deliverables

Updated: 2026-11-06. Scope is the four items the user asked to develop first, with on-device debugging deferred and batched. Code is integrated into the application; build and host-test evidence is not equivalent to phone authentication, real playback, or performance acceptance. On-device work remains paused.

## 1. Session Management and Simulation

`main/core/session.c` defines USB enumeration, trust, Lockdown, authentication, network, media, stop, and failure states; `app_session.c` is advanced by the USB owner task, and worker threads only post events carrying a connection generation. Stale events are ignored; missing auth material or auth rejection stops retries; communication timeouts use 1/2/4 second backoff, at most three times. After cancel, wait for USB callbacks and protocol tasks to exit before releasing resources.

`usb_probe.c`, `mux_channels.cpp`, `lockdown_client.cpp`, and `iap2_client.cpp` are wired to real events. The auth client obtains certificates and signatures via auth_port, verifies signature ACK, phone acceptance, and identification results, and starts the local media service after waiting for NCM IPv6. There is no real provider by default; when the NCM interface is configured as -1, do not assume upstream phone configuration is correct.

Tests separately cover the state core, the real app_session adapter layer, and the real iAP2 client simulating success and five failure classes; simulated provider/NCM/services cannot prove end-to-end authentication with a real phone.

## 2. NCM and RTSP

`ncm_host.cpp` uses the CDC Union descriptor to associate control and data interfaces, supports 64/512-byte bulk endpoints, NTB16 fragment reassembly and a bounded send queue, and connects esp-netif/lwIP IPv6. On disconnect, wait for outstanding DMA callbacks; do not free transfer buffers early.

AirPlay RTSP, pairing, control, timing, screen, and audio components are compiled and linked. `airplay_service.c` owns identity, entropy sources, and service lifecycle; stop the service before closing media. RTSP parsing has length limits and rejects illegal headers, overflowing bodies, and continued parse after error. Encrypted control and screen data use Mbed TLS integrity checks.

The existing phone observation config remains USBMUX config 3. NCM config switch, device-specific control requests, and real throughput need on-device adaptation; the current driver is not a general-purpose compatible implementation for arbitrary NCM devices. Host tests cover descriptors, NTB16, RTSP, plist, encrypted control fragment through RTSP parse, and screen HKDF/truncation/tamper/replay.

## 3. Video and Touch

`media_routes.cpp` connects authenticated H.264 config/AUs to `video_output.cpp`. The queue dual-limits 3 AUs and 512 KiB; after losing the reference chain, reset the decoder and request IDR via the event channel. Connection generation isolates old frames and old drop markers.

Pinned OpenH264 v2.6.0 portable decoder; after I420 packing, PPA performs color conversion, scale, and rotation into the panel frame buffer (not via LVGL); decode and display run in parallel. Default request is 1024×192 at 16 FPS; parse and rebuild run on two cores (`CONFIG_TAB5_VIDEO_PROFILE_HEIGHT`). The phone lays out at 16:9 then compresses vertically; PPA stretches back to full screen. Tiers above 320×192 are experimental—fall back to 320×192 when trial cannot keep up or is undecided. Max output size check is 1280×720; that does not mean decode performance or internal memory budget for malicious SPS has been measured.

Touch subtracts letterbox; first press on black bars is ignored; drag beyond bounds is clamped; lift and cancel emit release. `hid_output.cpp` posts single-finger 12-byte reports on an independent queue; at send time, verify epoch under the control-channel lock so old-session taps do not enter a new session.

Real decode host regression covers a synthetic High Profile first frame and 32-frame LTR per-frame pixel CRC; the combined path is screen codec→AU queue→decode→touch/HID. Fake color images were not used as a substitute for decode verification. The USB-off offline demo loops synthetic content at 4 FPS and outputs local touch; the UI clearly labels Simulation.

## 4. Audio Interface and UI

Authenticated RTP PCM writes into a timestamp jitter buffer supporting 44.1/48 kHz, 16-bit, mono/stereo, converting network endianness; handles reordering, duplicates, wrap, and underrun silence. 60 ms prebuffer, max 4800 sample frames. An independent task writes the codec in 10 ms blocks; the network thread does not wait for audio playback.

Current output supports one PCM stream; a second SETUP returns failure. There is no multi-stream mix, clock-drift compensation, microphone uplink, or AEC. Real ES8388 listening quality, A/V sync, and long-term stability remain to be measured.

`app_ui.c` provides a real LVGL status page, error reasons, simulation marker, and video canvas—not a static mock. Host framebuffer rendering covers disconnect, missing auth, simulation playback, and clear screen. UI is English; no Chinese font was introduced.

## Reproduction and Acceptance Evidence

From the repo root, source ESP-IDF v5.5.3/export.sh first:

| Check | Command | Evidence boundary |
|---|---|---|
| Default P4 firmware | `./scripts/build.sh` | Compile/link, no flash |
| Host core and adapter regression | `./scripts/test_host.sh` | ASan/UBSan, virtual clock/transport stubs |
| Real H.264 decode | `./scripts/test_decoder.sh` | Fixed synthetic bitstream, per-frame CRC |
| Real crypto and protocol combination | `./scripts/test_crypto.sh` | Mbed TLS + production cipher/screen/RTSP; no phone socket session |
| Real UI render | `cmake -S tests/ui -B artifacts/ui-build && cmake --build artifacts/ui-build && ./artifacts/ui-build/render` | LVGL host framebuffer, no DSI hardware |

Standalone offline config uses artifacts/sdkconfig.offline with CONFIG_TAB5_OFFLINE_DEMO=y:

```sh
idf.py -B artifacts/offline-build -DSDKCONFIG="$PWD/artifacts/sdkconfig.offline" build
```

Default artifact is build/carplay_tab5.bin; demo artifact is artifacts/offline-build/carplay_tab5.bin. Local config, test artifacts, and evidence live under artifacts and are not part of source. Formal releases must still be rebuilt; these artifacts must not be treated as distribution firmware adapted to every Tab5/iPhone.

## Todos When Resuming On-Device Work

1. Keep existing pairing data; confirm target iPhone descriptors, NCM config, and interface numbers.
2. Connect a usable auth provider; verify the full real iAP2/identify/StartSession/RTSP path.
3. Verify on-board High Profile decode, touch HID, single-stream audio, and media disconnect order.
4. Measure FPS, latency, CPU/PSRAM per tier, 20 plug/unplug cycles, and at least 30 minutes of playback; adjust tiers and buffers from results.

The on-device todos above were not executed in this round. Existing Lockdown pairing success does not substitute for CarPlay authentication success.

Final result: the default/demo firmware builds, host regression, real decode, crypto protocol combination, and UI render above all passed. Firmware size and SHA-256 are in artifacts/offline-delivery/firmware.json; build and regression logs are in the same directory.
