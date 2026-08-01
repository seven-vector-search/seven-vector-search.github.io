# Day 7: LanceDB

LanceDB is an open-source vector database built on Lance, a columnar storage format designed for AI workloads. It's the final chapter of this book for a deliberate reason: LanceDB sits furthest along the spectrum from pure library to full database and it solves a problem none of the previous six tools address - what happens when your dataset does not fit in RAM?

Every library and database we've used so far loads indexes and vectors into memory. LanceDB stores data on disk in the Lance columnar format and queries it without loading everything into RAM. This makes it the natural choice for datasets that are too large for available memory, while still supporting vector search, metadata filtering and SQL-style queries.

## When to reach for LanceDB

LanceDB is the right choice when:

- Your dataset does not fit in RAM - LanceDB queries disk-resident data efficiently
- You need SQL-style analytics alongside vector search - DuckDB integration gives you full SQL over your vector store
- You want versioned data with Git-style branching - Lance format supports time travel and rollback
- You need tight integration with the Arrow and pandas ecosystem
- You want a zero-server embedded database that scales to cloud storage (S3, GCS) without code changes

## What we found

We loaded all 500,000 Nordvik records into LanceDB in batches of 10,000, then built an IVF-PQ index with 256 partitions and 96 sub-vectors. The IVF-PQ index type introduces two sources of approximation that distinguish it from the HNSW indexes used elsewhere in this book: the IVF partitioning, which assigns vectors to cells and searches only the nearest cells at query time and product quantization, which compresses vectors to a fraction of their original size.

Recall came out at 0.828 - the lowest of any approximate library in the book. Latency was 4.986ms - the second highest after scikit-learn. Both figures are the honest cost of disk-based storage and double approximation.

| Config | Recall@10 | Latency (ms) | Metadata | Disk-based |
|---|---|---|---|---|
| LanceDB IVF-PQ | 0.828 | 4.986 | Yes | Yes |
| Chroma (reference) | 0.899 | 0.776 | Yes | No |
| FAISS IndexHNSWFlat (reference) | 1.000 | 0.060 | No | No |

One detail worth noting: the self-match distance for rank 1 - the query incident finding itself - came out at 0.030 rather than 0.000. This is product quantization introducing small distortions even for the exact query vector. It's a visible reminder that IVF-PQ is an approximation at the storage level, not just the search level.

## Metadata filtering

LanceDB uses SQL-style `where` clauses rather than Chroma's dictionary filter syntax. This will feel natural to anyone who has worked with a database:

```python
table.search(query_vec).where("CrimeType = 'Burglary'").limit(10)
table.search(query_vec).where("Weapon != 'none'").limit(10)
table.search(query_vec).where("Neighborhood = 'Millfield'").limit(10)
```

The filtering results showed the same pattern as Chroma: the burglary and armed incident filters worked cleanly and the Millfield filter produced higher distances (0.69 and above) than the unfiltered results (0.03 and above). The jump is more pronounced than in Chroma because IVF-PQ compression compounds with the geographic constraint.

## Analytics

LanceDB stores data in the Lance columnar format which is designed for efficient analytical queries. We used the source parquet directly for our analytics demonstration - computing crime type distributions and armed incident counts by neighborhood - which is the practical approach in production. You would run analytics on raw data and reserve LanceDB for vector search queries.

The crime type distribution matched our generation weights precisely: Burglary at 35%, Vehicle Crime at 25%, Robbery at 20%, Assault at 10%, Drug Offences at 7% and Other at 3%. The armed incident counts by neighborhood were evenly distributed across all 15 neighborhoods, which is expected from a random generator but would not be true of real crime data.

## API notes

Several things changed or behaved unexpectedly in LanceDB 0.33.0 that are worth flagging:

- `table_names()` is deprecated - use `db.list_tables()` instead
- `create_table` needs `mode="overwrite"` when rerunning a notebook, otherwise it fails with a `ValueError` if the table already exists
- `_distance` column in query results is inaccessible via `itertuples()` because pandas renames underscore-prefixed columns - use `iterrows()` with `row['_distance']` instead
- `.select()` that omits `_distance` triggers a deprecation warning; omitting the select clause entirely silences it

## Cloud storage

LanceDB's most significant production advantage - one we did not demonstrate in the notebook but worth calling out - is that it supports S3, GCS and Azure Blob Storage as backends with no code changes. Swap the local path for a cloud URI and the rest of the code is identical. This is the clearest path from local development to production of any library in this book.

## What LanceDB does not do

LanceDB's IVF-PQ index trades recall for storage efficiency. With 96 sub-vectors on 384-dimensional vectors, each vector is compressed from 1,536 bytes to 96 bytes - a 16x reduction. This is what enables LanceDB to handle datasets that would require hundreds of gigabytes of RAM in FAISS. But that compression is lossy and the recall cost is real.

For datasets that fit comfortably in memory and where latency matters, the in-memory HNSW libraries from earlier chapters are faster and more accurate. LanceDB's advantages are most visible at scale.

## When to look elsewhere

LanceDB is the wrong choice if:

- You need the highest recall - IVF-PQ compression loses information; use FAISS or PyNNDescent for maximum accuracy
- Your dataset fits comfortably in RAM and latency is critical - in-memory HNSW libraries are faster
- You want the simplest possible setup - FAISS or PyNNDescent are far simpler to get started with
- You need the exact search guarantee - LanceDB's IVF-PQ is always approximate
