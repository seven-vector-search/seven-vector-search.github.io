# Day 1: FAISS

FAISS (Facebook AI Similarity Search) is Meta's open-source library for efficient similarity search and clustering of dense vectors. It's the natural starting point for this book because it exposes the full index selection journey more clearly than any other library. Rather than hiding the tradeoff between speed and accuracy behind a single API call, FAISS asks you to choose an index type explicitly. That choice - flat, IVF or HNSW - is the central lesson of this chapter.

## When to reach for FAISS

FAISS is the right choice when:

- You need exact or approximate nearest neighbor search in Python with minimal dependencies
- Your dataset fits in RAM and you want fast, well-benchmarked performance
- You want control over the index type and its parameters
- You're prototyping and want a library that scales from 10k to 100M vectors without changing your API calls

## The three index types

FAISS offers many index types but three cover the vast majority of use cases.

**IndexFlatL2** stores vectors as-is and computes exact L2 distances against every vector in the index at query time. There's no training step, no approximation and no parameters to tune. It's the ground truth against which every other index is measured. The cost is linear scaling - query time grows proportionally with the number of vectors.

**IndexIVFFlat** partitions the vector space into Voronoi cells using k-means clustering. At query time it searches only the nearest cells rather than the entire index. Two parameters control the tradeoff: `nlist` (the number of cells) and `nprobe` (the number of cells searched at query time). Higher `nprobe` improves recall at the cost of speed. Unlike `IndexFlatL2`, an IVF index must be trained before adding vectors.

**IndexHNSWFlat** builds a multi-layer graph of vectors where each node connects to its nearest neighbors. Queries traverse the graph from the top layer down, narrowing the search at each level. This produces excellent recall at low latency with no training step required. Two parameters matter: `M`, the number of connections per node, which is fixed when the index is built, and `efSearch`, the size of the candidate list explored at query time. Higher `efSearch` trades speed for recall.

## What we found

We ran all three index types against the full 500,000-record Nordvik dataset using a single burglary incident as the query. The results were clear, though every recall figure below comes from that one query.

The flat index is exact but slow. At 500,000 records it takes around 16ms per query averaged over 50 runs. That's fast enough for batch processing but too slow for interactive applications at scale.

The IVF index showed the most interesting behavior. At `nprobe=1` recall dropped to 0.40 - the only setting in this chapter that genuinely loses results. This is the approximation tradeoff in practice: searching just one Voronoi cell missed 60% of the true nearest neighbors. Recall recovered to 1.0 by `nprobe=5`, which took 0.16ms. At `nprobe=10` recall was still 1.0 at 0.29ms - roughly 54 times faster than the flat index with no accuracy loss on this query. Latency at `nprobe=1` (0.24ms) measured slower than at `nprobe=5`, which is timing noise on a single query rather than a real effect. Beyond `nprobe=10`, extra probing only added cost: latency climbed to 1.21ms at `nprobe=50` with no gain in recall.

The HNSW index was the standout performer. At `efSearch=64` it achieved recall 1.0 at 0.06ms - more than 250 times faster than the flat index. No training required, no recall degradation on this query, just a larger memory footprint than IVF. Recall was 1.0 at every `efSearch` value we tried, from 16 to 256, so on this dataset the setting mainly moves latency. The fastest setting was `efSearch=32` at 0.04ms, but the differences at this scale are tiny and partly noise.

The scaling chart tells the most important story. Flat index latency grows roughly linearly, from 0.23ms at 10,000 records to 14.43ms at 500,000 (this separate scaling run differs slightly from the 15.75ms measured above because of timing variance). IVF and HNSW grow far more slowly: IVF goes from 0.03ms to 0.46ms and HNSW from 0.03ms to 0.08ms over the same range. We didn't measure beyond 500,000 records, but extending the flat index's linear trend to 10 million would put it near 290ms per query, while the approximate indexes would be expected to stay far lower.

| Index | Recall@10 | Latency (ms) |
|---|---|---|
| IndexFlatL2 | 1.00 | 15.75 |
| IndexIVFFlat (nprobe=1) | 0.40 | 0.24 |
| IndexIVFFlat (nprobe=10) | 1.00 | 0.29 |
| IndexHNSWFlat (efSearch=64) | 1.00 | 0.06 |

## A note on the query results

The ten results for our sample burglary query include the query incident itself at rank one with a distance of 0, so there are really nine neighbors. Nine of the ten are labeled Burglary and the last is labeled Other. All ten share the same narrative pattern: offender descriptions, entry to a residence and property taken. All three indexes returned the same ten incidents in the same order. This is the embedding model doing exactly what it was designed to do: finding linguistic similarity. In a dataset generated from templates, linguistic similarity and template similarity are almost the same thing.

This is worth keeping in mind throughout the book. Embedding-based search finds incidents described in similar language. In production with officer-written narratives - which have far more linguistic variety - the clusters would be semantically richer. Our synthetic data are honest about this limitation.

## What FAISS does not do

FAISS is a library, not a service. It has no built-in persistence - you must serialize indexes yourself using `faiss.write_index` and `faiss.read_index`. There's no server and no authentication. It also has no native metadata filtering of the kind a database offers. You can restrict a search to a set of IDs, but if you need to filter by crime type or neighborhood before running vector search, you need to build that logic yourself or move to Chroma (Day 6) or LanceDB (Day 7).

The IVF index requires training data representative of your full dataset. If your data distribution shifts over time the index will degrade and need retraining. HNSW doesn't require training but typically consumes significantly more memory than IVF at the same dataset size.

## When to look elsewhere

FAISS is the wrong choice if:

- You need metadata filtering alongside vector search
- You need a persistent queryable store - look at Chroma (Day 6)
- Your dataset doesn't fit in RAM - look at LanceDB (Day 7)
- You want automatic parameter tuning - other libraries handle this more gracefully
