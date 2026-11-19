# Contributing

Create a feature branch and open a pull request to `main`. Direct pushes,
force pushes and deletion of `main` are blocked, including for administrators.
Resolve review conversations, update the branch with current `main`, and pass
`Required CI` before merging. Use squash merge; merged feature branches are
automatically deleted.

The repository currently has one maintainer, so a second person's approval is
not mandatory. Add required approvals when another reviewer is available.

## Local CLI checks

```sh
./scripts/check_cli.sh
./scripts/test_host.sh
# Optional: check all commits against the PR base
BASE_SHA=origin/main ./scripts/check_cli.sh
```

CI validates Python/shell syntax, changed-line whitespace and Actions workflows,
then runs the host suite with warnings treated as errors and ASan/UBSan. A
separate ESP-IDF v5.5.3 job fetches verified OpenH264 sources, builds ESP32-P4
firmware, and runs decoder and crypto regressions. All jobs run on every PR,
including documentation-only changes; failed, cancelled or skipped jobs fail
the required aggregate gate. Workflows use read-only repository permissions
and receive the explicitly bundled public experimental identity with the checkout,
but build firmware with software authentication disabled. Do not add other private
identity files, device pairing records or secrets.

Firmware build instructions are in README.md. Passing CI does not establish
USB, CarPlay, audio/video or touchscreen behavior on real hardware. Record
hardware results and remaining limitations in the PR.
