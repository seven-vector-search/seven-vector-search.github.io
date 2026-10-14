# Day 7: LanceDB

LanceDB is an open-source vector database built on Lance, a columnar storage format designed for AI workloads. It's the final chapter of this book for a deliberate reason: LanceDB sits furthest along the spectrum from pure library to full database and it targets a problem we haven't tested for in the earlier chapters - what happens when your dataset doesn't fit in RAM?

LanceDB stores data on disk in the Lance columnar format and queries it without loading everything into RAM. This makes it the natural choice for datasets that are too large for available memory, while still supporting vector search and metadata filtering with SQL-style conditions.

## When to reach for LanceDB

LanceDB is the right choice when:

- Your dataset doesn't fit in RAM - LanceDB queries disk-resident data without loading the whole table into memory
- You want SQL-style metadata filters alongside vector search - `where` clauses are plain SQL strings
- You want versioned data - Lance tables keep every version and support time travel, restore and tags
- You need tight integration with the Arrow and pandas ecosystem
- You want a zero-server embedded database that can also use cloud storage (S3, GCS) - we didn't test the cloud side

## What we found

We loaded all 500,000 Nordvik records into LanceDB in batches of 10,000, then built an IVF-PQ index with 256 partitions and 96 sub-vectors. Loading took 7.1 and 7.6 seconds in our two runs and the index build took 42.8 and 43.0 seconds, a small fraction of the 321 to 415 seconds Chroma needed to load the same records. The table took 3,407 and 4,271 MiB on disk in the two runs, about 4.7 to 5.8 times the 732 MiB of raw embeddings, and we didn't investigate why the runs differed. The index added only 49 MiB in both.

The IVF-PQ index type introduces two sources of approximation that distinguish it from the HNSW indexes used elsewhere in this book: the IVF partitioning, which assigns vectors to cells and searches only the nearest cells at query time, and product quantization, which compresses vectors to a fraction of their original size.

With default search settings recall came out at 0.8827 at 3.476ms, averaged over the same 1,000 sampled queries and Day 1 ground truth we used for the other libraries. As before, we asked for 11 results and removed the query's own ID before scoring. That's the lowest recall and the highest latency of any approximate index in the book.

| Config | Recall@10 | Latency (ms) | Metadata | Disk-based |
|---|---|---|---|---|
| LanceDB IVF-PQ, default settings | 0.8827 | 3.476 | Yes | Yes |
| LanceDB IVF-PQ, nprobes=10, refine_factor=5 (first 200 queries) | 0.9970 | 5.248 | Yes | Yes |
| Chroma (reference) | 0.9967 | 0.532 | Yes | Not tested |
| FAISS IndexHNSWFlat (reference) | 1.000 | 0.060 | No | Not tested |

Much of the recall gap comes from the default settings. Without refinement, recall stopped improving at about 0.8845 however many partitions we searched (`nprobes` of 10, 20 and 50 all gave 0.8845), so quantization error rather than partition coverage was the limit. Asking LanceDB to re-score candidates on the full vectors with `refine_factor=5` raised recall to 0.9895 at `nprobes=5` (4.713ms), 0.9970 at `nprobes=10` (5.248ms) and 1.0000 at `nprobes=50` (9.216ms). A `refine_factor` of 20 gave the same recall as 5, only slower. We measured these settings on the first 200 queries.

One detail worth noting: the query incident found itself at rank 1 with a distance of 0.037 rather than 0.000. Re-scoring on the full vectors (`refine_factor=1`) brought it to 0.000, so the difference comes from product quantization. It's a visible reminder that IVF-PQ is an approximation at the storage level, not just the search level.

## Does it stay out of RAM?

This is LanceDB's headline claim, so we tested it as far as our laptop allows. We ran 200 queries in a separate Python process that opened the table but never loaded the embeddings or the dataframe. Its peak memory was 137 MiB right after opening and 228 MiB after the queries, against a table of 4,320 MiB on disk. The first query took 10.5ms and the rest averaged 3.559ms, so expect a slower first query after a cold start. We didn't separate the operating system's file cache from other start-up costs, and a table larger than the machine's memory would be the real test.

## Metadata filtering

LanceDB uses SQL-style `where` clauses rather than Chroma's dictionary filter syntax. This will feel natural to anyone who has worked with a database:

