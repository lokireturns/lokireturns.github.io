---
layout: post
title: "How Delta Format Achieves Isolation"
date: 2025-11-21
categories: Delta-Lake
---

There is a very heavy penalty to pay in Delta Lake when transactions conflict. This note explores how the delta format achieves isolation between transactions.

In the best case scenario we enjoy lock free concurrency, in the worst case we waste compute and storage. That is the consequence of how concurrency is achieved with the Delta format - Optimistic Concurrency Control (OCC). 

At a high level OCC consists of 5 stages:

1. Read phase (no locking)
2. Modification phase (isolated work)
3. Write `parquet` files (copy-on-write)
4. Commit phase (serialisation point - only one wins)
5. Conflict resolution (reconcile or retry)

We should keep in mind that writing a single file to object storage is atomic - the file either exists or it doesn't, never partially. This property, guaranteed by S3/Azure/GCS, is what makes the commit phase work.

Let's imagine two simultaneous transactions (A & B) attempting to modify a table: mf_doom_greatest_hits.

1. Both can read the parquet file referenced by `00000.json` (transaction log)
2. They perform separate in memory transformations (the contents of the parquet file are copied into memory)
3. They both write their new versions of the `parquet` file to storage

Everything up to this point has happened without coordination. However, at step 4 both attempt to write to the JSON transaction log (atomically guaranteed - there can only be one winner):

- Transaction A attempts to write `00001.json` → succeeds
- Transaction B attempts to write `00001.json` → fails (file exists)

Assuming there are **irreconcilable differences** (when both transactions modify overlapping data files - for example, both updating rows in the same partition), Transaction B pays a heavy price: It has wasted the compute for modifying and writing a parquet file that will never be referenced.

This trade-off might seem harsh - wasted compute for simple coordination. But Delta Lake targets analytical workloads with large batch jobs, not high-concurrency OLTP with frequent small updates. The occasional wasted computation is a fair price for avoiding the complexity of fine-grained locking mechanisms like MVCC. Platforms like Databricks are offering [row level concurrency](https://www.databricks.com/blog/deep-dive-how-row-level-concurrency-works-out-box)) but it's not a Delta format native feature.

If the differences are reconcilable Transaction B just writes `00002.json` immediately. In such contexts OCC is fantastic there's no coordination required in the compute heavy phases. Coordination is only needed in the cheapest step (writing metadata to JSON) where the first sign of conflict appears. Conflicts in analytical work loads are rare  (that's why it's an optimistic mechanism).
## Why Conflicts Are Rare

So what makes conflicts in analytical work loads rare?

Analytical workloads usually don't work on overlapping data:

- Incremental loading (rarely modifying previous files)
- Generally low write concurrency it's read heavy (BI Reports etc)
- Workloads are typically scheduled to not overlap time partitions (Overnight batch jobs)

In general analytical workloads are far more controlled and predictable. Unlike the random nature of OLTP workloads. for example, many concurrent transactions attempting to decrement the same value (think Amazon shopping carts).

At this stage that covers how the delta format adds the I in ACID atop a Data Lake. However, I'd like to categories the types of conflicts and how they are resolved.

I'm surprised at how simple concurrency control is in Delta Lakes compared to say #MVCC . It's almost too easy but given the analytical workloads it makes sense.