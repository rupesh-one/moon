# perf: add tracing spans to the task hash path

## Problem

`TaskRunner::hash` carries `#[instrument]`, but none of the functions it calls
do. A `--dump` trace therefore attributes almost the whole hash to one span and
cannot say where the time goes.

Profiling a Docker build of a single service in a large monorepo put 1184s of a
1184s `hash` span into that span's own self time, while every instrumented child
summed to roughly 290s. The remainder belonged to no span at all. Finding the
cause needed a patched binary, because the shipped one cannot answer the
question.

## Change

Add `#[instrument(skip_all)]` to the functions `hash` reaches:

| Crate | Function | What it isolates |
| --- | --- | --- |
| `moon_task_hasher` | `hash_common_task_contents` | Inputs, dependency hashes, args and env |
| `moon_task_hasher` | `hash_toolchain_task_contents` | The per-toolchain half of the hash |
| `moon_task_hasher` | `apply_toolchain` | One toolchain's contribution |
| `moon_task_hasher` | `apply_toolchain_dependencies` | Manifest and lockfile parsing |
| `moon_task_hasher` | `hash_inputs` | Input discovery and hashing overall |
| `moon_task_hasher` | `aggregate_inputs` | Glob and VCS file discovery |
| `moon_task_hasher` | `process_inputs` | Filtering discovered files |
| `moon_vcs` | `GitClient::get_changed_files` | Status collection across git trees |
| `moon_vcs` | `ChangedFiles::into_workspace_relative` | Path conversion of every entry |

## Notes

`skip_all` keeps arguments out of every span, so the change adds no `Debug`
bounds and records no paths or file contents.

`tracing` is already a dependency of both crates, so `Cargo.toml` is unchanged.

There is no behaviour change. The spans are inert unless a trace is being
recorded.

## Why these functions

They are the direct children of `hash`, plus the two places that walk the
changed-file set. That is enough to split the hash into parts on the first
profile rather than after several rounds of guessing. `hash_checks` and
`HashEngine::save_manifest` already carry spans, so they are left alone.

## Verification

`cargo check -p moon_task_hasher -p moon_vcs` passes.
