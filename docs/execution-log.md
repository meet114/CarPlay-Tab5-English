# Execution Log — 2026-11-05

## Completed

- Read the input plan; checked M5Stack official pages, Espressif docs, Apple MFi notes, and Carplay-S3 source.
- User confirmed ST7121; verified the 2026-04-28 panel change on the official site and actual BSP 1.3.2 FW=1 detection plus reset logic. The first page summary was incomplete; corrected from downloaded component source and the full official version history.
- Downloaded the upstream reference checkout and pinned the commit (see sources.lock.json). Did not download Release firmware that embeds authentication identity, and did not import private keys.
- Built a standalone ESP-IDF app: landscape color blocks, touch press/release logging, optional ES8388 short tone and ES7210 peak diagnostics, USB descriptor probe, plug/unplug counters, and heap monitoring.
- Pinned BSP/LVGL/adapter versions and generated dependencies.lock. esp_lvgl_port 2.9.0 DSI callbacks are incompatible with IDF v5.5.3; pinning 2.6.2 made the build pass without editing managed_components source.
- Host tests: valid USB configs, all truncated lengths, zero/overlong/undersized endpoints, total-length errors, illegal input not partially dispatched; 100000 deterministic random inputs. Clang AddressSanitizer + UndefinedBehaviorSanitizer passed.
- P4 v3.1–3.99 default diagnostic cross-build passed; P4 v1.0–1.99 audio diagnostic cross-build passed. Final v1.x default build also passed (app 721824 bytes); default delivery config is v1.x, 360 MHz, audio diagnostics off.

## Build environment and evidence

- ESP-IDF: v5.5.3, commit `2c211b236707889e8400c4dc5644dd5c4ee071e0`, SDK workspace unmodified.
- GCC: riscv32-esp-elf 14.2.0 (esp-14.2.0_20251107).
- `artifacts/rev3/`: v3.1 bin/elf, sdkconfig, build log.
- `artifacts/rev1-audio/`: v1.x with audio self-test and logs.
- `artifacts/rev1/`: final v1.x default build and logs.
- `artifacts/SHA256SUMS`: artifact digests.
- Upstream adapter emits one deprecated touch API warning; not a build failure. Usable at the locked versions; later upgrades must revalidate IDF/LVGL/BSP together.

## Incomplete / external conditions

- Not yet connected to a recognizable Tab5 USB serial port. Therefore no flash, display/touch/audio device measurements, iPhone enumeration, 20 plug cycles, thermal/power, or soak results.
- Actual chip revision unknown; v1.x and v3.x artifacts must not be mixed. BSP “board version 3” for the panel does not mean chip revision 3.
- No CarPlay authentication material usable for this project was provided. USBMUX, Lockdown, iAP2, NCM, AirPlay, video decode, and media playback are **not yet integrated into this app**.
- This round completed technical design and M0/M1 software implementation/build — not a full CarPlay delivery, and not M0/M1 device acceptance.
- Next: connect Tab5, read chip/Flash/existing partitions and prepare a backup, then run device acceptance; after the auth path is clear, advance M2 per the technical plan.


## Device connection update

- User cancelled the “face-video download test”; removed from the task.
- Identified `/dev/cu.usbmodem2101`: ESP32-P4 revision v1.3, 16 MB Flash, USB Serial/JTAG.
- Read existing partitions: nvs 0x9000/24 KiB; phy_init 0xf000/4 KiB; factory 0x10000/8 MiB; apps 0x810000/4 MiB; storage 0xc10000/4032 KiB.
- v1.x diagnostic app is 721824 bytes, image check passed, chip range 1.0–1.99. Can write only the factory app; no need to change bootloader, partition table, or data partitions.
- User said backup is not needed; a read that had already started finished by the time the message arrived. No further backups; local copy is at artifacts/device-2026-11-05/original-flash.bin.
- Write of `0x10000 build/carplay_tab5.bin` was rejected by auto-approval: user had not clearly authorized replacing existing firmware. **No Flash write was performed.** Waiting for explicit user authorization for that flash; do not bypass the rejection.
- Display, touch, and USB iPhone enumeration still await the diagnostic firmware running.


## Authorized flash and boot verification

- After the user explicitly replied “authorized”, only the existing factory partition `0x10000` was written; 721824 bytes, esptool reported Hash of data verified. Bootloader, partition table, and data regions were not written.
- Device now runs the default diagnostic firmware (audio self-test off); Flash SHA-256 is in this directory’s device run record.
- On-device boot confirmed P4 v1.3, CPU 360 MHz, 32 MB PSRAM/200 MHz, SPI SRAM memory test OK.
- BSP log identified ST7121 / FW 1; DSI native 720×1280, app rotates to landscape; touch created successfully, backlight set to 40%. This is driver-init evidence; visual screen quality still needs user confirmation.
- USB Host install completed, client Probe ready. No evidence yet of successful iPhone enumeration.
- Sampled free internal heap 261203 bytes, free PSRAM 29862848 bytes; short-term readings stable; does not prove soak pass.
- Logs: artifacts/device-2026-11-05/flash.log and runtime.log. Some early-boot character duplication; app-stage logs are normal; originals retained.
- Audio and CarPlay auth/session not yet verified.

- User confirmed on-screen content matches red/green/blue blocks and diagnostic text: basic display test passed. Four-corner touch accuracy and iPhone insertion not yet confirmed separately.


## Touch device verification

- While live serial was monitored, user tapped center and four corners; captured 5 complete press/release pairs.
- Press coordinates: (583,382), (6,67), (74,676), (1234,678), (1212,61); release: (580,373), (17,67), (65,666), (1228,676), (1212,61).
- Coordinates fall within landscape 1280×720 and cover center and corners; single-touch press/release basic test passed. No pixel-target calibration, multi-touch, long-press/drag, or CarPlay HID testing.
- Raw record: artifacts/device-2026-11-05/touch-live.log. First offline touch capture missed events; that alone must not be treated as touch failure.


## 2026-11-06 iPhone USB enumeration

User confirmed iPhone was connected to Tab5 USB-A; no reflash. Restarted via USB Serial/JTAG to capture a full log.

- Device: VID 05ac, PID 12a8, USB address 1.
- speed_enum=2: High-Speed per locked USB component enum (480 Mbit/s nominal link rate, not measured throughput).
- Active configuration=1; interface 0 / alt 0 / class 06 / subclass 01 / protocol 01 (Still Image/PTP interface traits).
- EP 0x02 Bulk OUT / 512 bytes, EP 0x81 Bulk IN / 512 bytes, EP 0x83 Interrupt IN / 64 bytes, bInterval=10.
- Driver notes the device has multiple configurations; current probe only inspects the active one, has not selected USBMUX/NCM config, and has not done pairing or CarPlay session.
- At boot 1567 ms one `HUB: Root port reset failed`; enumeration succeeded at 3268 ms. Do not ignore the earlier error because of later success; plug/unplug regression needed to separate transient recovery from stability issues.
- After enumeration free internal heap 259999 bytes, free PSRAM 29862412 bytes; four subsequent 10 s samples matched. Long-term stability not proven.
- Raw evidence: artifacts/device-2026-11-06/iphone-enumeration.log.


### First plug/unplug record and interference factors

- Captured disconnect count=1, then addr=2 successful High-Speed enumeration; after disconnect count=2 saw Root port reset failed and no recovery before the record ended.
- After both disconnects memory returned to the prior baseline (internal=261203, PSRAM=29862848); no heap evidence yet of handle leaks this round.
- Host serial later reported Device not configured; port changed from usbmodem2101 to usbmodem101. User confirmed moving/unplugging the debug link or the asked connection change during the test. Host serial interruption must not be attributed directly to a device crash; this round is insufficient to claim 3 plug cycles passed.
- Raw record: artifacts/device-2026-11-06/iphone-hotplug.log; continue control runs on the new port with USB-C left undisturbed.


### Three independent plug/unplug control runs passed

User completed three iPhone-side-only plug cycles; log iphone-relisten.log captured initial connect count=1, then three DISCONNECTED count=1/2/3 and CONNECTED count=2/3/4. All four enumerations were 05ac:12a8, High-Speed, configuration=1, same interfaces and endpoints.

- No Root port reset failed, panic, or serial drop in this window.
- Stable free heap after each connect was the same: internal=259999, PSRAM=29862412; after the first two disconnects periodic samples recovered to internal=261203, PSRAM=29862848; after the third disconnect immediate total free heap recovered to 30092020. Immediate heap on the second disconnect was still mid async free; later samples recovered — do not judge leaks from the immediate value.
- Conclusion: these 3 plug/unplug recoveries passed; no sustained memory decline observed. Does not mean the earlier port-reset failure root cause is fixed, and does not meet the plan’s 20-cycle / 30-minute acceptance.
- iPhone remains enumerated; still PTP config only; USBMUX/Lockdown/CarPlay not yet integrated.


## M2 prep: multi-configuration scan implementation

- Added main/usb_config_scan.c: for each Apple device, read 9-byte config header and bounded 4096-byte full descriptor per configuration; distinguish config index vs bConfigurationValue; record USBMUX/NCM interfaces. Supports P4 High-Speed 512-byte Bulk endpoints; did not keep upstream S3 full-speed-only limits.
- Does not send vendor requests, switch configuration, or start pairing. On scan timeout, retain in-flight DMA buffers until callbacks return, then free; on unplug, drain scan before closing the device.
- Added periodic STATUS logs so offline periods after the first enumeration still show current connection.
- Host ASan/UBSan tests cover config index/value, disconnect drain, late timeout callbacks, short packets, overlong claims, submit failures; original 100000 input tests passed. Cross-build passed.
- Within authorized scope, updated only factory app at 0x10000, 725440 bytes, write verified. Device boot OK, ST7121 init success. Current boot window active=0; asked user to reconnect the phone; on-device config-scan results pending.
- Evidence dir artifacts/m2-config-scan/. USBMUX transport and Lockdown/auth not yet implemented.


### M2 config scan passed on device

Kept iPhone connected and restarted diagnostic firmware; all 4 configurations read successfully (full-scan.log) without switching the active config:

| Index | Config value | Length | Related interfaces |
|---|---|---|---|
| 0 | 1 | 39 | PTP 06/01/01 |
| 1 | 2 | 149 | Audio 01/01/00, 01/02/00; HID 03/00/00 |
| 2 | 3 | 62 | PTP + USBMUX ff/fe/02 |
| 3 | 4 | 117 | PTP + USBMUX + vendor ff/fd/01 |

USBMUX on configs 3/4 is interface=1, alt=0, Bulk OUT=0x04, IN=0x85, MPS=512. No standard 02/0d NCM control interface observed; do not treat ff/fd/01 as NCM. Next: prefer observed config 3, wire USBMUX version handshake, then Lockdown QueryType.

Still saw Root port reset failed at boot 1567 ms, recovered enumeration at 3269 ms, scan done at 3276 ms; reset fault remains known. Active config stayed 1; no USBMUX transport, pairing, or auth started.


## M2 USBMUX / Lockdown QueryType passed on device

- Ported usbmux.cpp/h and lockdown_reply.h from the pinned upstream commit, kept GPL-3.0-or-later marks and license, added THIRD_PARTY_NOTICES.md. Removed multi-TCP and pairing process; kept only USBMUX v2, TCP 62078, and QueryType; adapted for 512-byte High-Speed endpoints.
- App owns Host init and enumeration filter; BSP only supplies USB power; register client before opening root port. Only for measured 05ac:12a8 / 4-configuration devices select configuration value 3; others keep default. Interfaces/endpoints still parsed from the active descriptor.
- Cross-build and three ASan/UBSan suites passed; added Lockdown reassembly at all split points, Error, overlong/zero-length/trailing-byte tests. Full USBMUX state-machine host fault-injection tests not yet done.
- Per authorization, updated only factory app, 738304 bytes, flash Hash verified, other partitions preserved.
- On device: speed=HIGH, config=3; claim IF=1 IN=85 OUT=04 OK; VERSION 2 accepted; TCP SYN/SYN-ACK done; QueryType returned 297-byte plist verified as com.apple.mobile.lockdown; recorded LOCKDOWN_QUERY_TYPE_OK. Later periodic status QUERY_TYPE_OK.
- This round still had Root port reset failed at 1568 ms, auto-recovered at 3269 ms, QueryType OK at 3276 ms; known fault not claimed fixed.
- Device evidence artifacts/m2-usbmux/runtime.log; flash/build records in same dir. Current free internal heap 194955, PSRAM 29792380; do not compare directly with older firmware for leaks (new transport/queue footprint).
- Plug drain/reconnect for this build not verified; 20-cycle regression not done. Next: Lockdown trust pairing and session/TLS; iAP2 auth material remains an external precondition — not entering CarPlay media yet.


## M2 trust pairing and TLS: implemented; device ops awaiting authorization

- Ported multiplexed USBMUX byte streams, Pair, StartSession, and TLS; truncated upstream CarKit/iAP2 follow-on. After TLS, run QueryType again to verify the service.
- Pairing records planned for existing NVS new namespace tab5_pair; no whole-partition erase code; stop on init or save failure. Private keys never logged; partition table unchanged.
- Build passed; artifact artifacts/m2-pair/carplay_tab5.bin; four ASan/UBSan host suites (descriptor, config-scan lifecycle, Lockdown fragments, XML response parse) passed. Full pairing/TLS and reconnect still lack device evidence.
- Auto-approval rejected flashing and running pairing: prior authorization did not clearly cover phone trust pairing and NVS credential persistence. Rejected commands were not run; device remains the verified USBMUX/read-only QueryType build.
- Awaiting explicit user authorization to: update factory app, have the user confirm Trust on iPhone, and save the locally generated pairing private key and record to tab5_pair.


## 2026-11-06 pairing, TLS, and reboot reuse passed on device

After explicit authorization to update factory, start iPhone trust pairing, and save private key/record to tab5_pair, wrote 983744-byte app, Hash verified. User confirmed Trust on the phone and entered the passcode.

- First run: 30406 ms `PAIR_OK identity saved for this phone`; 31386 ms `TLS_READY`, TLSv1.2 / TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384; 31487 ms `LOCKDOWN_TLS_QUERY_OK`, encrypted QueryType service-type check succeeded.
- Reboot verify: 3722 ms `PAIR_RECORD_REUSED phone key matched; trusted=yes`; 4637 ms TLS_READY; 4737 ms LOCKDOWN_TLS_QUERY_OK. Reused saved identity; no new identity or second Pair.
- Worker closes the TLS stream and exits after the diagnostic query; periodic USBMUX QUERY_TYPE_OK means basic query passed, not that a long-lived TLS connection is held.
- Wrote only existing app region and NVS tab5_pair; no partition-table change or NVS erase. No private key/pairing contents logged.
- Evidence: artifacts/m2-pair/flash.log, runtime.log, reuse.log, SHA256SUMS.
- Known boot Root port reset failed still appeared and recovered; full plug/unplug, lock-screen reconnect, and soak not done.
- Next: CarKit service bring-up, iAP2 link, and a lawful auth provider; no auth material yet — not entering CarPlay/NCM/media.


