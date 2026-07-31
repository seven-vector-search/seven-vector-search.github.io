# Day 4: PyNNDescent

PyNNDescent is a pure Python implementation of the Nearest Neighbor Descent algorithm, developed by Leland McInnes at the Tutte Institute for Mathematics and Computing. If you've used UMAP - one of the most widely used dimensionality reduction libraries in data science - you've already used PyNNDescent. It's the nearest neighbor engine that powers UMAP under the hood.

Unlike FAISS and Voyager which build HNSW graphs, PyNNDescent builds a k-neighbor graph by iterative refinement. It starts from a random graph and repeatedly improves it by checking whether neighbors of neighbors are better neighbors. This descent process converges to a high-quality approximate graph without requiring the careful parameter tuning that HNSW demands.

## When to reach for PyNNDescent

PyNNDescent is the right choice when:

- You want high recall approximate nearest neighbor search with minimal parameter tuning
- You are already using UMAP and want consistency between your dimensionality reduction and search pipelines
- You need a pure Python library with no C++ compilation requirements
- You want scikit-learn compatibility via `PyNNDescentTransformer` as a drop-in replacement

## What we found

PyNNDescent produced the most surprising result of the book. At n_neighbors=10 it achieved a query latency of 0.043ms - faster than FAISS HNSW at 0.06ms. A pure Python library outperforming a C++ library with GPU support is not what you would expect and it's worth understanding why.

The answer is the JIT compilation step. PyNNDescent uses Numba to compile its search function on first use. After that warmup - which we call explicitly in the notebook before timing - subsequent queries run compiled native code. The 0.043ms figure reflects post-warmup performance. If cold-start latency matters in your deployment, factor in several seconds for the first query.

Recall at n_neighbors=10 was 0.884, rising to 0.900 at n_neighbors=50. The plateau is familiar from Days 2 and 3 - it's the distance metric mismatch with the FAISS L2 ground truth rather than a genuine ceiling. Build time ranged from 5.4 seconds at n_neighbors=10 to 28.6 seconds at n_neighbors=50, which is notably faster than Voyager's 50 seconds for an equivalent M=32 index.

| Config | Recall@10 | Latency (ms) | Build time |
|---|---|---|---|
| n_neighbors=10 | 0.884 | 0.043 | 5.4s |
| n_neighbors=20 | 0.898 | 0.026 | 8.9s |
| n_neighbors=30 | 0.900 | 0.030 | 14.2s |
| n_neighbors=50 | 0.900 | 0.051 | 28.6s |

## Query result quality

The query results showed something encouraging that Day 1 did not. Where Day 1 returned ten results from the same template and the same neighborhood - all variations on "Residence - Single Family in Hartley Cross targeted" - PyNNDescent returned results from different neighborhoods and different template structures. The top nine were burglaries as expected, but they came from Hartley Cross, Millfield and other neighborhoods, using a variety of template openings.

Rank 10 was occasionally an Other crime type at very close distance. This is not an error - it reflects a genuine linguistic similarity between certain crime descriptions that crosses the categorical boundary. A forced entry incident described in similar language to a burglary will score high cosine similarity regardless of how it was categorized.

## The scikit-learn integration

PyNNDescent provides a `PyNNDescentTransformer` that's a drop-in replacement for scikit-learn's `KNeighborsTransformer`. This means we can swap PyNNDescent into any scikit-learn pipeline that uses nearest neighbors - including UMAP, t-SNE wrappers and graph-based clustering - without changing our pipeline code. We demonstrated this on a 5,000-record subset, producing a sparse graph matrix that could be passed directly to UMAP or any sklearn graph estimator.

## What PyNNDescent does not do

PyNNDescent does not support dynamic insertion. Like Annoy, any new data requires a full rebuild. The index can be pickled to disk with Python's standard `pickle` module, which is simpler than FAISS's `write_index` but produces larger files.

Build time at 500,000 vectors is longer than FAISS for equivalent quality, though faster than Voyager. For datasets that grow continuously, the rebuild cost needs to be factored into operational planning.

## When to look elsewhere

PyNNDescent is the wrong choice if:

- You need the absolute lowest query latency after cold start - the JIT warmup adds several seconds to the first query
- Your dataset changes frequently - there is no dynamic insertion; use Voyager or FAISS instead
- You need metadata filtering - look at Chroma (Day 6) or LanceDB (Day 7)
- You need a language-agnostic library - PyNNDescent is Python only
