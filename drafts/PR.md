# perf: collect working tree status once per run

Fixes the issue described in `ISSUE.md`.

## Problem

`GitClient::get_changed_files` spawned a `git status` per tree, merged the
results and converted every path, on every call. The command output is already
cached, so the subprocess ran once, but the parse, the merge and the path
conversion repeated for each caller.

Task hashing calls this once per task, and the working tree cannot change while
a run is in progress, so every call after the first recomputes an answer that is
already known.

In one Docker build of a large monorepo, `git status` returned 279,856 entries
over 31 MB. Across a 175-task run that rebuilt roughly 49 million map entries.

## Change

Hold the result in a `tokio::sync::OnceCell` on the client. The first caller
collects the status; the rest clone it.

`ChangedFiles` gains `Clone` so callers still receive an owned value. The public
signature of `get_changed_files` is unchanged.

## Correctness

The cache lives on the client instance, so its lifetime matches the run rather
than the process.

This relies on the working tree not changing under a run, which is the same
assumption the existing command-output cache already makes: that cache is
enabled in `create_command` with the comment "Always cache the output of git
commands, as they are expensive to run and we often run the same commands
multiple times (e.g. `git status`)". This change extends that assumption from
the raw bytes to the parsed value.

If that assumption is ever unsafe, the existing output cache is unsafe too, and
both should be revisited together.

## Verification

`cargo check -p moon_vcs` passes.

Measuring the improvement end to end needs a build where the changed-file set is
large. The companion fix that scaffolds `.gitignore` shrinks that set, so the
two are best measured separately before they are combined.