## CarKit/iAP2 implementation and phone-switch prep

- Wired StartService com.apple.carkit.service, optional service TLS, iAP2 marker/SYN/control diagnostic messages. On auth request, emit AUTH_MATERIAL_REQUIRED clearly and exit; do not supply auth identity; do not reuse unverified upstream NCM interface/media claims.
- iAP2 core and upstream regression tests keep GPL/copyright attribution; host ASan/UBSan regression passed; cross-build passed. First flashed build had active=0; no CarKit/iAP2 on-device success evidence yet.
- User switched to iPhone 12 Pro Max / iOS 16.1.1. To protect the first phone’s record, added per-phone public-key digest keyed storage: old record keys not deleted; new key is p+ first 7 digest bytes hex; load/overwrite checks full 32-byte digest; on conflict/corruption/space shortage stop — no erase or eviction of old records. Multi-phone storage still needs a second-phone device test.
- Two ESP serial ports appeared; before write, confirmed usbmodem101 chip is still the original Tab5 P4 v1.3 / MAC <board MAC>. Did not operate the other device.
- Phone-switch build updated only factory 0x10000, 992736 bytes, Hash verified; ST7121 init OK; waiting for second phone. Evidence artifacts/m2-multiphone/.

## 2026-11-06 offline development in progress

User asked to complete four items: session management, NCM/RTSP transport, video display and touch, audio interface and app UI. Keep device work paused; this round had no serial listen, pairing, or flash.

Added `main/core/session.c/h`: single task owns the state machine; connection generation shields stale completions; invalidate then cleanup; retry after cleanup (1/2/4 s, max three); missing/rejected auth material does not auto-retry. Added virtual-time tests.
Added NTB16 codec (pinned upstream with provenance) and RTSP incremental receiver supporting consecutive requests and binary bodies; must reset after parse errors so recovery does not resume from a bad body. Added touch letterbox mapping, drag clamp, cancel release.

`./scripts/test_host.sh` all passed (ASan/UBSan); ESP-IDF v5.5.3 build passed, log `/tmp/carplay-offline-build.log`. New core is in component sources but not yet wired into the app call path; this build does not prove new protocol features run. State-machine adaptation, NCM USB/IPv6 driver, RTSP service, OpenH264 video path, PCM playback, and UI still to implement — do not mark the four items done.

### A/V offline module progress

- `pcm_buffer`: 44.1/48 kHz, 16-bit mono/stereo; RTP timestamp positioning, in-window reordering, duplicate suppression, endian conversion, 60 ms prebuffer, underrun silence, session clear. Input assumed already authenticated/integrity-checked upstream; this module does not provide crypto verification.
- `video_queue`: up to 3 complete AUs / 512 KiB; on overflow drop broken reference chains, wait for IDR, and offer recovery request; isolated by connection generation. Dequeue copies into independent decode input so queue reset cannot invalidate decode pointers.
- Pinned Cisco OpenH264 2.6.0; 112 files verified; introduced portable decoder component and I420/RGB565 output wrappers. Decoder and host tests share long-term reference and flush fixes.
- `scripts/test_decoder.sh` actually decoded a High Profile synthetic first frame and 8 continuous rounds totaling 32 LTR frames; per-frame CRC all matched; RGB565 black/white and scale tests passed. Requires ESP-IDF configure/build generated fix files first.
- `scripts/test_host.sh` including new A/V buffer tests all passed ASan/UBSan. Standalone OpenH264 host tests themselves do not enable sanitizers.
- Decoder module P4 build passed (`/tmp/carplay-decoder-build.log`). Module not yet referenced from the app path; linker may GC unused code, so firmware size then did not represent full media Flash budget.
- Added `audio_output.c/h` independent playback task; USB/network submit uses zero-wait lock; stop and codec close are command-confirmed and serialized; not yet device-verified.

Still needed: media worker/display integration, unify state machine with existing USB/Lockdown connection, NCM/RTSP network service and integrity checks, UI, end-to-end simulation and final build. Four goals continue; not marked complete.

### App display and media task integration

Added `app_ui.c/h`, `video_output.cpp/h`, `offline_demo.c/h`; entry initializes real decode and audio workers; video worker consumes complete AUs and displays via OpenH264/I420/RGB565 to a persistent LVGL canvas. Update pixels while holding LVGL lock; source buffers not freed during async refresh; connection generation and decode revision shield stale stream results; errors/queue loss request IDR. Still needs upper transport to deliver media and consume IDR requests.

`TAB5_OFFLINE_DEMO` defaults off; when on, skips USB init and only loops synthetic High Profile bitstream plus local touch; clearly marked Simulation; does not affect real auth state. Default device mode still needs state-machine adaptation to connection events.

`tests/ui` builds host rendering with real LVGL 9.3 and production app_ui.c, outputs artifacts/ui-{disconnected,auth-missing,simulation,cleared}.ppm; reviewed auth-missing/demo screenshots and fixed small fonts and demo-label background contrast. BSP replaced by host framebuffer stub — does not prove DSI timing or on-board response. Real UI is English; no Chinese font introduced. Host render ≠ full video worker path already running.

After first real link of video and UI, firmware size 0x1405e0; font config change was rebuilding. Not flashed; NCM/RTSP service, session adaptation, HID delivery, full integration tests remain.

### NCM driver and AirPlay protocol port

- Firmware after the prior font-size change built successfully.
- Added `cp_ncm_discover`: uses CDC Union association (supports non-contiguous interface numbers), filters Ethernet MAC string, supports 64/512-byte bulk endpoints, rejects missing/duplicate/unsupported endpoints; truncation/association/endpoint tests added to full ASan/UBSan suite and passed.
- `ncm_host` ported from pinned upstream, switched to the new discoverer, adapted notification endpoint packet length, frees allocations on init failure; on disconnect stop network and halt/flush this module’s data/status endpoints, still wait for outstanding callbacks to return DMA. Shared control endpoint not force-flushed. USB probe wired for NCM start/poll/disconnect; app initializes esp-netif; no unknown mode-switch commands sent to the phone.
- NCM-integrated firmware build passed, size 0x17bbd0; real NCM enumeration, throughput, and exit races still need focused device verification.
- Introduced pinned upstream AirPlay RTSP/session, pairing, control, screen, timing, media_bridge, and net/time/auth adapter components; licenses recorded. Auth provider returns unsupported when material is absent.
- First AirPlay build reported ChaChaPoly/HKDF not enabled; SDK config filled in and rebuild in progress. AirPlay service not yet started from the app; upstream media_bridge still references video_live_* APIs to replace; audio still needs local playback wiring — must not treat media integration as complete.

### Video, touch, and audio delivery interfaces

- AirPlay component built after enabling ChaChaPoly/HKDF.
- `media_routes.cpp` adapts upstream media_bridge video_live_*: caches bounded Annex-B config, submits complete AUs, keeps reset generation, requests IDR via event channel; actual RTSP start still awaits connection.
- `hid_output.cpp` posts LVGL callbacks to an independent queue, builds single-finger 12-byte reports per upstream two-touch descriptor; on send failure/queue overflow prefer injecting a release. Added report endian and release tests; full host suite passed. Still needs session-generation/reconnect integration regression.
- First video/HID adapter build found missing C linkage for C APIs included from C++; fixed with extern C; size 0x17e100.
- AirPlay PCM gained a sink registration API; only data that authenticated/decrypted successfully submits timestamp and network-order PCM into the local jitter buffer; supports 44.1/48 kHz; other formats rejected explicitly. Local output supports one PCM path; a simultaneous second SETUP fails explicitly — do not claim playback success. Multi-stream mix / navigation concurrency not in completed evidence.
- Audio-delivery build compiling. Protocol service start, identity persistence adaptation, session management wired to real events, and integrated simulation tests still pending; no flash or listen.

### AirPlay identity and service lifecycle

Audio-delivery build passed. Added local AirPlay Ed25519 identity provider and controller store from pinned commit with license retained; switched from upstream pairing partition to default NVS namespaces tab5_ap_id/tab5_ap_pair; does not modify Lockdown records or erase on failure. P4 has no BT MAC; derive local management metadata from chip default MAC — does not claim a Bluetooth link. Module built; identity not yet generated/written on device.

Added airplay_service start/stop API: check connection generation/NCM IPv6, init NVS and identity, start RTSP; stop waits for service exit then closes media; API awaits session-manager call — standalone compile ≠ service running. Added refcounted entropy-source management; Lockdown and AirPlay holders acquire/release separately so one ending does not close entropy the other still uses. Build log /tmp/carplay-service-build.log.

Still critical: wire app session manager to real USB/Lockdown/iAP2 events; drive protocol success path with a mock provider; align /info with actually supported formats; end-to-end/error/concurrency tests. Not a complete four-item delivery.

### Capability declaration review and full link

Found upstream /info declaring two screens, voice/low-rate outputs, and OEM menu — mismatches current implementation. Added scripts/configure_tab5_info.py to normalize all templates: single screen and 44.1/48 kHz 16-bit PCM only; disable nonexistent OEM menu; keep template format and identity placeholders; audio SETUP accepts only type 100. Script --check added to host tests. Concurrent multi-stream still explicitly rejected by output layer; mixing not implemented.

Forced retention of tab5_airplay_start/stop in the link so unused entry points are not GC’d hiding dependency errors. Full link passed; firmware 0x1a6580 (1,729,920 bytes). Added binary plist and screen codec upstream boundary regression tests; ASan/UBSan on local protocol sources; full suite passed.

Video profile changed from upstream default 480x288 to plan-agreed 320x192; storage moved to independent tab5_video; does not read/rewrite upstream screen_ui; higher tiers kept as experimental. Awaiting next build to validate.

Service auto-start, iAP2 auth/identify state integrated with session manager, and integrated simulation success path still incomplete. Goals remain in progress; no device access.

### Session manager wired to real protocol events

Added app_session as single-task state owner; USB task runs attach/detach/poll; Lockdown/iAP2 worker only posts captured transport epochs to a bounded queue; expired events dropped. Successful TLS QueryType → Paired; CarKit → authenticating; missing material/reject → terminal error. Normal diagnostic-worker return also treated as communication end; later generic errors do not overwrite an earlier auth-missing arrival.

USB path STOP→halt/flush→wait DMA and Lockdown worker done→DRAINED, then core state machine 1/2/4 s bounded retries. Init order: UI/media/state management first, then USB. Fixed clearing the stopping flag when physical disconnect cleanup completes so a new device is not released by a stale cleanup flag. That last integration fix awaits a later build review.

Prior session-adaptation P4 build passed; tests/test_app_session.c uses real app_session.c + queue/virtual-clock stubs to verify real adaptation events, stale generation, no retry on missing material, max three timeout retries. Full ASan/UBSan passed. Real USB DMA/task concurrency not covered by that test.

Scope still insufficient to close goals: iAP2 remains diagnostic; no provider success path for auth/identify/StartSession; AirPlay service entry exists but is not started from auth-success; full success-path simulation and concurrency fault regression incomplete. No device access.

### iAP2 success-path wiring (offline verification in progress)

Migrated cert/challenge sign/signature ACK/phone accept, identification, and CarPlayAvailability→StartSession path from pinned upstream control client. Removed S3 identity and unimplemented send capabilities; use Tab5 model and chip serial. NCM control interface requires explicit CONFIG_TAB5_IAP2_NCM_INTERFACE; default -1 stops; do not treat upstream interface=3 as on-device fact. When auth_port has no real provider, keep showing missing material — do not forge accept.

After both auth and identification succeed, report CP_AUTH_OK; after NCM IPv6 ready, call tab5_airplay_start, report CP_NETWORK_OK, send actual service port and local identity in 0x4301. Stop media service when leaving the control loop. Report CP_MEDIA_OK only for frames successfully decoded and handed to UI — SETUP alone must not show Streaming.

Added AuthState core used by the client; tests cover success before ACK, early success, repeated cert steps, continue-after-reject; parameter parse tests added to suite. Full ASan/UBSan passed, but whole iAP2/RTSP success path not yet driven by a simulated phone. First build found new Kconfig not generated; reconfigure/build in progress; later capability-list shrink also needs build validation. Device work remains paused.

### iAP2 client simulation and media composition regression

After reconfigure, full iAP2 firmware build passed. Added tests/test_iap2_client.cpp compiling production main/iap2_client.cpp with mocked CSM/core callbacks, signing provider, NCM, and service lifecycle for 6 scenarios: success start/exit, missing provider, early accept, missing signature ACK, NCM timeout, service start failure. ASan/UBSan passed. No real wire core, real signing, or RTSP socket — not full phone end-to-end auth.

Extended test_decoder: real screen length-prefixed→Annex-B conversion→bounded AU queue→OpenH264→pixel CRC→touch map/HID report; High Profile and 32 LTR frames still pass. Path assumes input already authenticated; does not cover encrypted network or board display.

Fixed video_output lock-race flag: a failed submit from an old generation must not force reset of a new video stream; visible generation and drop flags synced under a short critical section. That firmware build passed. Also added HID event epoch: enqueue snapshot, check under control-channel lock on send, drop old queue on reconnect so stale taps do not enter a new session; that change was building.

Still pending full offline acceptance audit, native RTSP/encrypted media tests, and important concurrency path review. Four goals not marked complete; no device ops.


### Closing the first four offline items (2026-11-06)

Session management, NCM/RTSP, video/touch, and audio interface/app UI are wired in source and delivered under the “develop first, device adapt later” scope. Current status and limits are unified in [Four Deliverables](four-deliverables.md); earlier “still to develop” wording in this file is retained as history.

Final check fixed USB unplug order: wait for Lockdown/iAP2 to stop media before releasing NCM, and defer opening a new device while an old session is stopping. Default firmware full build passed (1,736,864 bytes). Independent TAB5_OFFLINE_DEMO=y build exposed media-adapter static-lib symbols not pulled in; fixed by adding link roots (1,434,480 bytes); default config then rebuilt successfully.

Final scripts/test_host.sh ASan/UBSan full suite passed; real OpenH264 High/LTR 32-frame per-frame CRC and Screen→queue→decoder→touch/HID composition tests passed. Added scripts/test_crypto.sh driving production cipher/screen with SDK-native Mbed TLS: all control-data fragments, decrypt-to-bytewise RTSP parse, poison-after-replay, screen HKDF, truncation, tag tamper and replay reject passed. Fixed a wrong empty-input state assumption in the test: empty input with no pending may legally return OK. This verification does not cover a real phone socket / full pair-verify exchange.

Production app_ui.c host-rendered four UI states with LVGL passed; auth-missing and simulation screens reviewed; simulation gray video comes from UI render tests — actual H.264 decode is proven separately by CRC tests. Firmware hashes, build, and host logs saved under artifacts/offline-delivery/. README, provenance notes, and delivery docs synced.

