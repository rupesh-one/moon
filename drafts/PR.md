## Summary

Add a default-off experiment that reduces duplicate dependent expansion across `job` and `job_total` shards.

The change preserves the existing index-based primary partition. It uses a deterministic hash owner only when a shard reaches another primary through downstream expansion.

## Behavior

- Keep primary starts on their current index-based shards.
- Keep dependency expansion unchanged.
- Suppress only directly selected primaries that another shard owns by hash.
- Keep relation-only dependents eligible on every shard that reaches them.
- Keep the suppression state scoped to one `run_tasks` call.
- Preserve deep dependent replay when an existing node is revisited in scope.

The hash coordinates shards from one Moon binary. It is not a cross-version target-assignment contract.

The experiment does not provide global exactly-once execution. A target can still run on its index shard and on its hash-owner shard.

`run_tasks_with_plan` clears `job` and `job_total` after it selects a partitioned execution-plan job. The experiment is therefore inert for partitioned execution plans.

## Configuration

```yaml
experiments:
  dedupeShardedDependents: true
```

```text
MOON_EXPERIMENT_DEDUPE_SHARDED_DEPENDENTS=true
```

## Verification

- Confirmed that disabled behavior preserves duplicate dependent expansion.
- Confirmed that enabled shard target sets remain subsets of disabled shard target sets.
- Confirmed that the union of enabled shard targets matches the unsharded graph for job totals from 1 through 6.
- Confirmed one-job behavior, relation-only dependents, `runInCI` filtering, and call-scoped state.
- Confirmed that partitioned execution plans are identical with the experiment enabled and disabled.
- Ran focused action-graph regression tests.
- Ran Rust formatting.
- Ran Clippy for `moon_action_graph` and `moon_config` with warnings denied.

## Related work

- #2147
- #2613
- #2630