```python
table.search(query_vec).where("CrimeType = 'Burglary'").limit(10)
table.search(query_vec).where("Weapon != 'none'").limit(10)
table.search(query_vec).where("Neighborhood = 'Millfield'").limit(10)
```

The filtering results showed the same pattern as Chroma. We checked that every result satisfied its filter, for these three queries and for 50 further queries per filter. The Millfield filter produced larger distances (0.7039 to 0.7673) than the nearest unfiltered neighbor (0.2721, excluding the query itself), a ratio of about 2.6 against about 2.4 in Chroma. LanceDB's distances are approximate and aren't comparable with Chroma's.

Filtering cost less here than in Chroma. Averaged over 200 queries, an unfiltered search took 3.369ms against 13.909ms for Burglary, 16.603ms for armed incidents and 10.692ms for Millfield, roughly three to five times slower. Chroma's filtered queries took 126 to 426ms. The three filters match very different shares of the records: about 7% for Millfield, 35% for Burglary and about 40% for armed incidents. Latency rose in the same order, which is consistent with cost tracking the number of matching records, although we didn't test that directly.

LanceDB applies the filter before the vector search by default. To check, we picked the rarest combination of crime type and neighborhood with at least 20 records (Other incidents in Northgate, 941 of the 500,000 records). With the default pre-filtering we got all ten results. With `prefilter=False` we got none, and none of those records appeared in the unfiltered top 100. If you turn pre-filtering off, a selective filter can return fewer results than you asked for.

## Versioning

Every write to a Lance table creates a new version, and old versions stay readable. After loading and indexing, our table had 255 versions, more than the 50 writes plus one index build we expected, and we didn't investigate why. Checking out version 1 showed the first 10,000 rows, and returning to the latest version restored all 500,000. We also tagged a version by name. We didn't try branches.

## Analytics

LanceDB stores data in the Lance columnar format, which is designed for efficient analytical queries. We ran our analytics demonstration on the source parquet with pandas instead, computing crime type distributions and armed incident counts by neighborhood, so we didn't test SQL or DuckDB access to the Lance table.

The crime type distribution matched our generation weights: Burglary at 35%, Vehicle Crime at 25%, Robbery at 20%, Assault at 10%, Drug Offences at 7% and Other at 3%. The generator picks each neighborhood uniformly from 15, so the armed incident counts are evenly spread (the top ten ranged from 13,247 to 13,553, and we printed only those ten). Real crime data wouldn't look like this.

## API notes

A few things behaved unexpectedly and are worth flagging:

- A `.select()` that omits `_distance` triggers a deprecation warning, so include `_distance` in the select list
- The `_distance` column is inaccessible via `itertuples()` because pandas renames underscore-prefixed columns - use `iterrows()` with `row['_distance']` instead
- LanceDB's documentation says the search metric must match the metric used to build the index, so set `distance_type` explicitly

## Cloud storage

LanceDB supports S3, GCS and Azure Blob Storage as backends. We didn't test them in the notebook, so check the documentation for how to switch the connection URI and supply credentials. If it works as described it's a path from local development to production, but we can't vouch for it.

## What LanceDB does not do

LanceDB's IVF-PQ index trades recall for index size. With 96 sub-vectors on 384-dimensional vectors, each vector is compressed from 1,536 bytes to 96 bytes in the index, a 16x reduction, and the index added only 49 MiB to our table. The full-precision vectors stay on disk, so the table itself doesn't get smaller. That's what lets LanceDB search large datasets without holding the vectors in memory. The compression is lossy, and at default settings the recall cost is real. In our tests re-scoring candidates on the full vectors recovered most of it, at the price of extra latency.

For datasets that fit comfortably in memory and where latency matters, the in-memory HNSW libraries from earlier chapters are faster and, at their default settings, more accurate. LanceDB's advantages are most visible at scale, which we couldn't test on our machine.

## When to look elsewhere

LanceDB is the wrong choice if:

- You need the highest recall at low latency - IVF-PQ compression loses information, and recovering it by re-scoring costs time (5.2ms for 0.997 recall in our tests); use FAISS or PyNNDescent
- Your dataset fits comfortably in RAM and latency is critical - in-memory HNSW libraries are faster
- You want the simplest possible setup - FAISS or PyNNDescent are simpler to get started with
- You need the exact search guarantee - LanceDB's IVF-PQ is always approximate