This round: no flash, serial listen, phone pairing, or on-device NVS writes. Auth provider, phone NCM config, and real media performance remain paused device adaptation; mixing beyond single PCM and mic uplink/AEC are outside these four deliverables. Completed offline development baseline — not on-device CarPlay acceptance.


### 2026-11-06 resume device bring-up: pairing storage insufficient

User confirmed iPhone connected and asked to continue. Confirmed usbmodem2101 is original board ESP32-P4 v1.3 / MAC <board MAC>; existing factory is 0x10000/8 MiB. Updated only factory app; did not write bootloader/partition table or erase NVS.

New display/touch drivers started; USB 05ac:12a8 config 3 enumeration, USBMUX v2, and Lockdown QueryType all succeeded. Current phone needed a new local pairing identity; save returned ESP_ERR_NVS_NOT_ENOUGH_SPACE. NVS stats used=444/free=312/total=756; tab5_pair has 2 blobs totaling 13536 bytes, 6768 bytes each. Free entries include NVS page management and copy-on-write constraints — not directly usable blob bytes. Identity contents not logged or exported.

Added CP_STORAGE_ERROR terminal state and UI hint; storage errors are no longer treated as communication faults that regenerate keys and retry. Host ASan/UBSan regression and firmware build passed; flashed to original board, Hash verified; on failure device entered Stopping→Failed error=5, USBMUX IDLE. Both existing records retained.

Firmware SHA-256: 02006b44aeb412c092c0945f8b1df9a6c7d7f567037a03855dc4d9d0a6245e79. Logs: artifacts/device-2026-11-06/storage-diagnostics.log, storage-fix-flash.log. Not yet into phone Trust / CarPlay auth / media. If clearing old pairing, scope must be tab5_pair only, with explicit user permission to discard old identities and re-trust; must not erase whole NVS.


### 2026-11-07 targeted clear and re-pair succeeded

User explicitly authorized clearing only tab5_pair old pairings. Re-checked original board MAC <board MAC>; temporary maintenance app used only the tab5_pair namespace handle for nvs_erase_all/commit, then verified the namespace empty and stopped. Log PAIR_RESET result=ESP_OK. Did not erase NVS partition or modify other namespaces. Then rebuilt with TAB5_PAIR_RESET_MAINTENANCE=OFF and restored normal factory app; both writes Hash verified. Maintenance mode does not start USB/pairing, avoiding generating a new record then deleting it on reboot.

After clear, NVS used=12/free=744, tab5_pair blobs=0. First user tap mistakenly chose Don’t Trust; captured UserDeniedPairing; user then confirmed Trust. Latest boot log confirmed tab5_pair blobs=1/payload=6768, NVS used=228/free=528; PAIR_RECORD_REUSED phone key matched; trusted=yes — trust record persisted and reused.

On-device pass: TLSv1.2, LOCKDOWN_TLS_QUERY_OK, CARKIT_SERVICE_READY encrypted=yes, IAP2_MARKER_OK, IAP2_LINK_READY. After phone CSM 0xaa00, stopped normally due to authentication_identity_missing; identified=0/authenticated=0, final Failed error=3. No real auth provider yet; not into NCM/RTSP/media; do not ask for repeated phone Trust. Observation-window free internal heap held at 121119 — not soak acceptance.

Logs: artifacts/device-2026-11-07/reset-result.log, normal-flash.log, re-pair.log, trust-confirmed.log. Current normal firmware SHA-256: 1b5fa5afa66a9f185ac9ba31de96d62349f5d4846a555f658acc488b2b9c1bfe.


### 2026-11-07 auth integration conditions confirmed

User explicitly confirmed no auth material is available now. Verified phone Trust and iAP2 link ≠ accessory authentication success; lacking inputs for a real CarPlay challenge-response. Keep successful on-device pairing and current firmware; do not re-clear; do not use a mock provider to impersonate a real identity.

Source fix: NCM interface not configured separated from CP_AUTH_MISSING into CP_PROFILE_MISSING/CP_ERR_PROFILE with a distinct UI network-config hint, so later config issues are not misread as missing certificates. Added production client regression for CONFIG_TAB5_IAP2_NCM_INTERFACE=-1; full host ASan/UBSan passed. This source fix not flashed. Restoring a real path needs materials or hardware for a usable auth provider; afterward still need NCM descriptor/config adaptation and real media acceptance.


### 2026-11-07 software auth passed on device

User asked to implement and continue along the Carplay-S3/DiPlay provenance path. Prior build was blocked by auto-approval quota; this round re-approved normally without bypassing review. Ported optional auth_software_provider, fixed IDF dependency pre-scan, adapted Mbed TLS field wrapping. Kept provenance licenses; private key and cert only under ignored secrets/mfi; contents not printed. Fixed APK hash and public/private key match verified; details in [software-authentication-setup.md](software-authentication-setup.md).

TAB5_SOFTWARE_AUTH=ON, TAB5_PAIR_RESET_MAINTENANCE=OFF build passed. Verified original board P4 v1.3/MAC <board MAC>; wrote only factory 0x10000, 1743296 bytes, Hash verified. Did not clear pairing/NVS.

Device logs in order: P-256 raw challenge signature self-test passed; PAIR_RECORD_REUSED trusted=yes; TLS_READY; CARKIT_SERVICE_READY encrypted=yes; IAP2_LINK_READY; MFI_CERTIFICATE_SENT bytes=607; MFI_SIGNATURE_SENT digest_bytes=32 signature_bytes=64; MFI_SIGNATURE_ACKED; CSM_RX id=aa05; IAP2_AUTHENTICATED phone accepted challenge response.

This is evidence the connected phone accepted the experimental identity — not simulated auth. Later when the phone requested 0x1d00 Identification, stopped normally because NCM interface defaults to -1: ncm_profile_not_configured, identified=0/authenticated=1. Full media session / picture / sound not started — cannot claim full CarPlay available.

Logs: artifacts/device-2026-11-07/software-auth-runtime.log; flash: software-auth-flash.log; firmware hash and switches: software-auth-build.json. Auth source build switch left ON; maintenance clear left OFF.


### 2026-11-07 CarKit second TLS: memory root cause confirmed and PSRAM allocation

Symptom: after NCM connected, second handshake failed with `esp-aes: Failed to allocate memory for end alignment buffer`, mbedTLS returned -0x1. Same diagnostic code, single-variable control: only mbedTLS allocation mode switched.

- INTERNAL (control, tls-internal-runtime.log): deterministic reproduce. `TLS_ALLOC_FAILED carkit count=1 last_size=128 caps=0x80c in=heap_caps_aligned_alloc`; at failure dma_free=1475, dma_largest=256, internal_free=16835, lp_free=14460, tcm_free=900, ~26 MB PSRAM free.
- EXTERNAL (tls-external-runtime.log, plus reset retests repeat1/2, 3 runs): both lockdown and carkit handshakes succeeded, alloc_fail=0, mbedtls_heap=psram, dma_min=38059, carkit total 672–721 ms.

Conclusion: root cause is internal DMA memory exhaustion. Under INTERNAL, mbedTLS depletes the internal DMA heap then falls into LP RAM/TCM; `heap_internal` still shows ~16 KB, but most of it is not DMA-capable, while AES-DMA alignment buffers need INTERNAL|DMA. Total internal heap numbers cannot diagnose this — use dma_free/dma_largest.

Ruled out: with PSRAM buffers, AES alignment buffers requesting DMA|SPIRAM do not fail; esp_mm/heap_align_hw.c strips DMA for SPIRAM requests; on-device alloc_fail=0 confirms.

Unconfirmed: prior firmware’s 7.5 s CPU0 watchdog did not reproduce. Then PC was in RSA blinding mbedtls_mpi_core_gcd_modinv_odd with operands in LP RAM. New INTERNAL control carkit CERTIFICATE_VERIFY was 914 ms (EXTERNAL ~500 ms); four runs had no watchdog — consistent with slowdown into LP RAM, but does not explain 7.5 s.

Tradeoff: Espressif docs recommend internal allocation for security. Under EXTERNAL, TLS session keys and pairing root private-key runtime copies live in unencrypted PSRAM; fine for experimental firmware — reassess for productization.

Code: lockdown_client.cpp added TLS_BEGIN/TLS_HEAP/TLS_STEP/TLS_ALLOC_FAILED; handshake made single-step with yields between steps. sdkconfig.defaults enables CONFIG_MBEDTLS_EXTERNAL_MEM_ALLOC. Wrote only factory 0x10000, Hash verified, MAC <board MAC>, NVS/pairing untouched. Board currently runs EXTERNAL firmware, SHA-256: 999d5c5e127210a81b07000b729431ebc24ada67410fd983cf331af03e45207e.

Next failure: after IAP2_AUTHENTICATED phone returns 0x1d03, IDENTIFICATION_REJECTED parameter=6,7 (send/receive message lists), identified=0, three times consistent. Also known: after failure, in-place retry sticks in USBMUX VERSION_PENDING, final Failed error=1. Neither handled; not into RTSP/media.


### 2026-11-07 Identification alignment: entered Streaming

Changed iap2_client.cpp: parameters 6/7 aligned to S3 (DiPlay wired control message lists — 14 send, 11 receive); added RouteGuidance component (parameter 30). New messages already have receive handlers; firmware does not actively send those Start requests. Ref: Carplay-S3 validation notes recorded the same rejection when the declaration was reduced. test_iap2_client added Identification content assertions that failed on old code first; after the change scripts/test_host.sh full suite passed.

On device (identification-aligned-runtime.log and -repeat-runtime.log, 2 runs, consistent): IDENTIFICATION_SENT bytes=267 usb_interface=3 → 0x1d02 IAP2_IDENTIFIED → IAP2_CONTROL_READY → 0x4300 CARPLAY_AVAILABILITY → CARPLAY_START_SESSION → RTSP pair-setup/pair-verify, rtsp_cipher=enabled, SETUP/RECORD → screen_peer=connected → SESSION stage=Streaming. Streaming is triggered by CP_MEDIA_OK — at least one frame decoded and successfully submitted to tab5_ui_frame. Screen not visually inspected; cannot claim correct/continuous picture or working audio.

Next failure: both runs at 65.6 s `USBMUX: TCP_RX flags=04 seq=302 ack=276` → `TCP RESET`, CarPlay session torn down. seq/ack identical both times; that usbmux.cpp branch only handles local port 1 — the boot-time QueryType probe: response 301 bytes+SYN=302, request 275 bytes+SYN=276. Connection was never closed after use; phone reset ~60.0 s after establish (idle timeout inferred; interval measured); usbmux.cpp:90 sets failed=true on any RST and tears down all healthy streams via mux_channels_transport(false). Unchanged.

Also: screen_message=discarded error=ESP_ERR_NO_MEM (14× / 4×) is not OOM — media_routes.cpp:71 maps tab5_video_submit false to NO_MEM; actually video queue reject; ~26 MB PSRAM free. First run at 45 s phone resent 0x4300 and firmware sent another 0x4301; retest did not. After teardown in-place retry still fails: `Unexpected version reply`, VERSION_PENDING, final Failed error=1.

Board firmware SHA-256: 11bacc194cf41dc06bab6f34b194898e471e99bdf917884afb0d7cf2440a97ae. Wrote only factory 0x10000, Hash verified, MAC checked, NVS untouched.


### 2026-11-07 QueryType probe connection close: session past 60 s

Changed usbmux.cpp: after QueryType, send FIN|ACK on port 1 (PROBE_CLOSED), same as normal stream close; RST on that connection after probe completion logged as PROBE_RESET and ignored; RST before probe completion still fatal. Phone returns RST ~15 ms after FIN, so both changes are required; one PROBE_RESET per boot is normal. usbmux.cpp has no host-test coverage; regression relies on device; scripts/test_host.sh full suite passed.

On device (probe-closed-runtime.log 150 s, probe-closed-repeat-runtime.log 130 s): both passed the old 65.6 s failure point; at capture end (~146 s / 126 s) still Streaming, no Stopping, no fatal TCP RESET path, no watchdog; NCM RX/TX kept growing ~rx_usb +220 / tx_frames +300 per 10 s; first run still had screen data at 141 s.

New observation: 67.1 s `Peer reset active connection local=3` (both runs); local=3 is the idle Lockdown TLS connection, ~60 s since last traffic; session unaffected. Error log currently harmless; confirms phone resets idle Lockdown ~60 s, and that the iAP2 control loop does not depend on that connection staying up.

Unverified: physical screen, audio, touch, long soak, plug recovery. screen_message=discarded still present (5× / 4×, video queue reject). In-place retry-after-session-break still unhandled.

Board firmware SHA-256: 914f3f069e54be9198a18a873ef5b2e2ac60548138be22a3da6a650f00180fd7. Wrote only factory 0x10000, Hash verified, MAC checked, NVS untouched.


### 2026-11-07 screen flash then blank: config restart loop and present latency

Symptom: user reported picture flashed briefly then blank. Streaming means first frame presented, not continuous display.

Added VIDEO STATS/START/CONFIG/IDR_REQUEST frame-level diagnostics (video-diag-runtime.log, ~58 s): sub=129 idr=117 dec=90 err=0 ui_fail=0, but starts=340 decoders=90. Root cause: every video_live_config unconditionally restarted video and tab5_ui_clear hid the canvas; queue reset triggered IDR; phone responded by resending the same config — self-sustaining ~6 restarts/s.

Fix 1: cp_video_config_changed — identical config bytes do not restart (video-config-fix-runtime.log). starts=1, but tab5_ui_frame fixed ~505 ms (28775 ms/57 frames) holding mutex the whole time, busy=224/284, ~1 fps.

Fix 2: pixel_convert.c precomputes column maps and reuses repeated source pixels/rows; removed per-pixel 64-bit divides; built -O2 alone; video_output.cpp split draw lock so present no longer holds queue mutex; start takes draw first so finished streams are not drawn; submit lock timeout 5 ms. tests/test_pixel_convert.c uses per-pixel algorithm as reference and passed before/after, covering Tab5 upscale, 1:1, downscale, and non-integer ratios.

On device (video-fast-runtime.log, ~56 s): sub=337 dec=241 idr=22 busy=0 rej=31 err=0 starts=1, ~4.3 fps, ui avg ~149 ms / max 169 ms, decode max 101 ms, no Stopping, no watchdog.

Leftover: input ~6 fps exceeds processing; when queue fills it clears and waits IDR — rej and ~2.7 s IDR recovery, decoder rebuilds (decoders=22). ui still ~149 ms, composition not split. Resolution still 320x192 scaled to 1200x720; codebase config max 640x384. Screen not visually inspected; single run; audio, touch, soak, plug recovery unverified.

Board firmware SHA-256: 2b197acb8a9fdc2310c48ab78aa4b606724bdafe39a19c8d2b3e7143adedeaa2. Wrote only factory 0x10000, Hash verified, MAC checked, NVS untouched.

## 2026-11-07 physical picture confirmed and geometry analysis

User photo confirmed Tab5 shows CarPlay. Icons look large; only a few fit.

