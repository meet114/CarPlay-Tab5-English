# iPhone 12 Pro Max / iOS 16.1.1 Did Not Enter CarPlay: Pairing State Save Failed

## Conclusion

The immediate blocker is confirmed: after the user confirmed trust on the phone, Tab5 failed to update the local pairing record and returned `ESP_ERR_NVS_NOT_ENOUGH_SPACE`, after which the state machine entered Failed / CP_ERR_STORAGE. This alone does not prove iOS 16.1.1 is incompatible; TLS, iAP2, and media were never reached, so compatibility also cannot be declared as passed yet.

The current firmware already has logic to store multiple records by phone identity, but the existing 24 KiB NVS and whole-blob update approach limit real phone-switching capacity. The earlier reply that the device “could switch directly to another iPhone” omitted this constraint.

## On-device evidence

Serial evidence: `artifacts/device-2026-11-13/iphone12-pairing/followup-runtime.log`.

- Line 14: the actual device NVS partition is `0x9000 / 0x6000`, i.e. 24 KiB.
- Around line 442: the NCM interface is ready; current phone IN=88 / OUT=06, showing descriptor auto-discovery identified a different receive endpoint than the previous phone.
- Line 452: `LOCKDOWN_QUERY_TYPE_OK`; basic USBMUX/Lockdown queries succeeded.
- Lines 457–458: NVS entries used=486, free=270, total=756; pairing blobs=2, totaling 13536 bytes, 6768 bytes each.
- Line 467: `PAIR_RECORD_REUSED phone key matched; trusted=pending`. An identity record for the current phone already exists and is still in a locally unconfirmed trust state.
- Line 469: waiting for phone trust; the user then explicitly confirmed they had tapped Trust.
- Line 477: `Pair persistence failed: ESP_ERR_NVS_NOT_ENOUGH_SPACE; existing NVS is not erased`.
- Line 480: `SESSION: stage=Failed error=5`, matching the on-screen “Pairing storage unavailable. Existing records preserved.”
- After failure, USB STATUS and NCM STATUS continued to print; the USB task did not hang.

## Why even a one-byte state change cannot be written

`main/lockdown_client.cpp:432` changes `trusted` from 0 to 1 after a successful Pair reply, then calls save. Save at line 109 uses `nvs_set_blob` to rewrite the entire 6768-byte PairRecord, which includes certificates, identity, and private key, even though only one trust-state byte actually changed this time.

ESP-IDF NVS does not overwrite the old blob in place; it writes a new version first, then reclaims the old one, and must keep spare pages for garbage collection. Therefore free=270 in the log cannot simply be multiplied by 32 and treated as contiguous space available for this update. What failed here is clearly storage allocation—not the phone refusing trust, and not ordinary runtime heap exhaustion.

Basis: this machine’s ESP-IDF v5.5.3 `nvs_storage.cpp` first calls writeMultiPageBlob to write the new version; `nvs_pagemanager.cpp` reserves free pages for reclamation. Official guidance likewise requires room for both old and new data during an update: [ESP-IDF File System and NVS Considerations](https://github.com/espressif/esp-idf/blob/v5.5.3/docs/en/api-guides/file-system-considerations.rst).

The current failure path does not erase existing pairing, so unplugging and replugging will again read trusted=pending and may fail repeatedly. Re-confirming trust or rebooting does not change this space problem.

## Recommended fix

Prefer separating the mutable trust state from the large identity blob: keep the existing two identity records, and store trusted as an independent small integer key indexed by phone; on load, prefer the independent state and fall back to the value inside the old blob when that key is absent. Confirming trust and InvalidHostID/InvalidPairRecord invalidation must both update the same state path—do not fix only the confirm-trust call site.

The implementation must validate the full phone public-key digest to avoid truncated-key collisions or stale state associating to the wrong identity; identity regeneration must also keep state consistent. On write failure, still stop and preserve existing records; do not free space by “delete old blob then write new blob.”

This avoids a 6.8 KiB rewrite for a 1-byte state change in this case, but it does not mean 24 KiB NVS can hold unlimited new phones. Long-term multi-phone support still needs compact storage or dedicated pairing space, plus migration and failure fallback that preserve existing identities. The current NVS size cannot simply be enlarged, because phy_init and factory follow immediately after it.

This round completed diagnosis only: no business code was changed, nothing was flashed, and no pairing records were actively deleted or cleared. When the phone confirmed trust, the original firmware itself attempted to save state; that save failure is recorded above.


## Recovery after user-authorized cleanup

After the user explicitly allowed clearing old records and re-pairing, the existing `TAB5_PAIR_RESET_MAINTENANCE` maintenance mode was used to erase/commit only the `tab5_pair` handle; it returned `PAIR_RESET: namespace=tab5_pair result=ESP_OK`, and the namespace was verified empty. The entire NVS was not wiped, and other namespaces were not modified.

The original release image `cb916322` was then flashed back; Hash verified. Boot stats showed NVS used=54/free=702, pair blobs=0. After the phone reconfirmed trust:

- 31.879 s: `PAIR_OK identity saved for this phone`.
- 32.824 s: Lockdown TLS_READY; 33.723 s: CarKit TLS_READY.
- 34.285 s: `IAP2_AUTHENTICATED phone accepted challenge response`.
- 47.428 s: `SESSION: stage=Streaming error=0`, then video decode counts kept rising.
- The user explicitly confirmed the CarPlay UI was displayed.
- The first music segment received 3361 packets, bad=0; during that period audio packet loss, reordering, underruns, and submit rejects were all 0.

This proves the pairing-storage fault was recovered by clearing old records, and that this iPhone 12 Pro Max / iOS 16.1.1 completed the basic CarPlay video and audio path. A long-term multi-phone storage design was not implemented; switching back to the previous phone requires re-pairing, and accumulating more records may still hit capacity limits.

The maintenance switch is already off (`TAB5_PAIR_RESET_MAINTENANCE=OFF`), software authentication remains ON, and the locally rebuilt `build/carplay_tab5.bin` SHA-256 matches the `cb916322` release image. The device was not left on a maintenance build that clears records on boot. Evidence: `reset-runtime.log`, `repaired-flash.log`, `repaired-runtime.log`, all under the local evidence directory above.
