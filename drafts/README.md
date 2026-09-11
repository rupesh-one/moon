# Drafts

Working copies of upstream text. Nothing here has been sent to `moonrepo/moon`.

| File | What it covers |
| --- | --- |
| `PR-hash-path-instrumentation.md` | Pull request text for the tracing spans on the branch `experiment/hash-path-instrumentation` |
| `ISSUE-changed-files-reparsed.md` | `get_changed_files` rebuilds the same map once per task |
| `ISSUE-scaffold-missing-gitignore.md` | `moon docker scaffold` omits `.gitignore`, so git reports installed dependencies as untracked |

The two issues are independent. Either fix helps on its own, and together they
compound: the first stops the repeated work, and the second shrinks the input
that work runs over.

## Where the measurements come from

A Docker build of one service in a large pnpm monorepo, on Moon 2.5.2,
linux/arm64, 4 CPU, inside `node:22-bookworm-slim`. The build runs
`moon docker scaffold`, then `moon docker setup`, then
`moon run <project>:build`. All 175 tasks in the run are remote cache hits, so
no build command executes and the measured time is almost entirely hashing.

Span figures come from `moon run --dump`, aggregated by module and source
location. Two properties of that data are easy to misread:

- `#[instrument]` on an `async fn` enters the span once per poll, so `n` is a
  poll count rather than a call count, and per-call averages are meaningless.
- Tokio moves child futures onto other worker threads. A child on a different
  thread does not nest inside its parent, so a parent's self time absorbs work
  its children performed. Treat self time as an upper bound.
