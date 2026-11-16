# Waiting-to-Connect Page: Clear Pairing Button

Date: 2026-11-14. Status: implemented, flashed, and confirmed by the user on device (2026-11-14); the text below preserves the design requirements; actual results are in the final section.

## Goals and Scope

Let the user clear the local trust records of a previous phone directly on Tab5, free pairing space, and then connect another iPhone—without a computer or temporary maintenance firmware.

This feature addresses manual phone switching. It does not promise larger pairing capacity, and it does not provide a multi-phone list, automatic deletion of old records, or factory reset. Design is based on the existing `main/app_ui.c`, `main/app_main.c`, and [iPhone12 Pairing Storage Failure Analysis 2026-11-13](iphone12-pairing-storage-failure-analysis-2026-11-13.md).

## Waiting Page Layout

Continue the existing brand, centered connection prompt, and bottom mode info. Baseline is 1280 × 720 landscape:

- Top: Tab5 / CarPlay, keep the existing mint green.
- Center title: Connect your iPhone.
- Description: Connect an unlocked iPhone to the USB-A port.
- Bottom left: Device mode; bottom right: outlined secondary button "Clear pairing". Margin 48 px; button suggested 224 × 64 px, corner radius 8 px, text 24 px.
- Everyday waiting page does not use a prominent red button; only the final confirmation uses the destructive action style.
- Keep the current dark background #101820, body #e8eef3, secondary text #aab8c5, accent #70c6ad. Errors continue to use #ffb6a3.

Preview copy uses Chinese product wording. The current firmware is an English UI; first implementation may uniformly adopt the English mapping below and reuse the existing font. If Chinese is used, corresponding glyphs must be added and firmware size plus on-device display checked to avoid mixed Chinese/English layout.

## Interaction Flow

1. User taps "Clear pairing"; open a confirmation dialog—do not delete immediately.
2. If a phone is still connected on USB-A, the dialog says "Unplug your iPhone from USB-A first." and the confirm button is disabled; after unplug it becomes confirmable automatically. No need to reopen the dialog.
3. Confirm dialog title: "Clear phone pairing?". Body: "Remove all phone trust records saved on Tab5. Your iPhone will need to trust Tab5 again. Volume and brightness settings are kept." Buttons: left "Cancel", right "Clear pairing". Cancel has default focus.
4. After confirm, show "Clearing pairing…", block duplicate submit, and do not show a fake progress percentage. Do not connect a phone during this time.
5. Only after clear, commit, and verification succeed, show "Pairing cleared" with guidance "Connect an unlocked iPhone and confirm Trust." Enter a connectable state directly—no extra "Done" tap. The next connection event replaces the success message.
6. If there are no pairing records, show "No saved phone pairing"; the phone can still be connected directly.

Target experience: return to waiting-to-connect without reboot. If implementation cannot reliably clear old identity from memory, automatically reboot after a successful clear and change the progress/success copy to "Pairing cleared. Restarting…"; do not keep the old identity while claiming a successful clear.

## Entry and State Rules

| State | Entry and behavior |
| --- | --- |
| Disconnected, waiting to connect | Shown, tappable; do not auto-open the clear dialog |
| Waiting for phone trust | Shown, dialog can open; must unplug phone before confirm |
| Connecting, authenticating, network/media starting | Entry grayed out; prompt "Connecting—unplug the phone before clearing" |
| Pairing storage failure | Show button in the same place; main copy becomes "Couldn't save phone pairing", description "Clear old pairing, then reconnect". Do not describe every storage error as capacity full |
| Other failures, retry wait | Show entry; still must unplug phone and finish session exit before confirm |
| Session exiting | Keep confirm disabled; prompt "Ending connection…"; do not allow early proceed just because the cable is already unplugged |
| CarPlay is showing | Hide entry; do not cover navigation, music, or touch areas |
| Clearing in progress | Block another confirm; defer new phone pairing; touch must not go to an old CarPlay session |

If the user stays on the confirm dialog and a phone is reinserted, disable confirm immediately. If the session has already entered CarPlay, close any unexecuted confirm dialog and do not delete.

## Error Feedback

- Clear failed: "Couldn't clear pairing. Please try again." Offer "Back" and "Retry". Do not promise records are intact on failure, and do not show success. Retry still requires the disconnect condition.
- Session exit failed or timed out: prompt "Connection has not ended. Restart Tab5 and try again"; do not start deletion.
- Clear takes longer than 5 seconds: show "Clearing is taking longer than usual. Please wait"; this is a hint threshold, not a timeout that allows a second clear task. Failure or hang must converge via worker-task timeout / existing watchdog; do not block forever on the UI thread.
- Power loss during clear: on next boot, check actual storage state and enter waiting or a clear error; do not persist a maintenance flag that clears again on every boot.
- New phone inserted too early: re-check connection state before clear executes; if already connected, refuse deletion. During clear, serialize storage work and handle new USB connections only after completion—never concurrent with pairing writes.
- Phone never shows the trust prompt: first prompt to keep unlocked and reinsert; do not default to asking the user to reset all phone settings.

