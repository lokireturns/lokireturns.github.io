layout: post
title: "Delta Lake Atomicity"
date: 2025-11-21 17:07:00 -0000
categories: Delta-Lake


My notes on Atomicity in Delta Lake - it's easier for me to compare it to traditional databases.

Atomicity means that when we commit a transaction it's an all-or-nothing operation. Either the transaction succeeds or it fails, nothing in between. Consistency means that despite our transaction being atomic, if some database constraint (e.g., a foreign key) is violated, then the operation is rejected and rolled back in spite of the commits being atomic. This distinction is important and will become clear why later. Atomicity pertains to the mechanism, whereas consistency checks if the transactions make logical sense given the constraints of the data model.

Let's first cover how "traditional" databases like MySQL and Postgres achieve atomicity with the WAL. It's best understood from the perspective of **recovery**:

We make changes to pages in memory - they can be flushed to disk at unpredictable times. Imagine: we modify a page, the buffer pool keeps it in memory (maybe it's pinned), and the DB crashes before it's written to disk. That means our commit was not durable - the user received a success message but the page was still in RAM.

```
[Traditional DB - Without WAL]

Transaction modifies Page X in buffer pool
  ↓
User gets "COMMIT SUCCESS"
  ↓
💥 CRASH (before page flushes)
  ↓
Page X changes lost forever
```

By writing the WAL (with `fsync`) to disk first before acknowledging the COMMIT, the buffer pool can flush pages whenever it wants (following something like LRU-K). If the DB crashes, we simply replay the log, ensuring no partial writes.

```
[Traditional DB - With WAL]

1. Modify Page X in memory
2. Write changes to WAL + fsync  ✓
3. Return "COMMIT SUCCESS" to user
4. (Later) Flush Page X to disk
   
If crash happens at step 4:
→ Replay WAL on restart
→ Page X changes restored
```

So how does the transaction log in the Delta format give us atomicity? After all, it's not a WAL, is it?

```
[Delta Lake Architecture]

Compute Engine (Spark/DuckDB/Trino)
  ↓
Delta Libraries (embedded)
  ↓
Object Storage (S3/GCS/ADLS)
```

In the Delta format, we have a compute engine (e.g., Spark, DuckDB, Trino). The engine manages data in memory, reading and writing Parquet files. The Delta libraries are embedded in the engine to intercept and interact with the storage layer (our proverbial disk) - in this case, distributed object storage (S3, GCS, ADLS, etc.).

**Here's the key difference: There is no recovery... what?**

The compute engine does not update pages in place, so there's no need to worry about unpredictable page flushing to disk. The Parquet files are rewritten to storage, and a JSON log entry is added describing the write operation (copy-on-write). The JSON log entry says:

- `Remove: file_A.parquet` ← The one we modified
- `Add: file_B.parquet` ← The new one with modified data

Therefore, if there's a crash (typically some network error), either the JSON entry exists or it doesn't. Even if the new Parquet files exist, they are logically invisible (orphaned) to the compute engine. Thus, the sequence is opposite to traditional databases: we write the Parquet files first, and only then write our JSON log.

```
[Delta Lake Commit Sequence]

1. Write file_B.parquet (new data) ✓
2. Write JSON log entry atomically  ✓
   {
     "remove": {"path": "file_A.parquet"},
     "add": {"path": "file_B.parquet"}
   }
3. Return "COMMIT SUCCESS"

If crash at step 1:
→ file_B.parquet orphaned
→ No log entry → logically invisible
→ No recovery needed

Object storage guarantees:
Single file write is atomic ✓
```

This opens a new question! How do we handle this seemingly explosive storage growth? That'll be the next note.