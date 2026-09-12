# Distribution profile slows file-heavy CLI commands

**Describe the bug**

Moon's distribution profile inherits the release profile's size-first
optimization:

```toml
[profile.dist]
inherits = "release"
```

This gives published binaries `opt-level = "z"`. File-heavy commands run
substantially slower than the same source built with `opt-level = 3`.

**Steps to reproduce**

1. Build Moon with the distribution profile.
2. Build the same commit with `CARGO_PROFILE_DIST_OPT_LEVEL=3`.
3. Run `moon docker scaffold` for a service in a large monorepo with each binary.
4. Compare the elapsed time and the generated output.

**Expected behavior**

Published Moon binaries should use `opt-level = 3` for execution speed. Local
release, benchmark, and Nix builds should keep their current profiles.

**Environment**

```text
Moon source: v2.5.2 plus timing-only instrumentation
Target: aarch64-unknown-linux-gnu
Rust: 1.97.0
Container: node:22-bookworm-slim
CPU limit: 4
Command: moon docker scaffold rewards-campaign-svc dev-scripts
```

**Additional context**

A controlled build from the same source isolated `opt-level`:

| Profile | Scaffold time |
| --- | ---: |
| `opt-level = 3`, no LTO | 77.753 seconds |
| `opt-level = "z"`, no LTO | 169.843 seconds |

The packaged GNU binary, which uses `opt-level = "z"`, fat LTO, and aborting
panics, took 301.320 seconds. All runs produced the same canonical scaffold
digest.

Adding `opt-level = 3` to `[profile.dist]` overrides only the setting isolated by
this comparison. The distribution profile retains fat LTO, one codegen unit,
aborting panics, and stripped debug information.