## Data Boundaries and Implementation Constraints

- Delete only records in the `tab5_pair` namespace; do not erase the whole partition.
- Keep `tab5_ui` volume/brightness, `tab5_ap_id` receiver identity, `tab5_ap_pair` records, and SD logs. Product copy uses "phone trust records" and must not imply clearing all protocol identities.
- UI only submits the request; a background task runs after USB is disconnected, the session has exited, and pairing writes have stopped. The clear entry is mutually exclusive with auto-connect/pairing writes; UI stays responsive.
- After a successful submit, re-read and verify the target namespace is empty, then clear in-memory caches and report success. No records is an idempotent success.
- Logs record request, cancel, reject reason, start, clear result, and duration; do not output certificates, private keys, or phone unique identifiers.
- Do not reuse the current "flash maintenance firmware then wait forever" interaction path; provide a one-shot action in the production firmware.

## English Copy Mapping

| Chinese | English |
| --- | --- |
| 清理配对（入口） | Clear pairing |
| 清除手机配对？ | Clear phone pairing? |
| 确认正文 | Remove all phone trust records saved on Tab5. Your iPhone will need to trust Tab5 again. Volume and brightness settings are kept. |
| 请先拔下 USB-A 接口上的手机 | Unplug your iPhone from USB-A first. |
| 取消 / 清除配对 | Cancel / Clear pairing |
| 正在清理配对… | Clearing pairing… |
| 配对已清除 | Pairing cleared |
| 成功引导 | Connect an unlocked iPhone and confirm Trust. |
| 没有需要清理的配对 | No saved phone pairing |
| 配对未能清除，请重试 | Couldn't clear pairing. Please try again. |

## Acceptance Criteria

1. When disconnected, clear can be completed from the waiting page in at most two taps; Cancel, tapping the overlay, or Back must not delete records.
2. While a phone is still connected or the session has not exited, deletion cannot run; after unplug, the dialog need not be reopened.
3. Clear affects only the designated namespace; volume, brightness, and other identities stay unchanged; logs remain readable.
4. Double-tap confirm, phone insert during clear, no records, and write failure all have definite outcomes—no deadlock or false success.
5. After clear, a new iPhone confirms trust and completes CarPlay video and music verification; switching back to the old phone can re-pair.
6. No button overlay during CarPlay; touch, navigation announcements, and music are unaffected.
7. Recoverable after power loss or abnormal exit; does not auto-clear on every boot.

## Implementation Results (2026-11-14)

- Production UI uses English, reusing the existing Montserrat font and dark layout. Waiting page bottom-right is `Clear pairing`; confirm dialog offers `Cancel` and `Clear pairing`. Confirm can run only after the connect/pairing thread has fully exited and been idle for 1 second.
- `main/pairing_reset.c` accepts a non-blocking request; the USB owner task re-validates after event handling and cleanup, then performs bounded-namespace erase/commit/empty-record verification. Reuses the independent USB 15-second progress watchdog; clear does not reboot the device.
- On failure, return to the result page and reconfirm retry through the same entry; with no records, show `No saved phone pairing`. New connection state takes priority over a stale clear result.
- Host ASan/UBSan regression and real LVGL UI tests passed. LVGL tests cover cancel, disabled while plugged/exiting, consecutive identical results, long-duration hint, failure, empty records, and hidden while Streaming; generated images were reviewed.
- Firmware `19f34ead` was written to the factory app region only and hash-verified. On device, Cancel did not trigger deletion; after confirm, `tab5_pair` clear and verification succeeded in 7 ms. User confirmed "Clear works; entered CarPlay"; serial log confirmed re-saved pairing, TLS, iAP2, and Streaming success.
- This is phone-switch functionality plus short video/audio verification; power-interrupt, real flash-error, and plug/unplug-during-clear fault injection were not run on device. Storage-failure / concurrent-reject branches were verified by host tests and are not equivalent to full hardware-fault acceptance.

Records and evidence: `docs/execution-log.md`, `artifacts/device-2026-11-14/pairing-button/`.

Follow-up retest: the same production firmware completed a single continuous capture. After cancel then reconfirm, clear took 4 ms; pairing record count was 0, then re-pair succeeded; continuous casting ~4 minutes, video 2,824 frames with no errors, music 31,017 packets with no bad/sample drops or reordering. After the user manually unplugged as requested, the waiting page and Clear pairing button were confirmed restored. This closed loop passed; details are in the execution log under "User requested full re-verification".
