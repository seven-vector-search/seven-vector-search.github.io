# Day 1: FAISS

FAISS (Facebook AI Similarity Search) is Meta's open-source library for efficient similarity search and clustering of dense vectors. It's the natural starting point for this book because it exposes the full index selection journey more clearly than any other library. Rather than hiding the tradeoff between speed and accuracy behind a single API call, FAISS asks you to choose an index type explicitly. That choice - flat, IVF or HNSW - is the central lesson of this chapter.

## When to reach for FAISS

FAISS is the right choice when:

- You need exact or approximate nearest neighbor search in Python with minimal dependencies
- Your dataset fits in RAM and you want fast, well-benchmarked performance
- You want control over the index type and its parameters
- You are prototyping and want a library that scales from 10k to 100M vectors without changing your API calls

## The three index types

FAISS offers many index types but three cover the vast majority of use cases.

**IndexFlatL2** stores vectors as-is and computes exact L2 distances against every vector in the index at query time. There is no training step, no approximation and no parameters to tune. It's the ground truth against which every other index is measured. The cost is linear scaling - query time grows proportionally with the number of vectors.

**IndexIVFFlat** partitions the vector space into Voronoi cells using k-means clustering. At query time it searches only the nearest cells rather than the entire index. Two parameters control the tradeoff: `nlist` (the number of cells) and `nprobe` (the number of cells searched at query time). Higher `nprobe` improves recall at the cost of speed. Unlike `IndexFlatL2`, an IVF index must be trained before adding vectors.

**IndexHNSWFlat** builds a multi-layer graph of vectors where each node connects to its nearest neighbors. Queries traverse the graph from the top layer down, narrowing the search at each level. This produces excellent recall at low latency with no training step required. The key parameter is `M` -- the number of connections per node.

## What we found

We ran all three index types against the full 500,000-record Nordvik dataset. The results were clear.

The flat index is exact but slow. At 500,000 records it takes around 14ms per query averaged over 50 runs. That is fast enough for batch processing but too slow for interactive applications at scale.

The IVF index showed the most interesting behavior. At `nprobe=1` recall dropped to 0.40 - the only setting in this chapter that genuinely loses results. This is the approximation tradeoff in practice: searching just one Voronoi cell misses 60% of the true nearest neighbors. At `nprobe=10` recall recovered to 1.0 at 0.23ms - 60 times faster than the flat index with no accuracy loss.

The HNSW index was the standout performer. At `efSearch=64` it achieved recall 1.0 at 0.06ms - 230 times faster than the flat index. No training required, no recall degradation, just a larger memory footprint than IVF.

The scaling chart tells the most important story: flat index latency grows linearly from near zero at 10,000 records to 17ms at 500,000, while IVF and HNSW stay essentially flat throughout. At 10 million records the flat index would be unusable for interactive queries; the approximate indexes would barely notice.

| Index | Recall@10 | Latency (ms) |
|---|---|---|
| IndexFlatL2 | 1.00 | 14.00 |
| IndexIVFFlat (nprobe=1) | 0.40 | 0.11 |
| IndexIVFFlat (nprobe=10) | 1.00 | 0.23 |
| IndexHNSWFlat (efSearch=64) | 1.00 | 0.06 |

## A note on the query results

All ten results for our sample burglary query came from the same neighborhood using the same template structure. Every result was some variation of "Residence - Single Family in Hartley Cross targeted." This is the embedding model doing exactly what it was designed to do: finding linguistic similarity. In a dataset generated from templates, linguistic similarity and template similarity are almost the same thing.

This is worth keeping in mind throughout the book. Embedding-based search finds incidents described in similar language. In production with officer-written narratives - which have far more linguistic variety - the clusters would be semantically richer. Our synthetic data is honest about this limitation.

## What FAISS does not do

FAISS is a library, not a service. It has no built-in persistence - you must serialize indexes yourself using `faiss.write_index` and `faiss.read_index`. There is no server, no authentication and no metadata filtering. If you need to filter by crime type or neighborhood before running vector search, you need to implement that yourself or move to Chroma (Day 6) or LanceDB (Day 7).

The IVF index requires training data representative of your full dataset. If your data distribution shifts over time the index will degrade and need retraining. HNSW does not require training but consumes significantly more memory than IVF at the same dataset size.

## When to look elsewhere

FAISS is the wrong choice if:

- You need metadata filtering alongside vector search
- Your dataset does not fit in RAM - look at LanceDB (Day 7)
- You need a persistent queryable store - look at Chroma (Day 6)
- You want automatic parameter tuning - other libraries handle this more gracefully
