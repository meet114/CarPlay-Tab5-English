# CarPlay-Tab5 Technical Implementation Plan

Original plan date: 2026-11-05. The following preserves the initial-stage design and the verification results from that time; it is not a statement of current implementation status.

2026-11-06 update: Per user request, prioritize completing the first four offline development items and defer on-device debugging. Current code, interfaces, tests, and pending adaptation limits are governed by [Four Deliverables](four-deliverables.md). USB Host is already installed by the application filter/daemon; BSP handles power. Full protocol and media components are integrated; phrases such as "pending port" and "no serial port detected" in this document are historical. Microphone/AEC are outside these four items.

## 1. Inputs and Technical Decisions

The input is the user-provided *CarPlay-Tab5 Practice Plan*. That document is a requirements and design reference; its historical feasibility judgments, examples, and operating steps are not treated as already-verified facts.

Use a native ESP-IDF C/C++ project. Complete the M0/M1 board baseline first; port the full protocol only after hardware and authentication paths are verified. Do not flash S3 firmware directly onto P4, and do not copy authentication identity from upstream Releases.

Pin ESP-IDF v5.5.3, espressif/m5stack_tab5 1.3.2, LVGL 9.3.0, and esp_lvgl_port 2.6.2; remaining dependencies are pinned in dependencies.lock. Use a locally installed, executable stable SDK; avoid introducing an unverified mainline SDK. The BSP component declares compatibility with IDF >=5.4. Compilation found that the automatically selected esp_lvgl_port 2.9.0 uses DSI callbacks this SDK does not provide, so 2.6.2 is pinned explicitly. USB uses the BSP's unified start interface; do not start a second Host daemon. Actual build results are in the execution log.

User confirmed the panel is **ST7121**. M5Stack's public version history states a switch from ST7123 to ST7121 on 2026-04-28. The downloaded BSP 1.3.2 already includes the ST7121 init table: probe touch address 0x55, read register 0x0000; FW=1 identifies ST7121, FW=3 identifies ST7123. Use BSP auto-detection; do not force an unknown version. LCD reset uses output for low and switches back to input pull-up for high, matching M5Stack's I2C stability requirements.

## 2. Verification Results and Boundaries

- Official Tab5 configuration: ESP32-P4, 32 MB PSRAM, 16 MB Flash, MIPI-DSI panel, ES8388 output and ES7210 input. Reuse the BSP's I2C, expander, display, and audio drivers.
- The USB-A power switch is managed by the BSP's IO expander; do not treat BSP_USB_EN as a main-controller GPIO, and do not promise that the API's limit_500mA parameter actually limits current (the board docs mark it unused). Call bsp_usb_host_start directly and let the BSP select its adapted USB controller. Do not copy S3 GPIO19/20 PHY settings; BSP macros themselves must still be judged against the P4 schematic, and do not burn additional eFuses. The USB component's default peripheral_map=0 selects the high-speed controller on P4; actual link speed is measured from enumeration logs.
- P4 H.264 is encode hardware; decode still requires software. Follow the upstream-fixed OpenH264 2.6.0 portable C/C++ decode path; do not misuse the H.264 encoder API. High Profile / reference-frame requirements must be checked against the real stream.
- Upstream public source lacks a complete authentication identity; a successful build does not mean an iPhone accepts authentication. Upstream audio is PCM receive diagnostics only; there is still no evidence of I2S playback or microphone uplink passing.
- No Tab5 USB serial port was detected in this round, so no on-device conclusions for enumeration, display, sound, or performance are available.

P4 v0/v1 and the v3 series have SDK-documented hardware incompatibility. Default build config selects v1.0–1.99 at 360 MHz; also verify compilation for v3.1–3.99. Final selection must follow the device chip info; the ST7121 panel model cannot infer silicon revision.

## 3. Architecture and Data Flow

```
USB Host (BSP/IDF)
  -> descriptor probe [current implementation]
  -> USBMUX -> Lockdown/TLS -> CarKit/iAP2/auth [pending port]
  -> NCM/NTB16 -> esp_netif/lwIP IPv6
  -> AirPlay RTSP + Event + Timing + Screen + Audio
      -> bounded H264 queue -> OpenH264 -> RGB565 -> MIPI DSI
      -> authenticated RTP PCM -> jitter buffer -> ES8388
Touch BSP -> orientation/letterbox mapping -> HID -> Event channel
ES7210 -> capture/AEC -> microphone uplink [phase two]
```

USB client callbacks only record events; switch devices and parse cached descriptors in the client task context. Do not claim interfaces yet, do not send Apple mode-switch commands, and do not request phone trust. Parse interface numbers, alternate settings, class/subclass/protocol, and endpoint address/type/packet size; distinguish Apple VID from a real iPhone identity (VID alone cannot prove the model). On disconnect, release handles; record connect/disconnect counts and minimum free heap.

Subsequent protocol states: DETACHED -> ENUMERATED -> USBMUX -> TRUST_PENDING -> LOCKDOWN -> IAP2_AUTH -> NCM_READY -> SESSION -> STREAMING. Allocate a generation per connection; async completions carry the generation, and stale completions must not update a new session. On disconnect, stop producers first, then cancel transfers, wait for callback reclaim, release interfaces, and finally close the device. Each stage has a timeout; retries use exponential backoff with a cap, and do not loop-erase pairing NVS.

Authentication interfaces express MISSING, READY, REJECTED, and ACCEPTED separately; mere presence of files must not set ACCEPTED. Authentication material stays out of the repo and logs; Lockdown pairing identity and CarPlay authentication identity are stored separately.

## 4. Upstream Port Checklist

