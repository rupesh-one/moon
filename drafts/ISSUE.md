# [feature] Deduplicate dependent expansion across sharded jobs

## Problem

`moon ci --job <index> --job-total <total>` partitions affected primary targets before Moon expands their dependencies and dependents. Two shards can therefore run the same dependent when one shard starts that target as a primary and another shard reaches it through downstream expansion.

Issue #2147 described duplicate work in the previous CI implementation. That issue is closed, but the `job` and `job_total` path can produce a related result in the current action graph.

## Proposed experiment

Add a default-off `experiments.dedupeShardedDependents` experiment.

The experiment would keep the existing contiguous, index-based partition for primary starts. It would assign each primary target a deterministic hash owner and suppress that target only when another shard reaches it through dependent expansion.

The experiment would not:

- change which primary targets a shard starts;
- suppress dependency expansion;
- suppress dependents that are included only through project relations;
- provide a global exactly-once guarantee;
- change partitioned execution-plan behavior.

The hash owner would coordinate shards from the same Moon binary. Target ownership would not be stable across Moon or hash-library versions.

## Configuration

```yaml
experiments:
  dedupeShardedDependents: true
```

The equivalent environment variable would be:

```text
MOON_EXPERIMENT_DEDUPE_SHARDED_DEPENDENTS=true
```

## Acceptance criteria

- The experiment is disabled by default.
- Disabled behavior matches the current sharded action graph.
- For job totals from 1 through 6, the union of enabled shard targets matches the unsharded target set.
- Each enabled shard target set is a subset of the same shard with the experiment disabled.
- At least one shard removes duplicate dependent work in a graph that has overlapping downstream expansion.
- A one-job run is unchanged.
- Over-partitioning does not remove target coverage.
- Deep dependent replay remains compatible with #2613.
- Partitioned execution plans produce the same graph with the experiment enabled or disabled.

## Related work

- #2147 reported duplicate CI work in the previous implementation.
- #2613 restored dependent expansion when a task is revisited with dependents in scope.
- #2630 proposes pre-execution output for the resolved CI action graph. That interface is outside this experiment.
