# Conclusions

We've run the same experiment seven times. Same dataset, same embedding model, same ground truth, same query. The results are in and the picture is clear enough to draw some practical conclusions.

## The numbers in one place

| Library | Recall@10 | Latency (ms) | Metadata filtering | Disk-based |
|---|---|---|---|---|
| FAISS IndexHNSWFlat | 1.000 | 0.060 | No | No |
| Voyager | 0.888 | 0.295 | No | No |
| Sklearn brute force | 0.158 | 17.530 | No | No |
| PyNNDescent | 0.900 | 0.043 | No | No |
| USearch HNSW | 0.897 | 0.143 | No | No |
| Chroma | 0.899 | 0.776 | Yes | No |
| LanceDB IVF-PQ | 0.828 | 4.986 | Yes | Yes |

A note on the recall numbers: every library except FAISS and sklearn appears capped at around 0.88 to 0.90. This is a measurement artifact, not a genuine performance ceiling. Our ground truth was computed with FAISS using L2 distance; the other libraries use cosine distance. On normalized vectors these are mathematically equivalent in theory but floating point differences in implementation produce slightly different result orderings at the margin. Against a same-metric ground truth the approximate libraries would score higher. FAISS gets credit for recall 1.0 because it's both the ground truth provider and one of the libraries being evaluated.

The sklearn figure of 0.158 is the same artifact at larger scale, compounded by the fact that we evaluated it on 50,000 records rather than 500,000.

## What we actually learned

**There is no single best library.** The right choice depends on what you're optimizing for. The table above has five dimensions and the libraries trade them off differently. No library wins on all five.

**HNSW is the dominant algorithm for in-memory search.** FAISS, Voyager, USearch and PyNNDescent all use it or something similar. The differences between them are in API design, default parameters, persistence and platform optimization - not in the fundamental algorithm.

**PyNNDescent is the surprise of the book.** A pure Python library posting 0.043ms query latency - faster than a C++ library - is not what you'd expect. Numba JIT compilation is doing real work here. If you need fast in-memory search without FAISS's complexity, PyNNDescent deserves serious consideration.

**The exact search claim for USearch does not hold on Apple Silicon.** This is the most useful negative result in the book. SIMD optimization is architecture-specific. Benchmark on your target hardware before committing to a library based on headline numbers.

**Metadata filtering has a real cost.** Chroma and LanceDB are the only libraries that support it natively and both show higher latency than the pure search libraries as a result. At 0.776ms and 4.986ms respectively, they are still fast enough for most applications. The question is whether you need the filtering at all - if you don't, the pure search libraries are significantly faster.

**Disk-based storage changes the tradeoff fundamentally.** LanceDB is the only library in this book that works beyond the RAM boundary. Its recall and latency numbers are the worst of the group, but they are the only numbers that remain stable as your dataset grows past what fits in memory. If your dataset is large enough, LanceDB is not competing with the other libraries - it's the only option in the room.

**Annoy is broken on Apple Silicon.** We attempted to include it and discovered that it consistently returns only one neighbor regardless of K on M-series chips. It was dropped from the book. If you're on Linux or x86 hardware it may work correctly, but we cannot verify this and the library is not actively maintained.

## A decision framework

Here is a simplified way to choose:

**Start with PyNNDescent** if your dataset fits in memory, you do not need metadata filtering and you want the simplest path to fast approximate search. It has sensible defaults, no compilation step for the user and good recall.

**Reach for FAISS** if you need exact results, are working at very large scale within memory, or want fine-grained control over index type and parameters. It's the most flexible and best-documented library in this book.

**Use Voyager** if you want HNSW with a cleaner API than FAISS, need dynamic insertion and appreciate that it's battle-tested at Spotify's scale.

**Use USearch** if you're deploying to edge or mobile environments where binary size matters or if you're on Intel hardware and want the exact search speedup. Avoid it for exact search on Apple Silicon.

**Use Chroma** if you're building a RAG pipeline, need metadata filtering and want persistence without managing it yourself. It's the natural choice for LLM application development.

**Use LanceDB** if your dataset does not fit in RAM or if you need to scale to cloud storage without changing your code. Accept the recall and latency costs as the price of that capability.

**Avoid scikit-learn NearestNeighbors** for anything beyond small datasets or quick experiments. At 384 dimensions the tree-based algorithms offer no advantage over brute force and brute force is 450 times slower than FAISS HNSW at ten times the data volume.

## What this book did not cover

We focused on single-node in-process search. We didn't cover distributed vector search, streaming index updates at scale, or the interaction between vector search and traditional relational queries in a production data platform. For those topics the companion book - *[Seven Vector Databases in Seven Days](https://seven-vector-databases.github.io/)* - covers the infrastructure layer: PostgreSQL with pgvector, MongoDB Atlas, Pinecone, Weaviate, Neo4j, Snowflake and Databricks.

We also didn't cover embedding model selection. We used `all-MiniLM-L6-v2` throughout because it's a standard baseline that makes the library comparisons clean. In practice, the choice of embedding model often matters more than the choice of search library. A better model with a slower library will outperform a worse model with the fastest library every time.

## A final observation

The most useful result in this book is not a latency number or a recall figure. It's the observation from Day 1 that all ten results for our sample burglary query came from the same neighborhood using the same template. Embedding similarity is linguistic similarity, not semantic similarity. The model found incidents described in similar language  and in a template-generated dataset, that means incidents from the same template.

In production with real officer-written narratives, the clusters are richer and the results more genuinely useful. But the fundamental point holds: vector search finds what is linguistically similar to your query. Understanding what your embedding model considers similar - and whether that matches what you consider similar - is the most important question to answer before deploying any of the libraries in this book.
