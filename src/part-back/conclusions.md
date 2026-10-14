# Conclusions

We've run the same experiment seven times. Same dataset, same embedding model and same ground truth, scored on the same 1,000 sampled queries, with scikit-learn on a 50,000-record subset as the one exception. The results are in and the picture is clear enough to draw some practical conclusions.

## The numbers in one place

| Library | Recall@10 | Latency (ms) | Metadata filtering | Disk-based |
|---|---|---|---|---|
| FAISS IndexHNSWFlat | 1.000 | 0.060 | No | Not tested |
| Voyager (M=12, ef=200) | 0.980 | 0.290 | No | Not tested |
| Scikit-learn exact (50,000 records) | 1.000 | 25 | No | Not tested |
| PyNNDescent (n_neighbors=50) | 0.999 | 0.056 | No | Not tested |
| USearch HNSW (ef=16) | 0.995 | 0.157 | No | Not tested |
| USearch HNSW (ef=64) | 1.000 | 0.335 | No | Not tested |
| Chroma | 0.997 | 0.532 | Yes | Not tested |
| LanceDB IVF-PQ (default) | 0.883 | 3.476 | Yes | Yes |
| LanceDB IVF-PQ (nprobes=10, refine_factor=5) | 0.997 | 5.248 | Yes | Yes |

Notes on the table:

- Recall is averaged over 1,000 sampled queries after removing each query's own ID. FAISS's recall and latency come from a single query.
- Scikit-learn ran exact search on a 50,000-record subset, so its latency isn't comparable with the others, which ran on all 500,000 records.
- The tuned LanceDB row was measured on the first 200 queries.
- "Disk-based" means we checked that the table can be queried without loading it into memory. We did that only for LanceDB. "Not tested" means we didn't check, not that the library can't.

A note on the recall numbers: Day 1's ground truth excludes the query itself, but every index returns the query as its own first result. For every library we ask for 11 results and remove the query's own ID before scoring, so the figures above reflect the true ten neighbors. Voyager at M=12 (0.980) and LanceDB at its default settings (0.883) give up some recall for speed or index size. USearch, Chroma and PyNNDescent (at n_neighbors=50) reached 0.99 or better.

Scikit-learn's exact search scored 1.000 on its 50,000-record subset, as expected for a brute-force method, at about 25ms per query.

## What we actually learned

**There's no single best library.** The right choice depends on what you're optimizing for. The table above has four dimensions and the libraries trade them off differently. No library wins on all four.

**HNSW is the dominant algorithm for in-memory search.** FAISS, Voyager and USearch use it and PyNNDescent uses a related graph-based method. The differences between them show up in API design, default parameters, persistence and platform optimization, and in speed, although FAISS's figure comes from a single query so that comparison is rough.

**PyNNDescent is the surprise of the book.** A pure Python library posting 0.033 to 0.062ms query latency, depending on `n_neighbors`, is in the same range as FAISS's C++ HNSW index (0.06ms from a single query). Numba JIT compilation is a likely reason, although we didn't test that. Recall ranged from 0.958 at `n_neighbors=10` to 0.999 at 50. The first query pays about 8 seconds to prepare and compile, and building took 6 to 29 seconds. If you need fast in-memory search without FAISS's complexity, PyNNDescent deserves serious consideration.

**The exact search claim for USearch didn't reproduce on our machine.** USearch's README reports exact search up to 20 times faster than FAISS's `IndexFlatL2` in a Google Colab test. On a MacBook Air M4 a single exact query took 87.6ms against 15.75ms for FAISS flat, and 14.5ms per query when batched. We haven't established why, and we didn't test Intel hardware. This is one of the most useful negative results in the book. Benchmark on your target hardware before committing to a library based on headline numbers.

**Metadata filtering has a real cost.** Chroma and LanceDB are the only libraries in this book where we used built-in metadata filtering. Both are slower than the pure search libraries even without a filter (0.532ms and 3.476ms), and filtering adds more. In Chroma, filtered queries took 126 to 426ms against 0.575ms unfiltered. In LanceDB they took 10.7 to 16.6ms against 3.4ms. Chroma's filtered queries may be too slow for interactive use at this scale, so measure on your data. If you don't need filtering, the pure search libraries are significantly faster.

