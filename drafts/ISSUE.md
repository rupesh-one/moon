# [feature] Deduplicate dependent expansion across sharded jobs

## Problem

`moon ci --job <index> --job-total <total>` partitions affected primary targets before Moon expands their dependencies and dependents. Two shards can therefore run the same dependent when one shard starts that target as a primary and another shard reaches it through downstream expansion.

Issue #2147 described duplicate work in the previous CI implementation. That issue is closed, but the `job` and `job_total` path can produce a related result in the current action graph.

A first experiment used a hash owner that did not match the contiguous primary slice. Remainder shards then expanded another shard's primaries as dependents. Ownership must use the same positional range as primary placement.

## Proposed experiment

Add a default-off `experiments.dedupeShardedDependents` experiment.

The experiment keeps the existing contiguous, index-based partition for primary starts. It treats that same slice as the owner set. A shard suppresses a primary only when another shard owns that primary and this shard reaches it through dependent expansion.

The experiment does not:

- change which primary targets a shard starts;
- suppress dependency expansion;
- suppress dependents that are included only through project relations;
- provide a global exactly-once guarantee;
- change partitioned execution-plan behavior.

Required dependencies can still run on more than one shard when cache state requires them.

## Configuration

```yaml
experiments:
  dedupeShardedDependents: true
```

The equivalent environment variable is:

```text
MOON_EXPERIMENT_DEDUPE_SHARDED_DEPENDENTS=true
```

## Acceptance criteria

- The experiment is disabled by default.
- Disabled behavior matches the current sharded action graph.
- For job totals from 1 through 6, the union of enabled shard targets matches the unsharded target set.
- Each enabled shard target set is a subset of the same shard with the experiment disabled.
- At least one shard removes duplicate dependent work in a graph that has overlapping downstream expansion.
- Dependent suppression uses the same contiguous slice as primary placement, including remainder shards.
- A one-job run is unchanged.
- Over-partitioning does not remove target coverage.
- Deep dependent replay remains compatible with #2613.
- Partitioned execution plans produce the same graph with the experiment enabled or disabled.
- Required non-cacheable and cache-miss dependencies can still execute on shards that do not own them as primaries.

## Related work

- #2147 reported duplicate CI work in the previous implementation.
- #2613 restored dependent expansion when a task is revisited with dependents in scope.
- #2630 proposes pre-execution output for the resolved CI action graph. That interface is outside this experiment.
