# Wired CarPlay Optimization Design

Date: 2026-11-19  
Status: approved in conversation; awaiting written-spec confirmation  
Scope: wired path only — no wireless / SoftAP / mDNS / RF work

## Goal

Improve wired CarPlay on M5Stack Tab5 across three known pain points without rewriting the USB→iAP2→NCM→AirPlay stack:

1. **Video smoothness** — fewer opaque rejects; clearer recovery when the decode queue falls behind
2. **USB stability diagnostics** — enough fault detail to distinguish phone detach vs host/controller error on the next soak
3. **Audio glitch reduction** — slightly more buffer headroom against brief underruns during music/nav mix

Success is measured by host regressions staying green and by richer on-device stats that make the next soak actionable. This design does **not** promise zero disconnects, higher default resolution/FPS, or wireless CarPlay.

## Context and constraints

- Current product path: USB Host → USBMUX/Lockdown → iAP2 → NCM IPv6 → AirPlay media (`docs/technical-implementation.md`).
- Soft-decode budget already limits sharpness/FPS; default remains in the proven wired profile family (README: 890×240 targeting 16 fps).
- Soak analysis (`docs/sd-log-analysis-2026-11-13-soak.md`) showed: 9 video rejects logged as `ESP_ERR_NO_MEM` that were actually queue rejects; brief audio underruns (~15 ms); one unexplained USB disconnect with recovery; internal heap ~47–48 KB during music.
- Internal heap is tight; large buffer increases are out of scope.
- ESP32-C6 Wi‑Fi/BLE is intentionally unused for this work.

## Approach (selected)

**Surgical wired pass** — small, testable changes in existing modules. Rejected alternatives:

| Approach | Why not |
|---|---|
| Observability-only | Delays user-visible audio/video tuning; still need a first code pass |
| Aggressive retune | Bigger queues/preroll risk heap pressure and worse disconnects |

## Design by area

### 1. Video

**Problem:** `tab5_video_submit` / `media_routes` map any push failure to `ESP_ERR_NO_MEM`, hiding full-queue vs waiting-for-IDR vs oversized AU. Overflow clears the queue and forces IDR recovery without structured counters for soak analysis.

**Changes:**

- Extend `cp_video_queue` (or submit path) to report a reject **reason enum**: e.g. `ok`, `wrong_generation`, `oversized`, `waiting_idr`, `overflow_reset`.
- Log/count reasons in `VIDEO` stats (alongside existing `rej=`).
- Map `media_routes` failures to a non-misleading error where practical (prefer reason-tagged log over pretending OOM).
- Optional modest increase of `CP_VIDEO_MAX_UNITS` by 1–2 **only if** host tests pass and SPIRAM-backed storage stays at 512 KiB (no larger byte capacity unless tests prove need).
- Keep existing profile-too-slow demotion and IDR request behavior.

**Out of scope:** raising default resolution/FPS beyond the current proven profile; replacing OpenH264; dual-stream decode.

### 2. USB

**Problem:** Soak could not tell phone-side reset vs contact/power glitch vs host-controller fault. Recovery already exists (`ncm_recovery`, power-cycle, HAL wrap for missing CHHLTD); policy changes without better logs are speculative.

**Changes:**

- On NCM TX/RX stall, NCM fault, and recovery start/skip, log structured fields when available: `usb_hal_recovered`, channel/endpoint if known, stall age, whether device is still attached, and whether power-cycle was skipped because the phone already detached.
- Prefer extending existing `ESP_LOG*` / `tab5_log_event` sites in `ncm_host.cpp` / `usb_probe.c` over new subsystems.
- Do **not** change power-cycle thresholds or invent root-port interrupt decoding that the current IDF hooks do not already expose cleanly.

**Out of scope:** rewriting `usb_hal_fix.c` semantics; claiming zero disconnects; wireless RF init.

### 3. Audio

**Problem:** Music + navigation soak recorded brief underruns (~14–15 ms) that may be audible. Preroll is 60 ms (`PREROLL_MS` in `audio_output.c`); clock-follow already speeds/slows when the jitter buffer is high/low.

**Changes:**

- Raise preroll modestly (target **80–100 ms**, start at **80 ms**) while keeping `LANE_MAX_FRAMES` at 0.5 s.
- Keep ducking and 10 ms mix loop unchanged.
- Ensure stop/stats lines remain comparable across firmware versions (same field names; underrun_frames still meaningful).
- Add or extend host coverage in existing PCM buffer tests for the new preroll assumption if behavior depends on it.

**Out of scope:** AEC rewrite; full phone-call path; changing I2S driver; large preroll (≥150 ms) that increases latency without soak proof.

### 4. Verification

| Gate | Required |
|---|---|
| `./scripts/check_cli.sh` | Pass |
| `./scripts/test_host.sh` | Pass (video queue + PCM + related) |
| `./scripts/test_ui.sh` / crypto / decoder as touched | Pass if scripts remain applicable |
| Hardware soak | Document as follow-up for the user; not a merge blocker for this docs/code PR if CI host tests pass |

## File touch map (expected)

| File | Responsibility |
|---|---|
| `main/core/video_queue.h` / `.c` | Reject reasons; optional max-units tweak |
| `main/video_output.cpp` / `main/media_routes.cpp` | Stats + truthful failure reporting |
| `tests/` video queue tests (existing or small extension) | Reason + overflow behavior |
| `main/ncm_host.cpp` / `main/usb_probe.c` | Structured stall/recovery logs |
| `main/audio_output.c` | Preroll bump |
| `tests/` PCM tests | Preroll/underrun expectations if needed |
| `docs/execution-log.md` or short note under `docs/` | Record intent after implementation (optional, keep brief) |

## Non-goals

- Wireless CarPlay, SoftAP, mDNS, Bluetooth, or `net_port` Wi‑Fi paths
- “Optimize everything” beyond the three areas above
- Flash encryption / NVS encryption / auth material changes
- Changing OEM icon label `Tab5 设置`

## Risks

- Extra logging on USB hot paths must stay rate-limited (stall warnings already periodic).
- Preroll increase adds a few tens of ms start latency; acceptable vs glitch reduction.
- Mis-attributing rejects if reason enum is incomplete — prefer explicit `unknown` over wrong OOM.

## Acceptance criteria

1. Video reject path exposes a concrete reason in stats/logs; `ESP_ERR_NO_MEM` is not the only signal for queue pressure.
2. USB stall/recovery logs include enough fields to classify attach-state vs recovered HAL errors on the next soak.
3. Audio preroll is 80–100 ms; host PCM tests pass.
4. No wireless code paths introduced.
5. CI host/CLI gates pass.
