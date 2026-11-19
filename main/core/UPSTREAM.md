# Provenance and local boundaries

`ntb16.h` and `airplay_rtsp_codec.c/h` come from `xmz28/Carplay-S3` commit
`79a9f169f275b283f2909c5a326ed03c716c0d02`, mapping to `src/ntb16.h` and
`components/airplay_rtsp/`. GPL-3.0-or-later and original copyright notices are
retained. `tests/test_ntb16.cpp` is from the same source (with a main-link
declaration tweak for macOS Clang). License text is at
`licenses/Carplay-S3-GPL-3.0.txt` in the repository root.

`session.c/h`, `rtsp_stream.c/h` and `touch.c/h` are new modules in this
project. Protocol core and physical transport adapters are validated
separately; a passing unit test does not mean NCM is up or that the phone has
finished an RTSP session. Request/body views in RTSP callbacks are short-lived;
callers must process or copy them before returning. All core state is owned by
a single task.