Baseline: xmz28/Carplay-S3 commit `79a9f169f275b283f2909c5a326ed03c716c0d02`. The original checkout is at reference/Carplay-S3 (reference only; not part of the current build).

| Module | Strategy | Verification requirements |
|---|---|---|
| usb_probe.cpp | Replace Host init; keep config/interface matching logic | Config switch, re-enumeration, unplug races |
| usbmux.cpp / mux_channels.cpp | Migrate protocol core; check threads and length bounds | Short packets, sticky packets, oversized frames, sequence wrap |
| lockdown_client.cpp | Adapt IDF TLS/NVS | Trust rejection, lock screen, invalid pairing records |
| iap2_client.cpp / vendor/iap2_core.c | Keep protocol/license; connect independent auth provider | ACK/retransmit/timeout/missing identity |
| ncm_host.cpp / ntb16.h | Verify USB DMA memory and esp_netif | NTB16 overrun, ring chain, reconnect |
| airplay_* | Check mbedTLS API and IPv6 scope | RTSP split packets, key isolation, integrity verification |
| openh264_sw | Portable C/C++; verify RISC-V/atomics/alignment | Synthetic High Profile/LTR pixel CRC + on-board timing |
| display.cpp / menu.cpp | Replace with Tab5 BSP; keep protocol-agnostic parts | Rotation, double buffering, async refresh ownership |
| PCM receiver | Add playback adapter; do not copy an already-implemented output | Clock drift, reordering, underrun silence, reconnect queue clear |

Upstream root license is GPL-3.0; components have separate provenance records. Future source copies must retain corresponding licenses and THIRD_PARTY_NOTICES; the current board-level application is independently written and does not import protocol source.

## 5. Media Design and Budget

The phone's logical layout size is determined jointly by pixel count and the declared physical aspect ratio: logical width is the larger of pixel width and "pixel height × physical aspect ratio"; icons are about 60 logical points. Therefore you can declare 16:9 while only having the phone encode 1024×192; the UI still lays out as 1024×576 and is then stretched by PPA. Default is 1024×192/16 FPS; optional 1024×240/12, 1024×360/7, 1024×576/5, 852×480/5, 640×384/5, 480×288/6, 320×192/8. These are experimental tiers, not performance commitments. The CarPlay home screen is a fixed 4×2 grid; logical width below 640 causes icon overlap, and there is no 6- or 4-icon layout. Default is a single main screen; do not decode two streams at once for now.

One 1280×720 RGB565 frame is 1,843,200 bytes; double buffering is 3,686,400 bytes. One 480×288 I420 frame is 207,360 bytes. OpenH264 internal reference frames, bitstream, and task stacks are additional; 32 MB PSRAM is not entirely available for frame buffers. USB DMA must use driver-allocated interfaces; do not arbitrarily hand ordinary PSRAM pointers to DMA.

The bitstream queue is dual-limited by byte count and frame count (starting budget 512 KiB / 3 access units, adjusted under later stress tests). On congestion, drop complete access units and request IDR; do not arbitrarily trim H.264 bytes. Display buffers must not be reused before refresh completes. After landscape transform, subtract letterbox offsets, then map into the negotiated pixel range; ignore presses on black bars, clamp active drags that leave bounds, and ensure release events are delivered.

Create the codec from the actually negotiated format; support 16-bit LPCM first, with explicit sample rate / channels / endianness. For example, 48 kHz stereo is 192,000 bytes per second; a 60 ms buffer is 11,520 bytes. Fill packet loss with silence; an independent high-priority task writes I2S, and network callbacks must not block waiting for playback. The microphone baseline only tracks peaks and does not persist voice; AEC and uplink sessions are accepted separately later.

## 6. Execution and Stage Gates

| Milestone | This round / later delivery | Pass criteria |
|---|---|---|
| M0 | Buildable BSP firmware; display/touch and optional audio self-test | Real device displays correctly; touch at four corners correct; short tone / mic peak valid |
| M1 | USB descriptor probe | iPhone recovers across 20 plug/unplug cycles; no sustained heap decline |
| M2 | USBMUX/Lockdown/iAP2, auth provider | Legal identity path clear; real authentication success logs |
| M3 | NCM/IPv6/RTSP | Real media received and correctly validated |
| M4 | OpenH264 + display | Average/slowest frame per tier; 30 minutes without crash |
| M5 | HID and PCM playback | Tap/drag; music/navigation healthy for 30 minutes |
| M6 | Microphone/AEC/stability | Siri, calls, 2-hour and power-fault regression |

Current work can complete source, builds, and host boundary tests; M0/M1 on-device acceptance depends on a connected device. Without authentication material, do not package later stages as complete. Before flashing, verify chip/Flash/partition and backup plan; do not assume existing devices use this project's partition table.

## 7. References (verified 2026-11-05)

- [M5Stack Tab5 product, pinout, and schematic entry](https://docs.m5stack.com/en/core/Tab5)
- [Official Tab5 BSP source and versions](https://github.com/espressif/esp-bsp/tree/master/bsp/m5stack_tab5)
- [ESP-IDF v5.5.3 USB Host](https://docs.espressif.com/projects/esp-idf/en/v5.5.3/esp32p4/api-reference/peripherals/usb_host.html)
- [Espressif video technology selection: no H.264 hardware decode](https://docs.espressif.com/projects/esp-video-components/en/latest/esp32p4/Technology_Selection.html)
- [Carplay-S3 pinned commit](https://github.com/xmz28/Carplay-S3/tree/79a9f169f275b283f2909c5a326ed03c716c0d02)
- [Apple MFi technology and development process](https://mfi.apple.com/en/how-it-works)
