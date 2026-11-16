# Offline Development and Focused Integration Plan

Updated: 2026-11-06. Per the user's adjustment, advance device-independent development first, then concentrate debugging and adaptation. While on-device work is paused: do not flash, do not start serial monitors, and do not initiate phone pairing.

## Current baseline

USB config scanning, USBMUX, Lockdown pairing/TLS, CarKit/iAP2 diagnostics, and host tests already exist. Pairing and TLS with the first phone were verified on device; phone-switch isolation logic has been added; pairing with the second phone (iPhone 12 Pro Max / iOS 16.1.1) is not yet complete. CarPlay receive, A/V, and interaction are not yet end-to-end.

## Development order

1. Session management: unify connection generation, phase, error, timeout, and exit logic; validate disconnect, reconnect, and late callbacks with a virtual clock and simulated transport. Distinguish pairing, CarPlay authentication, and media-ready states.
2. Transport and protocol: standalone NTB16 boundary parsing, incremental RTSP parsing, and bounded message queues; construct normal, truncated, oversized, cyclic-reference, and multi-packet inputs. Then wire the device transport adapter layer.
3. Video path: complete access-unit queue, backpressure, and IDR request interfaces; decoder wrapper, color conversion, and frame ownership; validate with legal local test bitstreams—do not claim iPhone video has been received.
4. Interaction and audio: letterbox coordinate mapping; press/drag/release state; PCM ring buffer, reorder/underrun policy, and format checks. Screen, touch, and codec calls stay concentrated in the board adapter layer.
5. Application UI: show connection phase, trust wait, missing authentication, error, and retry states; provide a clearly labeled simulation mode; simulated success must not change real authentication state.
6. Integration: firmware build, host tests, memory budget, and simulated end-to-end regression; produce a pending on-device verification checklist, then resume focused integration.

Each step delivers the corresponding implementation, automated verification, and remaining unverified items—do not treat interface stubs as finished features.

## Focused on-device verification

After restoring on-device work, proceed in order: independent pairing of the second phone and reuse of the old identity → CarKit/iAP2 authentication and negotiation → NCM/IPv6 → real media receive → decode/display performance → audio and touch → plug/unplug and long-running soak.

The authentication provider can define interfaces and a simulated implementation first; real authentication still needs valid materials and phone responses. USB timing, iOS differences, on-board decode frame rate, DMA, and audio clock cannot be replaced by host tests. Existing board-level verification is retained; later implementation may still change based on real negotiation results.