**Disk-based storage changes the tradeoff.** LanceDB is the only library in this book where we tested querying without loading the data into memory. A separate process queried a 4,320 MiB table with a peak memory of 228 MiB. Its default recall (0.883) and latency (3.476ms) are the worst of the group, but re-scoring candidates on the full vectors raised recall to 0.997 at 5.2ms. We couldn't test a dataset larger than our machine's memory, which would be the real test. If your dataset is large enough to need this, LanceDB is worth serious consideration.

**Annoy is broken on Apple Silicon.** We attempted to include it and found that it consistently returned only one neighbor regardless of K on M-series chips, as described in Day 2. We dropped it from the book. If you're on Linux or x86 hardware it may work correctly, but we can't verify this.

## A decision framework

Here's a simplified way to choose:

**Start with PyNNDescent** if your dataset fits in memory, you don't need metadata filtering and you want the simplest path to fast approximate search. It has sensible defaults, needs no C++ compilation to install (the first query compiles with Numba and takes about 8 seconds) and reaches good recall once you raise `n_neighbors`.

**Reach for FAISS** if you need exact results, are working at very large scale within memory or want fine-grained control over index type and parameters. It's the most flexible library in this book.

**Use Voyager** if you want HNSW with a cleaner API than FAISS and built-in persistence, and you value that it was developed and used at Spotify. It supports dynamic insertion too, although USearch and PyNNDescent also support updates in our tests.

**Use USearch** if you're deploying to edge or mobile environments where binary size matters, or if you want approximate and exact search, insertion, removal and custom metrics in one library. We couldn't reproduce its headline exact-search speedup on Apple Silicon.

**Use Chroma** if you're building a RAG pipeline, need metadata filtering and want persistence without managing it yourself. Budget for the costs we measured: loading 500,000 records took 321 to 415 seconds and filtered queries took 126 to 426ms.

**Use LanceDB** if your dataset doesn't fit in RAM, or if you want cloud storage without managing it (we didn't test the cloud side). Expect lower recall at default settings, and plan to tune `nprobes` and `refine_factor`.

**Avoid scikit-learn NearestNeighbors** for anything beyond small datasets or quick experiments. At 384 dimensions the tree-based algorithms offer no advantage over brute force, and brute force on 50,000 records was roughly 400 times slower than FAISS HNSW on 500,000.

## What this book did not cover

We focused on single-node in-process search. We didn't cover distributed vector search, streaming index updates at scale or the interaction between vector search and traditional relational queries in a production data platform. For those topics the companion book - *[Seven Vector Databases in Seven Days](https://seven-vector-databases.github.io/)* - covers the infrastructure layer: PostgreSQL with pgvector, MongoDB Atlas, Pinecone, Weaviate, Neo4j, Snowflake and Databricks.

We also didn't cover embedding model selection. We used `all-MiniLM-L6-v2` throughout because it's a standard baseline that makes the library comparisons clean. In practice, the choice of embedding model often matters more than the choice of search library. A better model with a slower library will often outperform a worse model with the fastest library.

## A final observation

The most useful result in this book is not a latency number or a recall figure. It's the observation from Day 1 that all ten results for our sample burglary query (the query itself and nine neighbors) shared the same narrative pattern, and nine of the ten were burglaries. Embedding similarity largely reflects the wording of the text. The embedding notebook shows it in one comparison: two burglaries described very differently had a cosine similarity of 0.41, lower than the 0.49 between one of them and an assault that happened to share its suspect description. The model found incidents described in similar language and, in a template-generated dataset, that means incidents built from the same template.

With real officer-written narratives we'd expect richer variation, although we haven't tested that. The fundamental point holds: vector search finds what your embedding model considers similar to your query, which is largely a matter of wording. Understanding what your embedding model considers similar - and whether that matches what you consider similar - is the most important question to answer before deploying any of the libraries in this book.
