# Task hashing rebuilds the same changed-file map once per task

## Summary

Moon caches the output of `git status`, but not the parse of it. Every task
hash calls `get_changed_files`, which re-tokenises the cached output, rebuilds a
`ChangedFiles` map, merges it, and converts every path. The answer cannot change
within a run, so all of that work after the first call is repeated.

The cost scales with the number of changed files times the number of tasks. It
is invisible in a profile, because the merge and the path conversion carry no
spans.

## Evidence

Moon 2.5.2, linux/arm64, 4 CPU. A Docker build of one service in a large pnpm
monorepo. 175 tasks in the run, all remote cache hits, so no build command
executes.

The `git status` that every task re-parses:

```
bytes=31141105        # 31.1 MB
entries=279856
index_files=49626      # tracked files in the index
```

The subprocess itself runs once and takes 4.2s. Spans from `moon run --dump`:

| Span | self | total | n (polls) |
| --- | --- | --- | --- |
| `task_runner::hash` | 1184.0s | 1184.3s | 2430 |
| `git::tree::exec_status` | 192.2s | 192.2s | 4948 |
| `sha256::from_file` | 0.2s | 1.0s | 11436 |
| `git::tree::exec_ls_files` | 0.2s | 0.2s | 929 |

File content hashing is 1.0s in total, so it is not the cost. `exec_status` is
192s, and it covers only the tokenise-and-map step. The merge and the path
conversion that follow it have no spans, and they account for a large part of
the ~890s of `hash` that nothing else explains.

## Cause

`create_command` in `crates/vcs/src/git/common.rs` caches the subprocess, with
the intent stated in a comment:

```rust
// Always cache the output of git commands, as they are expensive to run
// and we often run the same commands multiple times (e.g. `git status`).
command.set_cache(true);
```

`exec_capture_output` honours that and returns a cloned `Output` on a hit. The
parse, however, happens after the await in `GitTree::exec_status`, on every
call: `output_to_string` allocates a fresh `String` of the whole output, the
tokens are matched against `STATUS_PATTERN`, and a `PathBuf` is joined and
inserted per record.

`GitClient::get_changed_files` then adds no memoisation of its own:

```rust
for tree in self.get_all_trees() {
    set.spawn(async move { tree.exec_status().await });
}
while let Some(result) = set.join_next().await {
    changed_files.merge(result.into_diagnostic()??);
}
changed_files.into_workspace_relative(&self.workspace_root)
```

`merge` walks every entry, and `into_workspace_relative` walks them again,
calling `relative_to` per entry. `TaskHasher::aggregate_inputs` calls this once
per task.

So with 279,856 entries and 175 tasks, Moon builds roughly 49 million map
entries to answer the same question 175 times.

## Impact

Worst in Docker builds, where the working tree and the git index disagree by
construction, but the shape applies to any repository with a large changed-file
set and many tasks in one run.

## Suggested fix

Memoise the result for the lifetime of the run. `get_changed_files` is the
natural place: its inputs do not change while a run is in progress. Caching the
parsed `ChangedFiles` rather than only the raw `Output` would also remove the
repeated allocation of the 31 MB string.

If a per-run cache is unattractive, caching inside `exec_status` keyed on the
command cache key would remove most of the repetition, though it would leave
`merge` and `into_workspace_relative` running per task.

## Related

`moon docker scaffold` omits `.gitignore` from the configs skeleton, which is
what makes the entry count so large in a Docker build. That is filed separately.
Fixing it shrinks the input to this problem but does not remove the repetition.