- Scale: no crop in code; `cp_touch_configure` letterboxes the full frame into 1200x720. Icons are large because `info_192.plist` declares 320x192 to the phone (physical 200x120 mm); the phone lays out for that tiny display, so the UI itself only fits that much.
- Orientation: in the photo "Device mode" and video are both upside-down the same way. That label sits at the bottom in LVGL, so video and LVGL orientation match; upside-down came from how the board was held when photographed. Firmware fixed at `LV_DISPLAY_ROTATION_90`; whether install orientation is correct awaits user confirmation.
- Occlusion: `tab5_ui_frame` only hides title and detail; "Device mode" still overlays during casting.
- Resolution switch: current tier stored in NVS `tab5_video/profile_h`; no call site for `carplay_video_profile_select`, so no runtime switch; all four tiers declare physical 200x120 mm, `experimental` all false — raising tier has no automatic fallback protection.
- Constraint: at 320x192 processing already lags input (~4.3 fps vs ~6 fps); raising resolution directly would slow decode and present further — throughput must be fixed first.

Analysis only this section; no code change, no flash.

## 2026-11-07 PPA present and 480×288

Baseline firmware breakdown (`ppa-baseline-phone`, 320×192, 260 frames): color convert + scale ~43 ms, memmove and letterbox black ~60 ms, LVGL refresh ~86 ms (software 90° rotate; original `ui` stats did not cover it), decode ~50 ms.

Changes:
- `main/app_ui.c`: removed LVGL canvas. Frame converts to RGB565 at source size; PPA scales and rotates into an undisplayed DPI framebuffer, then `esp_lcd_panel_draw_bitmap` flips. During video LVGL invalidation is off; touch input unaffected; `leave_video` restores UI. Simulation mode keeps Simulation label.
- BSP does not expose panel handle; obtained via `--wrap=lvgl_port_add_disp_dsi` (`main/CMakeLists.txt`).
- "Device mode" label no longer overlays during casting.
- Profiles: added `CONFIG_TAB5_VIDEO_PROFILE_HEIGHT`, default 288. NVS `tab5_video` gained `wanted` key; tiers above 192 marked experimental. New request triggers one trial starting at first presented frame; after 100 frames decide; reject rate over one-fifth falls back. Crash or unfinished trial → next boot returns to 320×192 and that request is not retried. Boots without phone do not count as trial. `tests/test_video_profile.c` covers these rules.

Offline self-check (temporary build, reverted): four-corner color blocks read back match LVGL mapping, endian correct, black borders 0.

On device (phone home screen, picture mostly static):

| Profile | Run | rej | idr | Decoders | Decode mean/peak | Present mean | Input |
|---|---|---|---|---|---|---|---|
| 320×192 old path | ~60 s | 36/370 | 26 | 26 | 50/108 ms | 140 ms + LVGL 86 ms | ~6 fps |
| 320×192 PPA | ~62 s | 0/416 | 1 | 1 | 18/77 ms | 24 ms | ~6.7 fps |
| 480×288 | ~77 s | 0/344 | 2 | 1 | 32/148 ms | 43 ms | ~4.4 fps |
| 640×384 | ~74 s | 1/145 | 2 | 2 | 92/300 ms | 76 ms | ~1.9 fps |

640×384 peak ~440 ms/frame — no headroom in motion; default chose 480×288. At 480×288 PPA ~11 ms, color convert ~31 ms — next optimization target. Phone sends fewer frames at higher tiers; reason not investigated.

Final firmware SHA-256 `0e0c04aa342cf2e4e2d33665d7ff63aa56a6e08ef73f40ca91c05a20dea1f2b4`, 480×288, trial confirmed: after reboot `probe=0`, auto-reconnect and video restored, no Stopping, no watchdog.

Unverified: physical sharpness, motion fps, touch coordinates, plug return to LVGL (`leave_video`), simulation label on device, audio, long run.

## 2026-11-07 plug recovery, UI scale, and decode pipeline

User device test: touch accurate; unplug returns to Disconnected UI; re-insert cannot enter CarPlay; icons too large; map drag stutters.

Plug recovery:
- Root cause: `usb_probe.c` `mode_requests` budget of 2 never cleared; already spent at boot. After re-insert phone returns to default mode and needs another CarPlay mode request, which is refused (`No NCM configuration after bounded mode requests`), stuck at `USBMUX: STATUS IDLE`. Reproduced by powering down root port (`replug-before-runtime.log`).
- Fix: if more than 5 s since last enumeration, restore `mode_requests` and `config_restarts` budgets; still bounds re-enumeration loops.
- Verify: temporary hook powered down root port — equivalent disconnect, not physical unplug. Old path `replug-after` two rounds ~8.4 s recovery; final pipeline `replug-576` two rounds ~9 s.

UI scale:
- Added temporary serial frame-export tool to inspect decoded frames (tool not checked in).
- `widthPhysical/heightPhysical` do not change scale: 55×33 and 200×120 layouts match; 110×62 difference comes from aspect compensation. Phone lays out at 1×; icons ~60 px; ratio depends only on pixel count.
- Reference icons ~1/10 of height → 1024×576. Added 852×480 and 1024×576 tiers (16:9, no letterbox, PPA scale 1.5 / 1.25); `info_480/576.plist` HID descriptors synced; physical sizes by pixel ratio 142×80, 160×90.

Stutter:
- At 480×288 capped at 6 fps, `rej=0` — phone exactly fills 6 fps; the cap is the main stutter cause. Raising to 12 fps exceeded capacity → IDR storms.
- Fallback decision previously applied to already-confirmed tiers too, permanently dropping to 320×192 after one overload. Changed to trial-only; `carplay_video_profile_begin` returns whether in trial; tests updated.
- Decode and display split into two tasks with three rotating frame slots; when display lags only pending frames are skipped — decode reference chain intact.
- Color convert moved to PPA: `cp_i420_pack` packs `U Y Y / V Y Y`; display task only drives PPA. Offline self-check four corners exact f800/07e0/001f/ffff. `cp_i420_rgb565` no longer used in firmware; decoder tests only.
- L2 cache line → 128 B: 1024×576 decode mean 325→278 ms, peak 600→453 ms.

Simulated drag (36 s window, firmware-injected touch, not human finger; at 1024×576 drag region covers dashboard including page flips):

| Profile/cap | Input | Display | rej | IDR | Decode mean/peak | Present |
|---|---|---|---|---|---|---|
| 480×288@6 (pre-change) | 6.0 | 6.0 | 0/216 | 0 | 70/174 ms | 44 ms |
| 480×288@12 (parallel, mid-change) | 9.7 | 5.1 | 86/353 | 27 | 136/974 ms | 78 ms |
| 852×480@4 (final) | 3.8 | 3.6 | 2/133 | 3 | 246/2003 ms | 27 ms |
| 1024×576@3 (final) | 3.0 | 3.0 | 0/108 | 0 | 278/453 ms | 59 ms |

Final firmware SHA-256 `b159b4a6fc0e7af8cedb8c9e3d23aeda8da57e684edd22879b80501f041f37ba`, 1024×576, 3 fps, `probe=0`, held across reboot, auto-reconnect OK, host tests passed.

Risk (corrected next section): during streaming `heap_internal` ~40 KB. PPA and L2 128 B line not long-soak tested.

Unverified: real finger drag feel, touch coords at 1024×576, physical unplug, audio, long run.

## 2026-11-07 smoothness: align M5 config and CABAC optimization

User device test: physical plug recovery OK, touch accurate, map drag still stutters.

Research:
- M5Tab5-UserDemo uses XIP from PSRAM, L2 256 KB, line 128 B, QIO, global -O2; Espressif esp_lvgl_adapter docs give the same P4 config.
- Espressif esp_h264 soft decode on P4 is 25/31 fps at 640×480 (1/2 tasks) but only Constrained Baseline. Captured iPhone SPS/PPS: `profile_idc=100`, level 3.1, `entropy_coding_mode=1` — High + CABAC, unusable. No way found to make the phone send Baseline.
- Correction: prior section wrongly compared post-disconnect 115 KB with casting. During 480×288 casting `heap_internal` was also only ~49 KB.

Sampling (2 kHz, 1024×576, real bitstream): CABAC parse ~44%, of which `DecodeBinCabac` 24%; IDCT ~12%; motion compensation ~9%; ROM memcpy/memset ~7.7%; pack ~7%.

Changes:
- LVGL draw buffers moved to PSRAM (`bsp_display_start_with_config`); `heap_internal` ~40 KB → ~108 KB so L2 256 KB fits.
- `CONFIG_SPIRAM_XIP_FROM_PSRAM`, L2 256 KB written to `sdkconfig.defaults`.
- CABAC engine switched to 32-bit ops, refill 16 bits at a time (`components/openh264_sw/cabac_decoder_esp.cpp`, `cabac_bin_inline.h`); 3 residual call sites inlined.
- OpenH264 -O3; `cp_i420_pack` word writes; decode pinned core 1, display pinned core 0.

Decode latency (each row builds on prior; first four rows phone-measured, content varies each time):

| Step | Decode mean/peak |
|---|---|
| After L2 line 128 B | 278/453 ms |
| L2 256 KB | 269/459 ms |
| XIP | 248/411 ms |
| 32-bit CABAC | 211/372 ms |
| Inline + pack optimize (94-frame baseline) | 185/367 ms |
| -O3 (94-frame baseline) | 178/361 ms |

Pack 22→17 ms. Putting CABAC code in IRAM ineffective (184 vs 185 ms); reverted.

Verification:
- Recorded 157 real frames; old/new decoder per-frame CRC matched (Mac); bitstream not checked in.
- On-board -O3 firmware replayed 94 frames; cumulative CRC `daf95459` matched unmodified upstream decoder.
- Existing decode regressions and host tests passed.

Device caps (simulated drag): 1024×576 @4 fps: input and display both 4.0 fps, `rej=0`; @5 fps: input 4.6, display 3.6, 8 IDR recoveries. Default set to 4 fps.

10-minute soak (`soak-576-4`, 581 s window): 2400 frames, `rej=0`, IDR 1, decoder not rebuilt, decode mean 209 ms, no crash, memory stable.

Not adopted: global -O2 (decoder already -O3; rest not bottleneck); QIO (code already in PSRAM; would need bootloader change).

Unresolved: at 852×480 @6 fps overload once hit `assert failed: usb_dwc_hal_chan_decode_intr` (`motion-480-6-runtime.log`) then reboot and auto-recover. Cause unknown; not reproduced. Assert fires when error interrupt lacks CHHLTD. Did not appear in final-config soak.

Final firmware SHA-256 `1b0b7f69896cf2793565edba9e3a66a3bccea4dcb866ecab618eda388a2f81a2`, 1024×576, 4 fps. Port power-cycle simulated unplug two rounds ~9 s recovery; profile held across reboot.

Unverified: human drag feel, audio, longer run.

## 2026-11-07 non-square pixels: keep layout, raise fps

User asked whether resolution can drop without unbalancing icon proportions, and proposed measuring smoothness after 6-icon / 4-icon layouts — smoothness first.

Finding: phone converts declared physical aspect into logical size. Logical width is max(pixel width, pixel height × physical aspect). Declaring 16:9 for 512×576, 640×576, 1024×288 makes the phone lay out as 1024×576 and unidirectionally compress; PPA can stretch back.

6/4 icon schemes not viable:
- Logical 640×360 is 4×2 = 8 icons; labels almost crushed; 512×288 and 426×240 still 4×2 with overlapping icons.
- Declaring 4:3 or 1:1 changes dashboard layout; square-screen side rail moves to bottom; home still 4×2 overlapping.
- Home grid is decided by iOS; receiver cannot change it.

Simulated drag (~36 s window, home or dashboard page, firmware-injected touch):

| Pixels | Logical layout | Cap | Input | Display | rej | IDR | Decode mean |
|---|---|---|---|---|---|---|---|
| 1024×576 | 1024×576 | 4 | 4.0 | 4.0 | 0 | 0 | 209 ms |
| 1024×360 | 1024×576 | 7 | 6.8 | 6.7 | 0/211 | 0 | 118 ms |
| 1024×288 | 1024×576 | 7 | 6.8 | 6.8 | 0/250 | 0 | 104 ms |
| 1024×288 | 1024×576 | 9 | 8.2 | 6.2 | 36/291 | 12 | 122 ms |
| 640×576 | 1024×576 | 5 | 5.0 | 5.0 | 0/178 | 0 | 128 ms |
| 1024×240 | 1024×576 | 8 | 8.0 | 8.0 | 0/292 | 0 | 88 ms |
| 1024×240 | 1024×576 | 10 | 8.6 | 8.5 | 3/303 | 1 | 95 ms |
| 1024×180 | 1024×576 | 12 | 11.3 | 11.3 | 0/403 | 0 | 68 ms |
| 640×180 | 640×360 | 12 | 11.1 | 11.1 | 0/399 | 0 | 51 ms |

Sharpness: exported frames nearest-neighbor upscaled for inspection. 1024×240 icons and text clear; 1024×180 icons clear, small text soft but readable. Judgment from decoded-frame upscale; PPA real interpolation not photographed.

Changes:
- Profile table gained `aspect_width/aspect_height`; `cp_touch_configure` uses display aspect with independent X/Y scale; tests added.
- Profile key remains height: 400×240 replaced by 1024×240; added `info_180/240/360.plist`. Default `CONFIG_TAB5_VIDEO_PROFILE_HEIGHT=180`.

10-minute soak (`soak-180-12`, 584 s window): 6834 frames, average display 11.5 fps, rej 6, IDR 3, decoders+2, decode mean 69 ms / peak 147 ms, no crash, memory stable.

Final firmware SHA-256 `b5d5a8560670a5bd2c57740e65184e43c708ef0ac545429d26e7a419164851f8`, 1024×180, 12 fps, `probe=0`, held across reboot. Port power-cycle simulated unplug two rounds ~8 s. Sample `artifacts/device-2026-11-07/carplay-1024x180-frames.png`.

Unverified: human drag feel, photographed on-screen sharpness, human touch positions at this tier, physical unplug, audio. USB assert unexplained; did not recur this round.

## 2026-11-07 dual-core decode and video queue

User: still not smooth enough. Research found no ready P4 High+CABAC decoder and no hardware decode.

1024×180 segments (90-frame baseline, single core): parse 30 ms, recon 22 ms, deblock 12.4 ms, total 67 ms. OpenH264 parses the whole slice then reconstructs; recon only reads per-MB arrays and slice constants — so it can run in parallel.

Changes:
- `components/openh264_sw/recon_worker_esp.{h,cpp}`: parse on core 1; recon and deblock on core 0. Notify once per completed MB row; recon blocks when catching up. Recon uses context and `SDqLayer` copies; context 565 KB — copy only the recon-needed range (8.3→1 ms). Only single slice-group I/P slices parallelize; else fall back to serial.
- CMake generates `decode_slice_esp.cpp` hooks; host tests take serial fallback.
- `CP_VIDEO_MAX_UNITS` 3→6. Three frames at 15 fps is only 200 ms; peak frames or TCP bursts overflow and trigger IDR recovery.
- Touch poll period 33→10 ms.

