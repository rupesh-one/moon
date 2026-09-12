# perf: optimize distribution builds for execution speed

Fixes the issue described in `ISSUE.md`.

## Problem

The distribution profile inherits `opt-level = "z"` from the release profile.
This setting reduces the published binary size, but it slows file-heavy
commands.

In a file-heavy `moon docker scaffold` workload, changing only `opt-level` from
`3` to `"z"` increased execution time from 77.753 seconds to 169.843 seconds.
The packaged GNU binary, which also enables fat LTO and aborting panics, took
301.320 seconds.

## Change

Set `opt-level = 3` in `[profile.dist]`:

```toml
[profile.dist]
inherits = "release"
opt-level = 3
```

## Compatibility

`cargo-dist` builds with the `dist` profile, so the change applies to published
Moon artifacts. The distribution profile continues to inherit fat LTO, one
codegen unit, aborting panics, and stripped debug information.

The release profile remains unchanged. Nix, local `--release`, and benchmark
builds keep their current optimization settings.

## Verification

- `cargo build --profile dist --locked --package moon_cli --bin moon`
- The same-source Linux ARM64 runs produced identical canonical scaffold
  digests with the speed and size profiles.
