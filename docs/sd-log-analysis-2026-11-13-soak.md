# 2026-11-13 Music + Navigation Long-Run Log Analysis

Conclusion: **Cannot yet declare fully normal: one USB disconnect not manually triggered by the user still occurred, but the older permanent USB-task hang did not reproduce.** The phone re-enumerated afterward, video recovered in about 17 seconds, then ran continuously for about 49 minutes until the user manually unplugged. The navigation voice stream-ID fix remains effective.

## Data and firmware verification

- Device MAC matches the existing Tab5. After reading back the factory partition and extracting the valid application image, SHA-256 is `cb916322a6c038226f325aebb05197050025233df8f83ee5553ff37cee3581d3`, byte-identical to the previous release.
- The card-log prefix `560beefc0` is the ELF SHA-256 prefix for that same release image, not another firmware version.
- Exported `LOG00022`–`LOG00034`, 13 files, totaling 2,250,668 bytes. Every file passed both original-length and on-device SHA-256 checks.
- The first serial export truncated one block of LOG00029; the second pass also had partial truncation. That file was reassembled from complete, mutually consistent blocks across both passes; the whole-file SHA-256 matches the device. Core LOG00030 was complete with matching hashes in both passes. No unverified incomplete file was used for conclusions.
- This card-read program does not start the log-to-card writer; it only opens logs with `rb`. Configuration forbids formatting after a mount failure. After export, the original image was flashed back, write verification passed, and boot was normal; local `build/carplay_tab5.bin` was also rebuilt and confirmed restored to `cb916322`. NVS, identity material, and the partition table were not changed.
- Evidence directory: `artifacts/device-2026-11-13/soak-readback/`. File hashes and reassembly sources are in `sdcard-manifest.json`; raw logs are under its `sdcard/` subdirectory.

## Separating this run from earlier tests

- LOG00025 `usb_progress_timeout`, and LOG00028 TX/RX stalled plus three ncm_recovery events, belong to earlier controlled fault injection and must not be counted as failures of this long run.
- LOG00029: about 26 minutes of silent screen mirroring on the release build, 22,399 frames decoded, video rejects and decode errors both 0, no runtime USB disconnect.
- **LOG00030: this music + navigation run**; the last status line in the file is at 3597.519 seconds after boot, about 59 minutes 58 seconds. Screen mirroring ran to about 53 minutes 57 seconds; the phone was then unplugged, and the remaining ~6 minutes are idle.
- LOG00031 and later short logs have no phone connection; they include reconnecting the device to the PC and boots around this card-read session, not continuous mirroring test segments.
- Cold-start `other_watchdog` in LOG00030/31 matches this board’s existing power-on behavior; that name alone does not prove a watchdog restart during the run. This round has no panic save records and no usb_guard timeout records.

## One non-manual disconnect

User confirmation: only the final disconnect was a manual unplug; the first one was not.

| Time after boot | Evidence and interpretation |
|---|---|
| 04:23.404 | USB STATUS normal, `usb_hal_recovered=0`; audio and video were still progressing before/after; internal heap ~48 KB. |
| 04:25.730 | USBMUX and NCM both report `RX status=1`; ncm_fault recorded. |
| 04:26.133 | USB device already detached; cleanup completed; USB probe did not hang. |
| 04:26.733 | `NCM recovery skipped: the phone detached by itself`. The app did not perform recovery power-cycle. |
| 04:35.933 | Phone reappeared on the bus; its mode switch went through another normal enumeration. |
| 04:36.431 | NCM interface restored. |
| 04:42.595 | Post-reconnect decode frame counts rising again: about 16.87 seconds from fault to observable video recovery. |
| 53:57.216 | Second disconnect; user confirmed manual unplug at end of test; cleanup normal, likewise skipped redundant power-cycle. |

Evidence locations: LOG00030 lines 1242–1262, 1434, 1619, 10042–10063.

This differs from the earlier LOG00016 pattern of “TX filled, RX watchdog, permanent wait on endpoint halt”:

- No TX/RX stalled; send-queue sample max was only 1, never reached 8.
- USB STATUS continued throughout; max sample interval 10.1 seconds.
- ncm_fault occurred twice; both times app power-cycle was skipped after the device truly detached; ncm_recovery = 0.
- usb_guard did not reboot the device; USB cleanup and subsequent reconnect were normal.

**Root cause of the first disconnect remains undetermined.** Existing logs show the device disappeared after a USB transfer failure; they cannot distinguish a phone-side reset, a contact/power glitch, or a host-controller port error. `usb_hal_recovered=0` only means the “abnormal error without CHHLTD” counter did not increase; it does not rule out ordinary USB transfer or port errors. Blame cannot be assigned to cable or phone from this alone, nor can this round be treated as a zero-disconnect pass.

## Navigation, music, and display

- Navigation type 101: **100 SETUP, 100 normal stop, 0 SETUP rejects**. 10 before the first disconnect, 90 after reconnect—exceeding the old 32-slot stream-ID table limit; each stream `bad=0`.
- Main audio: 4 segments—3 music, 1 speechRecognition. Music received 423,881 packets total; speechRecognition received 334. Navigation received 118,611 packets total. Auth/format errors, reordering, audio submit rejects, and lock-wait failures were not recorded as non-zero.
- Around 17:01, the music stream and the then-active navigation stream each showed `lost=2`, i.e. RTP sequence missing 2 packets each; these are per-stream cumulative values, not another +2 on every later print. Music underrun cumulative 708 frames, navigation underrun 120 frames, counted at each stream’s own sample rate (music 48 kHz, that navigation 8 kHz)—about 14.75 ms and 15 ms respectively. They did not keep growing afterward. Whether a short audible glitch was heard needs listening confirmation; this cannot be called strictly zero packet loss.
- Brief underruns also appear at the end of both disconnects (668 and 6634 frames), in the window after the stream had already ended while the player was still cleaning up; those should not be counted as stutter during continuous playback.
- Video cumulative submit 44,715 frames, decode 44,652 frames; **decode errors 0, UI failures 0, but 9 rejects**. After each reject, a keyframe request and recovery were logged; the whole CarPlay session did not exit because of them.
- The 9 reject logs carry `ESP_ERR_NO_MEM`. Current `main/media_routes.cpp` maps `tab5_video_submit()` returning false to this error; here it increased in sync with VIDEO’s rej counter, so the error name alone must not be taken as proof of malloc failure or full heap exhaustion. Concrete reject conditions need further logging of queue depth, byte occupancy, frame size, and keyframe state.
- During music playback, internal free heap was mainly ~47–48 KB, sample low 46,807 bytes; after reconnect the early median was ~47.6 KB and later still ~47.9 KB, with no sustained downward trend. PSRAM was likewise stable. This supports “no obvious sustained leak observed,” not a complete leak audit.
- There were also 2 known I2S disable noise lines, and one IO-expander input read failure around 21:13; continuous normal operation followed, with no corresponding USB disconnect.

## Follow-up recommendations

Prioritize USB fault-source diagnostics, then re-run soak testing: record root-port disconnect/error/overcurrent type, failing channel and endpoint, raw interrupt bits and CHENA, plus onboard USB 5V operating timing. The next run can then distinguish a real disconnect from a controller error instead of inferring from RX status alone.

Also add concrete video-reject reasons and queue watermarks to locate these 9 brief rejects. There is currently no evidence supporting increasing heap size or lowering recovery timeouts first.

This round only read and analyzed logs; business firmware was not modified. The device remains on the `cb916322` release; the “permanent task hang” did not reproduce, but the “first non-manual disconnect” still needs further localization.