Bit-level verification found and fixed two bugs:
- Recon loop broke at frame end leaving copy MB count 0, skipping deblock. Displayed 47.6 ms was an artifact; added `Tab5ReconSettle`.
- Under `ERROR_CON_DISABLE`, serial parse failure skips recon, but parallel may have started recon — count mismatch caused one extra error. `Tab5ReconFinish` now follows parse result.
Final firmware replayed 200 units (including 19 errors from serial drops); per-frame CRC matched unmodified upstream; cumulative `c0a4cf9d`.

Baseline: decode 67→52 ms, peak 90 ms. P4 L1 data cache shared by both cores; recon actual work 35 ms, no clear contention; trailing ~13.6 ms is deblock that must wait for parse end.

Simulated drag:

| Profile | Cap | Input | Display | rej | IDR | Decode mean |
|---|---|---|---|---|---|---|
| 1024×180, queue 3 | 16 | 14.8 | 13.6 | 22/534 | 7 | 53 ms |
| 1024×180, queue 6 | 16 | 15.4 | 15.4 | 0/561 | 0 | 47 ms |
| 1024×180, queue 6 | 20 | 16.7 | 16.1 | 8/610 | 1 | 51 ms |
| 1024×240, queue 6 | 12 | 11.6 | 11.7 | 0/427 | 0 | 65 ms |
| 1024×576, queue 6 | 6 | 5.8 | 5.1 | 2/69 | 1 | 162 ms |

10-minute soak (`soak-180-16`, 583 s window): 8999 frames, 15.3 fps, rej 0, IDR 0, decoder not rebuilt, decode mean 50 ms / peak 116 ms, no crash, `heap_internal` ~95 KB.

Incident: backup overwrote two same-named CMakeLists; after wrong restore CMake configure failed and cached `TAB5_SOFTWARE_AUTH` flipped OFF — briefly flashed an auth-missing build. Restored ON; that build did not change NVS. Option lives only in `build/CMakeCache.txt`; after clearing the build dir must pass `-DTAB5_SOFTWARE_AUTH=ON` again.

Final firmware SHA-256 `23833896e455c552d2161fd7738b34751cdf1b87e754da783f5bf6d47d83e341`, 1024×180, 16 fps, `probe=0`, held across reboot. Port power-cycle simulated unplug two rounds ~8 s.

Unverified: human drag feel, practical benefit of 10 ms touch period, photographed sharpness, physical unplug for this build, audio. USB assert unexplained; did not recur.

## 2026-11-07 joint analysis: send path, 1024×192, and touch accuracy

User still felt some stutter and blur, provided a real video, asked Codex to analyze then merge a plan. Video carried com.apple.macl (macOS privacy); could not be read; not analyzed.

Codex (gpt-6-astra, read-only) aborted once on quota, finished on second try. Plan lived as 调试优化方案.md in the work directory.

Measurements:
- Phone QP: P-frame median 17 (1024×180), IDR 28, all reference frames.
- PPA bilinear: at 4× vertical, transitions are 4-pixel linear. Scale factors truncated to 1/16; formal tiers 1.25 and 3.75 represent exactly.
- DPI clock 70→35 MHz: decode 51.9→50.0 ms.
- TCP window 5760→32768: arrival P95 no improvement; reverted (single test).
- SWAR vertical luma sharpen: pack 5.3→22.8 ms; not adopted.

Changes:
- `ncm_host.cpp`: after `transmit()` enqueue, `usb_host_client_unblock`. Previously ACK and HID waited for the next USB event, up to 100 ms.
- Default profile 1024×192@16, replacing 1024×180 and 320×192, also as safe fallback. Both are 768 MBs; 1024×192 has 6.7% more rows.
- Touch reports use layout coords: HID logical height 576, `carplay_video_profile_touch_height`, `airplay_identity_config_t.main_touch_height_pixels`. Injected taps hit side-rail buttons.
- IDR request poll 250→20 ms. Under overload rejects 115→54; fps unchanged, latency worse — avoid overload.

Overload (1024×192@20): input 18.2, display 12.8, IDR 55, latency P50 259 ms. Default kept 16.

10-minute soak (`soak-192-16`, 585 s window): 9530 frames, 15.8 fps, rej 0, IDR 0, decode mean 46 ms / peak 106 ms, internal heap ~98 KB, no crash.
Display interval P50 62, P90 77, P95 84, P99 110–117, max 159 ms; ≥100 ms 18–25/1200, ≥200 ms 0; arrival P95 ~68 ms; latency P50 ~69 ms.
Port power-cycle simulated unplug two rounds ~7.5 s. Host tests and decode regression passed.

Final firmware SHA-256 `0f78f92c4377e3a3243cac10ebe9bc1c35ce5fc1bf031b8233b50327ed7e529e`; first run OK with correct profile. Phone then went offline (no USB device, `connected=0`), so post-reboot reconnect was not completed on this formal firmware; same code verified on a hooked build.

Unverified: human feel, physical unplug, audio, USB assert.

## 2026-11-07 sharpness and smoothness: frame pipeline, PIE sharpen, L2 prefetch

User provided a new real video (720×960, WeChat-transcoded, 4.4 s) and noted blur may be from low resolution.

Conclusions:
- Main blur cause is only 192 vertical lines; sharpen helps little. Simulating with a real 1024×576 frame, Y PSNR: 192 baseline 29.6 dB, 192+sharpen 30.1, 192+sharpen+Catmull-Rom 30.3, 240+sharpen 31.6, 288 = 32.9. 320×576 (horizontal crush) 30.0 — rejected for numeric distortion.
- Double ghosting in the video: two positions each about half brightness; most likely camera 33 ms exposure spanning two frames — unproven. Video has duplicated frames; cannot measure cadence from it.

Changes (workspace; not yet formal firmware):
- Frame pipeline: `components/openh264_sw/recon_worker_esp.*`. Two recon contexts; MB arrays doubled (`LAYER_NUM_EXCHANGEABLE` 2, patched headers under `core_inc`); task queue serializes recon, deblock, border expand, frame output. Only complete single-slice frames go through the queue; others drain then run upstream. iOS repeats SPS/PPS every frame, so drain only when content changes.
- Pack moved to core 0: `cp_decoder_set_sink`, `video_output.cpp` `on_frame`.
- PIE: `main/core/pixel_pie.S`, pack with `VUNZIP.8` + `VZIPT.8`, vertical sharpen strength 128, cap 24. C and PIE match byte-for-byte per frame.
- L2 miss triggers auto-prefetch covering all PSRAM: `enable_cache_preload()`.
- CMake patch gains `tab5_replace`; fail hard if upstream does not match.

Device baseline (phone180, 200 units, per frame, no CRC):

| Version | Per frame | Peak | Pack |
|---|---|---|---|
| Before (decode 52 + pack 5.3) | 57.3 ms | 106 ms | 5.3 ms |
| Pipeline | 50.9 | 99 | 6.0 |
| + PIE pack & sharpen | 50.4 | 98 | 4.8 (sharpen only +0.6 ms; pure C was 23 ms) |
| + L2 prefetch | 46.9 | 91 | 3.9 |

Each version CRC’d: cumulative `c0a4cf9d`, 181 frames + 19 error paths, matched upstream. Host tests passed (new sharpen tests).

Reverted (ineffective): IDCT clear-after-read to drop per-MB 768 B clears (became PSRAM misses; +0.9 ms/frame worse); significance-map memset→loop; L1 prefetch (slower); TCP window 23040 + inbox 16 (at 240@16 rejects 15, IDR 13, internal heap down to 98 KB).

Sampling (1 kHz PC sample, both cores): core 1 parse ~45 ms; core 0 recon 25, deblock 14, expand 2, pack 4–6 ms. ROM memset ~13%, mainly coefficient clears and CopyContext — essentially PSRAM misses.

Simulated drag 1024×240:

| Version | Cap | Input | Display | Reject/IDR | Display interval P50/P90/P95/P99 | ≥100/≥200 ms | Latency P50/P90 |
|---|---|---|---|---|---|---|---|
| Before | 16 | 14.5 | 11.2 | 31/31 | — | — | — |
| Pipeline | 16 | 15.3 | 15.3 | 0/0 | 62.5/84/101/142 | 51/2 | 97/262 |
| All changes | 16 | 15.3 | 15.3 | 0/0 | 63/85/98/127 | 44/0 | 85/107 |
| All changes | 18 | 16.1 | 15.6 | 4/3 | 62/75/84/106 | 12/5 | 241/361 |

240@16 has decode headroom (queue avg 0.52); remaining long gaps are arrival-side: phone skips ~1 frame every ~20 (arrival P95 ~120 ms; 192 tier ~68 ms). Cause unknown. Display 15.3 fps — not yet the 15.8 bar.

Status: user asked to pause. Board reflashed formal firmware `0f78f92c`; no test hooks in source. Workspace changes still lack 10-minute soak, decode regression script, and plug tests, and are not flashed as a formal release. During testing the phone dropped USB twice; old firmware also failed to enumerate — physical connection; recovered after user re-seated.

Todo: measure 192@16 on new code as control; investigate 240 arrival drops; then pick default (240@16 sharper, 192@16 hits fps target), run soak/regression/plug, flash formal build.

### Continued: same-condition comparison, profile lock, wrap-up

Picture content can swing decode time ~20% (same 240 home screen: 49 vs 61 ms), so the test harness now exports one frame at drag end to verify content. Exporting at drag start held the serial log lock and created 600–900 ms fake spikes — removed.

Arrival missing frames: comparing phone timestamps at submit, about half are phone not sending (timestamp delta ~125 ms); the other half left on time but arrived ~58 ms late, often after large frames. TCP timer 250→25 ms no change (133 vs 143); reverted. No receiver-side cause found; judged phone cadence — unproven.

Same period, both on home, each drag ~66 s:

| Version | fps | Display interval P50/P90/P95/P99 | ≥100 ms | Latency P50/P90 | Decode mean |
|---|---|---|---|---|---|
| Old code 192@16 | 15.4 | 62/74/88/151 | 29 | 75/103 | 50 ms |
| New code 192@16 | 15.3 | 63/75/86/143 | 30 | 71/93 | 44 ms |
| New code 240@17 | 15.8 | 62/77/85/102 | 13 | 165/294 | 61 ms |

In this window old code was also only 15.4 fps; arrival P95 ~116 ms. Morning’s 15.8 was a better phone state and not reproducible now. 240@17 was the only config hitting 15.8, but ~90 ms more latency. User chose 192@16 — responsiveness first.

Sharpness: one real 1024×192 frame, firmware-byte-identical sharpen, stretched to panel; before/after at `artifacts/perf-tools/锐化前后对比-同一帧.png`. Strokes and digit edges clearly sharper without halo — moderate improvement. For a clear step up, need 240 lines.

10-minute soak (`soak-new192`, 597 s): 9149 frames, 15.3 fps, reject 0, IDR 0, decoder not rebuilt, decode mean 44 ms, peak 109 ms. Across 9 stats: display interval P50 63, P90 74–78, P95 86–89, P99 127–145, max 161 ms; ≥100 ms 25–32/1000, ≥200 ms 0; latency P50 ~71, P90 85–94 ms; queue depth avg 0.51, max 1. Arrival P95 stuck at 114–116 ms — phone-limited.

Plug: firmware cut USB-A power 3 s then restored, two rounds; picture back at 6.0 and 5.9 s (was ~7.5 s). Internal heap 89751 bytes each connect; no leak.

Host tests and decode regression (`scripts/test_decoder.sh`) passed. Device per-frame CRC under pipeline/PIE/prefetch versions all `c0a4cf9d`.

Formal firmware SHA-256 `43ee9094cef6d2d431e07ff2dd6de72c44aad235f7cf2fe6042fb08e8ccb7063`, 1024×192@16, `TAB5_SOFTWARE_AUTH=ON`, no test strings. Flashed; picture OK after phone connect. Prior binary kept under `artifacts/snapshots/pre-sharpen-2026-11-07/`.

Test tools: `artifacts/perf-tools/` added `hook_run.sh` (independent build dir, hooks auto-revert, optional frame export / missing-frame log / plug), `bench_run.sh` (offline bench, optional stage timing, dual-core PC sample, cache mode, CRC), `symbolize.py` (dual-core hotspots, ROM callers).

Unverified: human drag feel, on-device sharpened look, audio. 240@17 can be restored anytime by changing `info_240.plist` maxFPS and default height.

### Switched to 1024×240@17 (user request)

Changes: `CONFIG_TAB5_VIDEO_PROFILE_HEIGHT` and Kconfig default → 240; `info_240.plist` maxFPS → 17; profile table fps synced to 17. 240 remains trial; if first 100 frames reject >20%, auto-fallback to 1024×192. Stale 320×192 wording in Kconfig help → 1024×192. Host tests, plist check, decode regression passed.

10-minute soak + two plug rounds (`soak-240-17`, 591 s window): 9271 frames, 15.7 fps, in-window reject 0, IDR 1, decoder rebuild 1, decode mean 62 ms, peak 143 ms. Across 9 stats: display interval P50 62–63, P90 73–77, P95 81–86, P99 90–112 ms; ≥100 ms 0–15/1000, ≥200 ms 0. Latency P50 147–219 ms, P90 231–346 ms, queue avg 1.3–2.4. Plug two rounds ~6 s; internal heap stable ~89.6 KB.

Vs 192@16 soak: long gaps more than halved, better P99, fps 15.7 vs 15.3; cost ~100 ms more latency and running against capacity:
- On each link-up, phone’s initial burst fills the 6-frame queue → 1 extra reject+IDR. 192 does not.
- Drag start peak overflowed once.

Tried queue 8 (`soak-240-17-q8`): still overflowed on link-up; in-window rejects became 2; latency P50 up to 315 ms. Reverted; keep 6.

Formal firmware SHA-256 `2999f5f5b92791a030947e9bf109900fb18b68a1de4f653e36de4fa3f5a92340`, same source as soak (minus test hooks), 1024×240@17, auth ON, no test strings. Flashed; picture OK after phone connect.

Alt: 240@16 (post-pipeline measured 15.2–15.3 fps, reject 0, latency P50 ~85–125 ms) more stable but slightly lower fps. Returning to 192@16 only needs default height → 192.

### Switched to 1024×240@16 (user final choice)

Changes: `info_240.plist` maxFPS and profile table fps → 16; height stays 240. Plist check, host tests, decode regression passed.

10-minute soak + two plug rounds (`soak-240-16`, 597 s window): 9031 frames, 15.1 fps, in-window reject 0, IDR 0, decoder not rebuilt, decode mean 62 ms, peak 137 ms. Across 9 stats: display interval P50 63–64, P90 79–82, P95 87–93, P99 117–124 ms; ≥100 ms 21–36/1000, ≥200 ms 0. Latency P50 118–127, P90 153–179 ms; queue avg 0.8, max 2–3 (was 1.3–2.4 / max 6 at 240@17). Plug two rounds ~6 s; internal heap ~94 KB, stable.

