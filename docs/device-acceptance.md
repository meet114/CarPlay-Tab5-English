# Device Acceptance

## Before flashing

1. Connect Tab5 to the development host over USB-C; connect only Tab5 first—do not connect the iPhone. Record the ST7121 marking on the back of the board.
2. Read chip revision, Flash size, and existing partitions. The default build currently targets P4 v1.x; there is also build verification for v3.1–3.99, but the hardware revision is not yet known. Do not use forced-flash parameters.
3. Save a full existing Flash backup and its SHA-256; the backup may contain credentials—keep it local. Confirm this firmware's factory partition layout and recovery method before flashing.
4. This project has no automatic flash script. Flashing, full-chip erase, eFuse writes, C6 updates, or SD operations are not being performed at this stage.

## M0 display / touch

- Serial output should show `BRINGUP` and the firmware IDF version.
- BSP should report `Discovered board version 3 (LCD ST7121, Touch ST712x, FW 1)`. This “board version 3” is the BSP screen version, **not ESP32-P4 silicon revision 3**.
- Observe RGB color bars, landscape text, and 40% backlight. If the screen is corrupted/black or probing does not match, capture the full power-on log.
- Touch the four corners and center; verify coordinate direction and range; one log line each for press and release. There is currently no CarPlay HID feedback.
- If display init blocks further debugging, disable display diagnostics in menuconfig and verify USB alone.

## M0 audio (optional build)

- Enable TAB5_AUDIO_TEST and start short-tone playback; `Tone write complete` only proves the write API returned—a person must confirm actual sound.
- MIC logs should show read=OK; peak levels should change when speaking quietly. Peaks do not prove each of the dual microphones works individually, and they do not prove AEC/Siri.
- Audio errors must be recorded from codec/I2S logs; soak and DMA faults are not yet covered.

## M1 USB

1. Power Tab5 independently and stably; connect the iPhone to Tab5 Host with a data-capable USB-A cable; no hub on the first round.
2. Observe `CONNECTED`, VID/PID, speed enum value, and IF/EP descriptors. Apple VID 05ac is vendor identification only—do not claim CarPlay is connected from that alone.
3. Unplug should produce `DISCONNECTED`; reinsertion should re-enumerate. Run 20 consecutive cycles; record cable, iPhone/iOS, per-cycle counts, and heap usage.
4. Compare free internal heap and PSRAM under a stable idle baseline; minimum free heap is a historical low-water mark and does not recover—do not treat it as sole evidence of a leak.
5. On enumeration failure, check IDF USB logs, the cable, and VBUS current; software parameters do not guarantee current limiting or adequate phone-charging power.
6. 30 minutes powered with repeated plug/unplug: no panic, watchdog, or sustained memory decline. This program does not send Apple-specific control requests; absence of a phone trust prompt is expected.

## Record template

| Item | Result |
|---|---|
| Date / firmware SHA-256 | Pending |
| P4 revision / Flash / screen probe | Pending |
| iPhone model / iOS / data cable | Pending |
| USB speed / VID / PID / IF / EP | Pending |
| 20 connect/disconnect counts | Pending |
| Start/end internal heap, PSRAM | Pending |
| Color bars / orientation / touch corners | Pending |
| Short tone / mic peak | Pending |
| VBUS / current / 30-minute anomalies | Pending |

Advance to M2 only after M0/M1 pass on a real device and the usable authentication path is clear. Items without measured records stay “Pending”.