Leftover: at 240, within 1–2 s after each link-up there is 1 reject + extra IDR (both 16 and 17; not on 192). Shows as one hitch right as picture appears, then gone. Suspected first IDR plus decoder-init burst; not investigated.

Formal firmware SHA-256 `e673e1880020e366ba2aaa2ae91bebe327f6d4c965673bfe8261e8b22dd894b3`, 1024×240@16, auth ON, no test strings. Flashed; picture OK after phone connect.

## 2026-11-08 side rail enlarge: 890×240 layout

User said side rail too small; asked to enlarge relative to right-side icon proportions.

Investigation: comparing prior exported real frames at widths 426, 512, 640, 1024 — iOS side rail is always ~90 logical px wide; side icons ~half home icons; home icons also fixed logical size. Declared params are only size, aspect, and safe area — no separate side-rail control. Hard-stretching the rail on the receiver only widens it horizontally (rail is full height) and distorts icons — not viable. User chose ~15% overall enlarge.

Changes:
- 240 tier → 890×240, declare 16:9, logical layout 890×500. `info_240.plist` width/view/safe areas → 890; touch descriptor range → 890×500 (firmware checks descriptor matches “profile width × touch height”).
- 1280/890 maps to PPA 23/16, output 1279 columns, no letterbox; vertical still 3×. Macroblocks 960→840.
- 1024×240 plist and profile table backed up under `artifacts/snapshots/pipeline-2026-11-07/`.

On device: phone accepted 890×240; frame export showed side rail, clock, recent apps, and home icons enlarged at the same ratio; home still 4×2. Comparison image: `侧栏放大前后对比.png` (local, not committed).

Soak (`soak-890`, ~10 min drag, ~8000 frames): reject 0, only 1 initial IDR at start, decoder not rebuilt, decode peak 163 ms. Across 8 stats: display interval P50 66–67, P90 81–84, P95 91–99, P99 119–137 ms; ≥100 ms 28–46/1000, ≥200 ms 0; queue avg 0.8–1.1. User ended during plug phase; plug test incomplete.

Formal firmware SHA-256 `c102a8479e9192ea5ba49bdf325bc1d4a82464fc08814c20f2f1bfe05c5155da`, 890×240@16, auth ON, no test strings. Host tests, plist check, decode regression passed. Flashed, but after flash no phone on USB (`connected=0`); picture not yet seen on this formal build.

Phenomenon (unexplained): after long USB-A power-off (offline bench firmware, interrupted plug hook), iPhone does not self-re-enumerate when power returns — three times needed a physical reseat. Firmware’s short 3 s power cuts recovered in ~6 s both rounds.

Unverified: picture and real touch positions on this formal build (touch descriptor now 890×500), plug recovery, human feel.

### 890×240 formal build confirmed on device (2026-11-08)

Picture: after reseating the phone, `c102a847` showed picture, `size=890x240`, 0 reject, 0 decode error. Opening serial reboots the board once; ~8 s to picture after reboot (`release-890-check.log`).

Human physical unplug two rounds (`release-890-replug.log`, 180 s):
- From first USB enumeration at insert to `VIDEO: START`: boot 7.8 s, round 1 6.2 s, round 2 5.6 s.
- Whole run reject 0, decode error 0. One initial IDR per session; no extra post-link reject this time.
- On unplug iAP2 and USBMUX closed cleanly (`service_closed`), then re-enumerated and selected NCM.
- Memory: both rounds at picture — internal heap 98.7 KB, PSRAM 13.30 MB; after unplug heap returned to the same values; no leak.

User confirmed: human tap positions (touch range 890×500) and side-rail size both OK. 890×240@16 locked as current formal release.

## 2026-11-08 power-off after unplugging USB-C: battery never charged

Symptom: Tab5 has an NP-F550 battery, but unplugging USB-C powers off; power button cannot start.

Cause: neither the board support package nor our firmware configured the second IO expander (0x44) charge pins. Power-on default is high-Z with pull-down; charger stayed off; battery was drained during prior long tests. First new-firmware reading: battery 1.1 V (protection open); after charge and rest 6.61 V → 0% by M5Unified algorithm (single cell 3.30–4.15 V).

Pins and conversion (per M5Unified `Power_Class.inl` Tab5 init):
- P7 CHG_EN: high enables charge.
- P5 nCHG_QC_EN: low selects 1000 mA; high selects 500 mA.
- P6 is IP2326 BAT_STAT: may be high both while charging and when stopped — cannot use alone to detect charging.
- INA226 (0x41): shunt 5 mΩ; charge current reads negative.

Process:
- P7 high only: first power-up charged ~470 mA. After reboot charging stopped; toggling P7 every 20 s did not restore (diagnostic build deleted).
- Following M5Unified, also set P5 output low: charge recovered ~955 mA. After software reboot charging auto-resumed (`charge-m5cfg`, `charge-reboot`). In 2 minutes voltage 6.94→7.29 V, charge current stable 957 mA. CarPlay video OK, 0 reject.

Changes: added `main/power_status.{c,h}`. At boot enable charge, select 1000 mA; every minute log one `POWER: STATUS` (charge enable, BAT_STAT, battery voltage, charge current). Original `app_main.c` and `main/CMakeLists.txt` backed up under `artifacts/snapshots/pre-charge-2026-11-08/`.

Formal firmware SHA-256 `b794315bc90dd807678b2b2c5bcfe94f7cc6aa222025b6e5583681c97b194f74`, 890×240@16, auth ON, no diagnostic code. Host tests passed.

On-device verify (battery ~7.3 V, not full): after unplugging USB-C screen stayed on — no blackout. After power-off/on, phone connect OK (user confirmed). Serial dropped on unplug; no log captured while unpowered. After replug: battery 7.67 V, charge 959 mA. Picture 6.9 s after phone enum, 0 reject.

To watch: full charge (~8.3 V) needs another 1–2 hours plugged; runtime after full not measured.

## 2026-11-08 audio stages 0–1: after declaring mic input, phone hands audio to Tab5

Design in [docs/audio-io-design.md](audio-io-design.md). Across prior 142 sessions the phone never opened an audio stream to Tab5; QQ Music stayed on the phone.

Changes:
- `/info` (`scripts/configure_tab5_info.py` for all tiers): aligned with DiPlay to 8 declarations. type 100: compatibility, default, alert, media, telephony, speechRecognition; type 101: compatibility, default. PCM only; 4 with `audioInputFormats` (8–48 kHz mono). `extendedFeatures` added vocoderInfo. plist 1850 bytes.
- Crypto keys: added derivation for `DataStream-Input-Encryption-Key`.
- `airplay_audio.c`: accept type 101 and 8–48 kHz PCM. When phone requires input, upload encrypted **silence** packets to its `dataPort`, logged `uplink=silence`. Uplink starts on first downlink packet; if phone sends no audio, start 500 ms after SETUP. If speaker busy or sample rate unsupported, stream still received but not played (`sink=none`).
- Logs: phone command types and parameter names; each audio SETUP’s format, input, phone port, and field names.

On device (`audio-stage1`, firmware 5d5dbc33, test build):
- ~10 s after connect, phone opened a `media` stream (48 kHz stereo PCM). **User confirmed QQ Music played from the Tab5 speaker.** ~170 s received 23357 packets, lost 0, reordered 0.
- Siri: phone opened `speechRecognition` (24 kHz mono with phone port). Silence uplink every 20 ms — 493, 307, 238, 314 packets sent, 0 failures. After Siri ended, phone rebuilt media; music continued. Also briefly saw a 48 kHz mono speechRecognition stream ~70 ms.
- Phone commands: `modesChanged` 18×, `disableBluetooth` 1×.

Issues found (stage 2):
- Siri 24 kHz did not play: speaker currently accepts only 44.1 and 48 kHz → `sink=none`.
- Playback drops: `tab5_audio_submit` drops when it cannot take the lock. One media stream dropped 150 packets; others 0–6.
- Video: 1 reject at 230.6 s, then IDR + decoder rebuild — coincided with the 150-drop media stream. Whether audio load caused it needs a controlled compare.
- On every stream switch, one `i2s_channel_disable: not enabled` error; no observed impact yet.
- SETUP `input` field is neither bool nor int — type TBD. Currently decide mic-on from audioType.

Note: this is a test build declaring a mic while uploading only silence — Siri cannot hear the user. Not a formal release.

## 2026-11-08 audio stages 2–3: real mic, Siri audio, dual-stream mix

Changes:
- `main/audio_output.c` rewrite:
  - Speaker and mic fixed at 48 kHz stereo full-duplex. One task mixes and writes speaker every 10 ms, then reads the same length from the mic.
  - Two playback paths: 0 = main (music, Siri, call), 1 = navigation prompts. During nav duck, path 0 attenuated to ~30%.
  - Each path has 0.5 s jitter buffer (60 ms prebuffer). When backlog >180 ms or <20 ms, read one extra/fewer frame to follow phone clock.
  - When buffer empty and a new packet arrives, restart playback from that packet (handles pause timestamp mismatch).
  - On receive, wait for lock up to 5 ms.
  - After 3 s idle, close codec.
- `main/core/audio_resample.{c,h}`: playback linear-interpolates 8–48 kHz → 48 kHz; mic averages down to 8/16/24 kHz. Host test `tests/test_audio_resample.c`.
- `airplay_audio.c`: uplink uses real mic (average of two mic channels), paced by mic clock; silence only when no mic. Forward duckAudio / unduckAudio.
- `/info`: mic input formats drop 32 kHz; keep 8, 16, 24, 48 kHz — integer divisions of 48 kHz.
- `cp_pcm` supports 8–48 kHz.

On device (`audio-stage23`, firmware 3f2da713):
- **User confirmed: music plays from Tab5; Siri understands the user and answers from Tab5.**
- Music: continuous 63 s plus ~20 s after — underrun 0, reject 0, restart 0, lock-wait fail 0, no clock correction. Buffer stable 63–75 ms. Prior build had a 150-drop segment; this time 8540 packets 0 errors.
- Siri: 24 kHz mono, uplink 512 packets, 0 fail, 0 wait. Mic peaks: silence ~1600–2200, speech 23018 (30 dB gain), no clipping. Siri replies played from Tab5, underrun 0.
- Video: 1 reject within 1.5 s of connect (known 240-tier behavior). Another reject+IDR at 69 s while music played. `decode_max` 186 ms, `ui_max` 176 ms — both higher than without audio (~92 and 69 ms). Whether audio slowed video needs same-condition confirmation.

Unverified: nav-prompt mix and duck (not triggered), telephony, long run. Speaker volume still 30.

### Video impact test (incomplete) and USB assert reproduce

Method: test firmware auto left/right swipes home for 120 s (`hook_run.sh`). A plays music; B pauses music; otherwise identical.

- A (`av-music`, music throughout, audio underrun 0): 13.7 fps, reject 0/1607. Decode avg 60 ms, max 129 ms. Display interval P50 71, P90 96, P95 107, P99 129 ms; ≥100 ms 81/1000, ≥200 ms 2.
- B (`av-quiet`): **invalid**. Log showed music still playing (peak 29105, packets arriving). After ~96 s board assert-restarted: `assert failed: usb_dwc_hal_chan_decode_intr usb_dwc_hal.c:504 (chan_intrs & USB_DWC_LL_INTR_CHAN_CHHLTD)`. Last 36 s before restart: 13.6 fps, decode avg 59 ms — same as A. Internal heap before restart 63 KB, ~35 KB less than ~98 KB without audio.
- Prior no-audio 890×240 data ~15 fps was old firmware at a different time — not directly comparable. Need a real music-paused B.

Assert cause: ESP-IDF USB host HAL asserts and reboots when a channel reports BNA (Buffer Not Available) without Halted — see espressif/esp-idf#19111. Upstream removed the assert on master in 63b468539f (“Do not abort on pipe error without halt INTR”); release/v5.5 (including local v5.5.3) not yet fixed. BNA means RX descriptors not refilled in time; audio load may make it more likely. Previously also appeared under overload.

Todo:
1. Fix in-project via linker `--wrap` of `usb_dwc_hal_chan_decode_intr` without editing shared ESP-IDF.
2. Confirm NCM and USBMUX recover after pipe errors instead of wedging.
3. Redo a true music-paused B.

### USB assert fix and video-impact conclusion

Fix (firmware 0ffd9792):
- `main/usb_hal_fix.c` replaces ESP-IDF’s function via `-Wl,--wrap=usb_dwc_hal_chan_decode_intr` without editing shared ESP-IDF. When a channel errors without Halted, no assert: actively halt the channel, then report the error once Halted arrives. BNA remapped to transaction error because espressif/usb 1.6.0 HCD `abort()`s on URB_NOT_AVAIL for BNA. Disassembly confirmed HCD calls the new function.
- `ncm_host.cpp` and `usb_probe.c`: after NCM link error, previously stuck until manual unplug. Now power-cycle the USB port 0.5 s (auto unplug), max 3× per minute. STATUS log gained `usb_hal_recovered` counter.
- Host tests passed. Error is intermittent; this round did not trigger (`usb_hal_recovered=0`), so recovery path not yet exercised on device.

Video compare (same firmware, same swipe script, 120 s each):
| | A music playing | B no audio stream |
|---|---|---|
| Display fps | 13.7 | 15.0 |
| Decode avg/max | 60/129 ms | 64/158 ms |
| Arrival interval P50/P90/P95 | 64/121/124 ms | 63/73/120 ms |
| Display interval P50/P90/P95/P99 | 71/96/107/129 ms | 66/78/88/127 ms |
| ≥100 ms / ≥200 ms (per 1000 frames) | 81/2 | 23/4 |
| Avg queue depth | 0.61 | 1.05 |

Conclusion: music costs ~1.3 fps. Not decode CPU: decode time did not grow; queue was emptier. Slowdown is frame arrival: arrival P90 73→121 ms, with many ~120 ms gaps. Possibly phone sends video less evenly while sending audio, or our USB/network RX path slows when audio packets mix in. Next: compare phone send timestamps.

Also: after prior B crash-reboot, iPhone did not re-enumerate — needed manual reseat (known). After tests, reflashed hook-free 0ffd9792; phone reconnected and showed picture.

### Music slows video: cause is on the phone

Method: test firmware frame-gap probe (`add_gap.py`). Log when arrival interval or phone send timestamp (pts) interval ≥95 ms; split into:
- **Phone skip**: pts gap ~125 ms — phone never sent a frame.
- **On-time but late**: normal pts gap, arrival ≥95 ms.

Results (`gap-music`: music started 18 s after swipe began, so same run has both segments):
| | fps | Phone skips | On-time but late |
|---|---|---|---|
| Same run, first 18 s no music | 14.6 | 0.99/s | 0.88/s |
| Same run, next 104 s with music | 13.6 | 2.38/s | 2.25/s (median 118 ms; only 13 followed large frames) |

Suspected Tab5 NCM RX: only one outstanding 4 KB bulk IN; with music each USB transfer averaged 1.16 packets, ~230/s. Changed to 4 concurrent transfers (`gap-music-rx4`, firmware b2a11fc9): 13.6 fps, phone skips 2.41/s, late 2.12/s — **no change**.

Conclusion: Tab5 USB RX is not the bottleneck. Extra skips and late arrivals under music happen before the phone sends — likely phone encode/send cadence worsens while transmitting audio. 4-transfer change reverted; rebuilt firmware byte-identical to 0ffd9792; reflashed; picture OK.

Possible next tries (not done):
- Compressed music (type 102 AAC — needs AAC decode on Tab5) to cut phone PCM load.
- Declare lower media sample rate.
- Compare another phone or iOS version.

### Volume and brightness in CarPlay (firmware 7bfb56f3)

Background: CarPlay has no receiver volume bar; phone volume keys only change phone output; Tab5 speaker was fixed at 30.

Approach:
- `/info` declares OEM icon (`oemIconVisible`, `oemIconLabel = "Tab5 设置"`, `oemIcons`, 128×128 PNG at `assets/oem_settings.png`). CarPlay home gains a “Tab5 设置” icon; tapping it makes the phone send `requestUI`. info_240 is 3172 bytes, still under the 3500-byte cap.
- `main/app_ui.c` on `requestUI` stops video display and opens a local LVGL settings page: volume (0–100), brightness (10–100) sliders, and a “Back to CarPlay” button. While open, touch is not sent to the phone. On return, request an IDR so picture resumes immediately.
- `main/audio_output.c` added `tab5_audio_set_volume()` and `tab5_audio_volume()`; audio thread applies `esp_codec_dev_set_out_vol` within 10 ms.
- Settings saved in NVS namespace `tab5_ui` (keys `volume`, `brightness`) on finger-up; read at boot. Only `nvs_flash_init()` — no NVS erase.

Result: user confirmed usable — icon appears, settings open, volume/brightness change live while dragging, picture resumes immediately on return.

## 2026-11-12 offline logging: capture disconnects and crashes without a PC (firmware 894cf797)

Background: occasional disconnects during navigation with no PC on site — all serial logs lost.

Approach (factory firmware only; no partition-table change; NVS untouched):
- `main/sd_log.c`: `esp_log_set_vprintf` intercepts all ESP_LOG; still prints serial, also writes a 128 KB ring in PSRAM. Buffer in `EXT_RAM_NOINIT_ATTR` (new `CONFIG_SPIRAM_ALLOW_NOINIT_SEG_EXTERNAL_MEMORY`); survives reboot; boot memtest skips it.
- Background task (prio 2) every 5 s appends new logs to microSD: `/sdcard/TAB5LOG/LOGnnnnn.TXT`. New file each boot; header records firmware version and reset reason. Rotate at 8 MB; keep max 100; delete oldest. Runs without a card; remount retry every minute.
- Disconnect events: USB disconnect (`usb_disconnected`), NCM auto-recovery (`ncm_recovery`), AirPlay session end (`airplay_stop`) log one `SDLOG: EVENT` with memory data and flush to card immediately.
- Crash: wrap `esp_panic_handler` at link time; before ESP-IDF print/reboot, store crash reason, abort/assert text, task name, registers, and 32 stack words in internal RAM noinit. Then force write-back L1 and L2 into PSRAM — ESP-IDF reboot path only writes L1. Next boot writes these into the log and recovers any pre-crash logs not yet flushed.
- Only panic and watchdog resets trust the PSRAM buffer. Other resets do not write back cache; rely on the 5 s flush.

Verify (temporary test build that aborted ~22 s after boot; test code removed):
- Reset reason `panic`; crash reason, `abort() was called at PC ...`, task `main`, registers and stack recorded correctly.
- First version skipped L2 write-back; recovered buffer content wrong after reboot (several times exactly 821 bytes). After L2 write-back, a marker printed 1 s before abort was fully recovered (`recovered marker found`).
- Card present; LOG00001–LOG00008 created normally.

Note: once mistakenly flashed the test build as formal (forgot rebuild after removing test code); found, rebuilt, and reflashed.

Formal firmware SHA-256 prefix `894cf797`, flashed, boot OK. After flash phone did not re-enumerate (known); card write impact on video with picture up not measured.

Not covered: power-loss / brownout may lose last <5 s of logs. Task WDT hints that go through ROM print bypass the log hook and never enter the buffer.

### Navigation disconnect: card readback and analysis (2026-11-12)

Read method: card stayed in Tab5; temporary card-reader firmware dumped TAB5LOG files numbered 9+ over serial, then reflashed formal 894cf797. Four files size-matched the card; saved under `artifacts/device-2026-11-12/sdcard/` (local, device logs, not committed). Note: console converts newlines to CRLF — normalize before splitting.

Timeline:
- LOG00009 (~36 min run):
  - ~416 s, right after a nav prompt ended and duck again, `usb_hal_recovered` 0→1 (`usb_hal_fix.c` caught a USB channel error). Then NCM `rx_frames` stuck at 40564; video stats also stopped; no `NCM: RX status`; auto-recovery not triggered. Picture frozen ~11 min.
  - During freeze internal heap ~52 KB → ~26.7 KB.
  - 1078 s: USBMUX RX error; session ended; phone re-enumerated. 1080 s NCM recovery (first real on-device success); video resumed.
  - ~1860 s: NCM RX stopped again; this time `usb_hal_recovered` unchanged.
  - 2157 s: USBMUX and NCM RX errors together; triggered NCM recovery (port power cut); then crash-reboot.
- LOG00010: normal casting ~7 min. At 436 s USBMUX and NCM RX errors together (`usb_hal_recovered=0` — looks like phone drop or bus-level error); NCM recovery; then crash-reboot.
- Both crashes identical: `usb_lib` task, `abort()` at `managed_components/espressif__usb/src/hub.c:681` `root_port_recycle`. Port state not ENABLED/SUSPENDED/RECOVERY because we had just powered the port down.

Conclusions:
1. **Reboots were caused by our own NCM recovery.** While the phone was already gone and the USB stack was recycling the device, `recover_ncm()` powered down the root port again; recycle hit an “unpowered” state and aborted.
2. **Freeze was NCM RX wedged and undetected.** After a channel error was caught, RX transfers never completed again; `ncm_host` got no failure callback, so recovery never fired.
3. Secondary: occasional `airplay_screen: screen_message=discarded error=ESP_ERR_NO_MEM`; each nav-prompt stream open produces two `i2s_channel_disable ... not been enabled yet` lines; during freeze internal heap down to ~27 KB.

Suggested fixes (not done):
- Before recovery wait ~1 s; confirm phone still present and port not already in USB-stack recovery; if phone already gone, do not intervene.
- During casting, if NCM receives nothing for ~3 s, treat as fault and recover.

### Navigation disconnect fixes (firmware 43b7c4bc)

User added: first time picture froze, recovered after manual unplug; second time CarPlay UI exited, also needed manual unplug.

Second “UI exit” cause (new finding, unrelated to USB):
- At 1860 s phone sent a nav-voice SETUP; Tab5 never called audio start (no `AUDIO: setup` in log); then phone tore down the whole session (`screen_peer=disconnected`, `rtsp_client=peer_closed`).
- Cause in `airplay_rtsp_session.c`: each stream’s connection ID is kept in a 32-slot table to refuse reuse of retired IDs. Entries are only marked invalid, never deleted. Every nav voice is a new type 101 stream. In the second session: 1 main audio + 1 screen + 30 nav voices = 32 slots. On the 31st nav voice SETUP returned 503 and the phone exited CarPlay.

Changes:
1. **Stream ID table**: added `components/airplay_session/airplay_media_keys.{c,h}`. When full, evict oldest retired entries; only still-active streams consume capacity. Recently retired IDs still refuse reuse. Rejects log reason: `setup=rejected reason=retired_stream_id` or `too_many_active_streams`. New host test `tests/test_media_keys.c`: 200 consecutive nav voices, eviction, capacity, reuse, and type changes all passed.
2. **NCM recovery** (`main/usb_probe.c`): on error record `ncm_fault`, wait 1 s. If phone already disconnected itself, skip (`NCM recovery skipped`) and let USB stack handle; only power-cycle the port if phone still present. Avoids `hub.c:681` abort.
3. **RX watchdog** (`main/ncm_host.cpp`): enable after 100 cumulative transfers. If then nothing received for 5 s, treat as link fault and take path 2. Field logs show normal casting gets ≥186 frames per 10 s.

Verify: all host tests passed; firmware built (auth ON). Flashed; boot OK (LOG00014). After flash phone did not re-enumerate — needed manual reseat. Still unproven on device: >30 nav voices, skip recovery on unplug, 5 s auto-recover after stall.

#### First on-device result (LOG00014)

- Phone inserted ~18 min after boot; casting ~7 min (1111–1531 s). No reject, no decode error, no crash, `usb_hal_recovered=0`. 6161 frames, ~14.7 fps. Internal heap stable ~52 KB.
- No nav voice in this window: no type 101 SETUP, no duckAudio. No unplug, no RX stall. None of the three fixes triggered — cannot count as verified.
- After reading logs, reflashed 43b7c4bc (LOG00015 is that reboot).


## 2026-11-13 USB stall fix and on-device verification

Pre-check: read device factory partition, truncate to valid ESP image length; SHA-256 `43b7c4bc0b7c2281263d77b7dfb923da24a20f23efab87ea15075881bab5dda8` — confirms device was running the nav-voice fix. Backup plus this build/flash/serial evidence under `artifacts/device-2026-11-13/usb-recovery/`; full partition backup sibling `pre-usb-recovery-factory.bin`; local only.

Log review: in LOG00016 after `usb_hal_recovered` → 1, TX queue hit `tx_pending=8`, at 295 s printed `RX stalled`, then USB STATUS vanished. Code’s `ncm_host_poll()` sync-halts endpoints before notifying probe — explains missing recovery records. LOG00017 long navigation success is carried forward from user field report; not treated as a soak for the new firmware.

Changes:
- HAL checks real CHENA: if channel errored and hardware already stopped, return error immediately; only if still running request halt and wait for interrupt. BNA still mapped to transaction error to avoid HCD BNA abort. When error and sync cancel race, return HALT_REQ to wake HCD waiters; clear leftover deferred error on each channel activate so reconnect reuse is not polluted.
- NCM fault only stops new submits and reports; no sync halt in the fault poll. Added single TX transfer 3 s timeout; keep existing 5 s RX watchdog.
- Recovery waits 1 s to see if the device leaves by itself. If still online, operate Tab5 USB-A 5V switch; wait for real DEV_GONE before cleaning endpoints; after cleanup wait 0.5 s then restore power. Config switches use the same path; keep max 3 NCM recoveries/minute. Pin is BSP_USB_EN — see [M5Stack Tab5 official pinout](https://docs.m5stack.com/en/core/Tab5). No longer fake gone early or set HCD root port unpowered directly.
- Added independent usb_guard task: if USB main loop heartbeat stale 15 s, or recovery unfinished 15 s, log `usb_progress_timeout`, notify SD write, wait up to 0.5 s, then reboot. Missing/corrupt SD must not block reboot; card flush is best-effort — not guaranteed on a bad card.

Stage verification:
- Host regression passed; new HAL tests cover stopped-channel error, still-running deferred report, channel reuse, error vs cancel race, normal completion.
- Formal build with `-DTAB5_SOFTWARE_AUTH=ON`; keep nav-voice stream-ID recycle and prior auth config. Wrote only `0x10000` factory; preserved NVS, partition table, and identity material.
- Independent guard on-device inject: temporarily blocked usb_probe; at 15299 ms timeout logged, then reboot with `reset_reason=software`; USB probe started normally. Evidence `guard-test-runtime.log`; not equivalent to real USB fault recovery.
- First link-inject round’s extra boot power-cut happened during first enumeration and exhausted the mode-request budget — round invalid. Extra boot power-cut limited to when no device has been discovered yet; later results below.

#### On-device inject results for three USB faults

Valid rounds used `link-test2.bin`; evidence `link-test2-runtime.log`. Only in the temporary image, real NCM channel interrupt handling was rewritten: first, rewrite a completed-but-already-stopped TX channel event as BNA without CHHLTD; second, swallow one TX completion; third, swallow one RX completion. Last two keep HCD active state and original in-flight transfers until real disconnect cleanup — covering “completion IRQ that never comes,” not merely calling the recovery function.

| Injected fault | Observation | Fault detected → NCM recovery / video growth again |
|---|---|---|
| Stopped TX channel BNA without CHHLTD | Immediate `NCM: TX status=1`, HAL error count → 1 | 3.01 s / 8.86 s |
| Drop TX completion | `TX stalled` at 3066 ms, `tx_pending=8` | 3.27 s / 9.09 s |
| Drop RX completion | `RX stalled` at 5042 ms, `rx_busy=1` | 3.24 s / 9.18 s |

All three first logged ncm_fault, ~1 s later cut VBUS, got real disconnect and cleaned up, restored power; phone auto recompleted NCM/CarPlay; USB STATUS kept printing. No abort/assert, no extra reboot, no decode error or video reject. Video recovery time is when serial `VIDEO: STATS` decode counts grow again after reconnect — not a human-measured screen latency. Real sporadic USB electrical faults and multi-hour navigation still need later soak.

Removed all TEST_ONLY / USB_TEST injects from source; rebuilt formal image, SHA-256: `cb916322a6c038226f325aebb05197050025233df8f83ee5553ff37cee3581d3`. `build/carplay_tab5.bin` byte-identical to `artifacts/device-2026-11-13/usb-recovery/release.bin`. Disassembly confirmed HCD calls the new decode and activate wrappers; full host regression passed again.

#### Final formal release status

Formal image `cb916322` written to device; esptool returned `Hash of data verified`. Final serial capture 150 s; observed video ~137.7 s, cumulative decode 2024 frames, reject 0, decode error 0, UI fail 0; USB STATUS continuous with `usb_hal_recovered=0`; no fault recovery or abnormal reboot. Evidence: `final-flash.log`, `final-runtime.log`, `final-sha256.txt`, `verification.json`.

Device left on formal build; source and image confirmed free of TEST_ONLY inject markers. Kept all prior uncommitted changes; no Git commit this round. Verified: host regression, three controlled USB faults, independent task-stall fallback, and short normal casting. Not claiming multi-hour navigation / real sporadic hardware-fault soak. Known I2S stream-open log noise, occasional screen OOM, and cold-boot reset_reason naming unchanged this round.

### 2026-11-13 evening music + navigation soak card readback

Readback confirmed device on formal `cb916322`; exported LOG00022–34, 13 files, 2,250,668 bytes; per-file length and SHA-256 verified. Core soak file LOG00030: ~60 min log, 100 nav stream open/close, 0 rejects; no TX/RX stalled, USB task stall, or guard reboot reproduced.

**This round still had one non-manual drop — cannot call zero-drop pass.** User confirmed the ~4:26 disconnect was not manual; the ~53:57 disconnect was the manual end of the test. After the first drop the device cleaned up normally, the app skipped a redundant power cut, the phone re-enumerated, and video resumed in ~16.87 s; then ~49 min continuous. Existing logs cannot distinguish phone vs contact/power vs host-port error. `usb_hal_recovered=0` does not mean all USB hardware errors were zero.

Also 9 recoverable video rejects (logged as ESP_ERR_NO_MEM), decode error 0; music and one nav segment each recorded 2 missing RTP packets; during continuous play underrun ~14.75 ms and 15 ms respectively; memory overall stable ~47–48 KB. Full evidence and recommendations in [docs/sd-log-analysis-2026-11-13-soak.md](sd-log-analysis-2026-11-13-soak.md).

After readback, reflashed original `cb916322` image, write verified, boot OK. Local build/carplay_tab5.bin restored to the same formal image; business firmware, NVS, and identity untouched. Raw data, hash list, and export scripts under `artifacts/device-2026-11-13/soak-readback/`.

### 2026-11-13 iPhone 12 Pro Max / iOS 16.1.1 switch-phone failure diagnosis

Screenshot showed Pairing storage unavailable. Serial confirmed USB/NCM ready, Lockdown QueryType OK; an existing trusted=pending pairing record for the current phone. After user confirmed Trust, updating the record returned `ESP_ERR_NVS_NOT_ENOUGH_SPACE`, then Failed error=5.

NVS is only 24 KiB, used=486/free=270, already two 6768-byte pairing blobs. `trusted=1` calls existing save and rewrites the whole blob; NVS copy-on-write needs old and new versions simultaneously and fails for space. Switch-phone capability exists in logic, but current storage limits second-phone state updates; earlier “just switch phones” answers must be corrected accordingly.

Recommend keeping existing identity blobs, splitting trusted into a small independent key, and handling legacy format, invalidation, and identity-regeneration consistency. Did not clear NVS, change business code, or flash. This round cannot yet judge that phone’s TLS/iAP2/media compatibility. Full analysis in [docs/iphone12-pairing-storage-failure-analysis-2026-11-13.md](iphone12-pairing-storage-failure-analysis-2026-11-13.md); evidence under `artifacts/device-2026-11-13/iphone12-pairing/`.


#### iPhone 12 Pro Max recovered after clearing old pairing

User explicitly authorized clear-and-re-pair. Used existing maintenance switch to clear only tab5_pair; log confirmed result=ESP_OK; other NVS namespaces preserved. Then restored formal firmware cb916322, write verified, NVS used=54/free=702, pair blobs=0.

After reconfirming phone Trust: PAIR_OK, Lockdown/CarKit TLS_READY, IAP2_AUTHENTICATED all succeeded; Streaming at 47.428 s; user confirmed CarPlay picture; first music segment 3361 packets with no bad/lost/reordered/underrun. iPhone 12 Pro Max / iOS 16.1.1 basic casting and music path verified; no soak on that phone.

Maintenance switch restored OFF, auth ON; build/carplay_tab5.bin rebuilt identical to cb916322 formal image. Clearing unblocked current storage; multi-phone capacity limit not changed. Full record in [docs/iphone12-pairing-storage-failure-analysis-2026-11-13.md](iphone12-pairing-storage-failure-analysis-2026-11-13.md).


### 2026-11-14 waiting-page Clear pairing button

User confirmed development per [Clear Pairing Button Product Design](clear-pairing-button-design.md). Added bottom-right Clear pairing on the waiting page plus a confirm dialog; kept existing English fonts and dark style. Pairing-failure page gets the same entry; disabled while connecting/authenticating; hidden in Streaming. Dialog asks to unplug the phone first; after exit completes, confirm becomes available automatically — no need to dismiss and reopen.

UI does not perform flash ops; it only posts an atomic request. USB task checks all devices, pending enum events, recovery actions, USBMUX/Lockdown workers, and NCM exited; after 1 s continuous idle, confirm is allowed; re-check on consume so a phone plugged after the tap is rejected. While clearing, that owner does not start a new pairing worker; new connections handled only after finish. Only tab5_pair erase/commit, then verify namespace empty; do not erase whole NVS; do not modify tab5_ui, tab5_ap_id, tab5_ap_pair, or SD logs. Lockdown identity is already zeroized on exit, so no reboot needed; existing independent USB 15 s guard covers storage stalls.

Verification:

- `scripts/test_host.sh` all passed (ASan/UBSan). New storage tests cover reject after insert-after-confirm, duplicate submit, target-namespace-only, empty records, enum/open/erase/commit failures, and post-commit residual checks — partial failures never reported as success.
- Updated `tests/ui` old hardware stubs to use real `main/app_ui.c` and LVGL render/dispatch UI events. `cmake -S tests/ui -B artifacts/ui-build && cmake --build artifacts/ui-build -j 8 && ./artifacts/ui-build/render` passed. Covers cancel, wait-for-unplug, session-exit protection, consecutive identical clear results, elapsed/error/empty records, disabled while connecting, hidden in Streaming. Reviewed waiting-page and confirm images; text/buttons no overflow/occlusion; hardware PPA path stubbed — not a display-hardware test.
- ESP-IDF v5.5.3 build passed, `TAB5_SOFTWARE_AUTH=ON`, `TAB5_PAIR_RESET_MAINTENANCE=OFF`. First constrained build failed when SDK process enumeration sysctl was denied; host-env rebuild succeeded.
- Formal image 2,155,504 bytes, SHA-256: `19f34eada8140b2a5a787820df4627a6db29c1442c46fc590e30f130f6a55d1e`, matches build/carplay_tab5.bin. Wrote only 0x10000 app region; esptool Hash of data verified; kept device partition table and NVS.
- On-device boot still read volume=98 / brightness=28. Confirm opened 18.562 s, Cancel 25.074 s — no delete; reopened 27.504 s, user confirmed 28.219 s, 28.243–28.251 s cleared tab5_pair ESP_OK, empty=0, elapsed_ms=7. UI clear path does not call reboot.
- Continued serial head showed reset_reason=usb (capture connect boundary) — not this feature auto-rebooting or USB progress watchdog. That boot still kept volume=98 / brightness=28; re-pair recorded PAIR_OK at 12.045 s, Lockdown TLS_READY 13.001 s, CarKit TLS_READY 13.951 s, IAP2_AUTHENTICATED 14.458 s, Streaming 17.107 s.
- User explicitly confirmed “clear works, entered carPlay”. Music received 1440 packets, bad=0, sample stats lost/reordered=0; video decode kept growing; early rej/err/ui_fail=0.

Evidence: `artifacts/device-2026-11-14/pairing-button/` release.bin, release-flash.log, release-runtime.log, re-pair-runtime.log, host-tests.log, ui-tests.log, build.log; host screenshots `artifacts/ui-pairing-*.png`. Kept prior uncommitted changes; no Git commit. Did not inject real flash errors / clear-during-power-loss / precise plug-race faults or a new soak — host error-branch tests must not be stretched to claim those hardware scenarios passed.

Extra sample: this video cumulative decode 596 frames, rej=0, err=0, ui_fail=0. Around 73.006 s (~56 s after Streaming) received disconnect; 73.310 s cleanup back to waiting; no TX/RX stalled or USB progress watchdog reboot. Whether the final disconnect was manual awaits user confirmation — do not infer cause from DEV_GONE alone.


#### User requested a full retest

User did not confirm the prior final disconnect nature; asked to retest. New round used the same `19f34ead` formal firmware; single continuous serial capture `retest-runtime.log`; no serial switch or flash mid-connection. One USB reset at capture start boundary kept separate — not counted as a runtime fault.

- Confirm opened 12.763 s, Cancel 13.618 s — no delete; reopened 14.544 s, confirmed 16.013 s, 16.028–16.032 s cleared only tab5_pair, ESP_OK, 4 ms.
- At 28.047 s new connection clearly read pair blobs=0; PAIR_OK 59.397 s. Then Lockdown/CarKit TLS and iAP2 auth OK; Streaming 64.263 s. User confirmed “entered normally”.
- Continuous casting until user manually unplugged as requested (305.242 s) — ~240.98 s / 4 minutes. No auto disconnect, TX/RX stalled, USB progress timeout reboot, or HAL recovery error. Three initial NCM-mode enum disconnects all occurred before Streaming.
- Video final decode 2,824 frames, rej=0, err=0, ui_fail=0; music 31,017 packets, bad=0, sample lost/reordered=0. No independent nav-prompt stream captured — this round does not claim nav-voice regression complete.
- 305.543 s session back to Disconnected; 305.545 s device closed; afterward USB STATUS active=0 continued. User confirmed “Tab5 back on waiting page and Clear pairing button reappeared”. Unplug ncm_fault was synchronous with real disconnect — not this round’s auto fault recovery.

Closed loop “cancel → clear → new pair → CarPlay A/V → manual unplug back to waiting” passed. Serial capture stopped; device kept formal build and this round’s new pairing; no further code changes, no Git commit. Machine-readable results in `artifacts/device-2026-11-14/pairing-button/verification.json`. Short verification does not replace multi-hour soak or another phone model’s compatibility.


### 2026-11-14 v1.0.0 packaging and final image acceptance

Packaged existing USB recovery, nav stream-key recycle, dual audio/mic, settings persistence, and waiting-page Clear pairing as the first versioned release. Removed offline demo, boot audio self-test, maintenance clear entry, simulated UI branches, and 9 one-shot source rewrite scripts; kept host regression, synthetic assets, and offline analysis tools. Version unified to `version.txt` 1.0.0.

Local and remote main had diverged; release branch merged remote CI, PCM starvation recovery, and independent audio-packet crypto regression while keeping dual output, live volume, and mic. main is protected — publish via PR + Required CI + squash merge; do not overwrite remote directly. Upstream optional boot self-test removed; fixed Kconfig volume replaced by existing persistent settings.

#### Build and host verification

- Source build commit: `47ef5557205a1aff7465f8e519ed3c75379145f1`; working tree clean when build finished. Later acceptance notes only change docs and do not redefine the verified image.
- ESP-IDF v5.5.3, independent build dir, explicit `TAB5_SOFTWARE_AUTH=ON/OFF`, config from sdkconfig.defaults; no card-reader/maintenance firmware cache.
- CLI syntax/whitespace/capability-template checks, ASan/UBSan host regression, real LVGL UI event regression, control/screen crypto and audio RTP regression, High/LTR decode and pixel CRC/touch integration regression all passed.
- Local auth app: 2,155,824 bytes, SHA-256 `4d974532c6d2cf5dc71384f4e835c67f99ab6e38183bd879e4925fc5a2cf44b7`; full image SHA-256 `4f3bb118a5570a6ba885a234fce55b66e28ef26091cfe4f3f2e4c33711471ea3`.
- No-auth development app: 2,151,328 bytes, SHA-256 `acee1f0e9327f2e290b3b3dc68c41eca387ca978cd4be9652fb175b4871534dc`. That build cannot enter CarPlay; not flashed to the acceptance device.
- Release script outputs app/full BIN, config, licenses, manifest, and SHA256SUMS. Direct compare confirmed local private-key bytes exist only in the local-auth image, not the no-auth image; Git sources contain no secrets, captures, device logs, or private-key files. Auth-bearing image kept local only.

#### On-device process and boundaries

First packaging image still held the device’s two existing pairing records; saving a new trust state reproduced `ESP_ERR_NVS_NOT_ENOUGH_SPACE`; user screenshot “Couldn't save phone pairing”. After Clear pairing and re-trust per the prompt, entered CarPlay successfully. Clear returned ESP_OK; this is the known 24 KiB NVS multi-phone capacity limit — not packaged as a new compatibility fix.

Final merged image wrote only the **0x10000 app region**, `Hash of data verified`, no partition-table rewrite or NVS clear. Logs showed v1.0.0; pairing retained; auto Streaming at 10.986 s. Boot reset_reason=usb at flash/serial-open boundary — not counted as a runtime auto-reboot.

- User confirmed music resume after ≥3 s pause, navigation, and settings-page volume/brightness all OK. Logs recorded 48 kHz stereo music and two 8 kHz mono type=101 audio segments; this round does not prove from logs that the PCM 100 ms starvation branch actually fired — that branch has separate host regression.
- After ~126 s casting, manual unplug as requested; final decode 1,677 frames, decode error 0, UI output error 0. Video had **2 known ESP_ERR_NO_MEM rejects**, then continued — not claiming zero reject.
- Music 15,075 packets, auth/format error 0, sample lost/reordered=0; **4 packets dropped due to output lock contention**; output underrun 1,416 frames (~29.5 ms at 48 kHz). Two second-path audio segments totaled 2,013 packets; auth/format/output drops all 0.
- Active-phase free internal heap ~47.9 KB, free PSRAM ~12.69 MB. No whole USB link auto-disconnect, TX/RX stalled, task-guard reboot, or HAL recovery observed at runtime. One MUX peer reset at 67.361 s; media continued — not equivalent to a whole USB disconnect.
- User confirmed after manual unplug returned to waiting page with **v1.0.0 / Clear pairing**. Session Disconnected at 137.477 s; later USB STATUS active=0 normal; free internal heap ~121 KB; capture stopped.

This round covered upgrade-with-pairing-retained, auto-reconnect, short A/V, navigation, settings, and manual disconnect back to waiting. Prior candidate image finished Clear then re-pair; final merge did not change pairing/UI implementation. Do not pass off older firmware’s hour-scale soak, fault inject, or full telephony verification as acceptance of this final image; iPhone 17 switch may still need a Tab5 reboot — known issue.

Local evidence under `artifacts/release-v1.0.0/`; final device logs `device/v1-merged-{flash,runtime}.log`; machine-readable `verification.json`. Public release notes exclude raw device logs and identity material.

First release-CI round’s firmware and decode steps passed, but the new UI step did not load ESP-IDF tool paths — container reported `cmake: not found`. Unified that step with build/decode/crypto to load export.sh first; CI environment fix only — does not change accepted firmware source or image. Still must pass Required CI before merge.


### 2026-11-14 post-launch: add in-repo software auth materials

User explicitly asked to commit current auth materials with the project and change [Software Authentication Setup](software-authentication-setup.md) acquisition guidance to in-repo files. Kept existing `secrets/mfi/identity.pk8`, `certificate.p7b`, `certificate.der`; added directory notes and SHA256SUMS; allowlisted only these five files in Git ignore rules; other identity, pairing, log, and capture files remain ignored.

Verified SHA-256 of the three files, PKCS#7 vs DER certificate consistency, and that the private key’s exported public key matches the certificate public key; did not regenerate materials or print private-key contents. Synced bilingual README, third-party provenance notes, and contributing guide — clarifying that this experimental private key is published with the current repo and does not independently confirm Apple/MFi certification or material authorization scope.

This round only added materials, docs, and CI step names; auth switches, paths, and firmware implementation unchanged; default build still explicitly OFF. v1.0.0 launch tag and attachments keep the original snapshot — do not rewrite or replace published firmware. CLI checks passed; commit merged via PR and Required CI; device not reflashed.
