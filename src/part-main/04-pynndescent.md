# Day 4: PyNNDescent

PyNNDescent is a pure Python implementation of the Nearest Neighbor Descent algorithm, developed by Leland McInnes at the Tutte Institute for Mathematics and Computing. If you've used UMAP - one of the most widely used dimensionality reduction libraries in data science - you've already used PyNNDescent. It's the nearest neighbor engine that powers UMAP under the hood.

Unlike Voyager and FAISS's HNSW index, which build HNSW graphs, PyNNDescent builds a k-neighbor graph by iterative refinement. It starts from a random graph and repeatedly improves it by checking whether neighbors of neighbors are better neighbors. This descent process converges to a high-quality approximate graph without requiring the careful parameter tuning that HNSW demands.

## When to reach for PyNNDescent

PyNNDescent is the right choice when:

- You want high recall approximate nearest neighbor search with minimal parameter tuning
- You're already using UMAP and want consistency between your dimensionality reduction and search pipelines
- You need a pure Python library with no C++ compilation requirements
- You want scikit-learn compatibility via `PyNNDescentTransformer` as a drop-in replacement

## What we found

We built PyNNDescent indexes on the full 500,000-record Nordvik dataset and scored them against the same 1,000 sampled queries and Day 1 ground truth we used for Voyager.

PyNNDescent landed in the same territory as the HNSW libraries. With n_neighbors=10 recall was 0.958. At n_neighbors=20 it reached 0.991, and at the default of 30 it was 0.997, rising to 0.999 at n_neighbors=50. That's close to Voyager at M=32 (0.997 at ef=200) and above Voyager at M=12 (0.980). Latency was between 0.03 and 0.06ms for most settings, in the same range as FAISS HNSW (0.06ms from a single query in Day 1). The figures vary between runs, though. The same n_neighbors=30 index measured 0.033, 0.052 and 0.120ms in three passes over the same queries, so we wouldn't read anything into small differences.

Build time grew with n_neighbors, from 5.8 seconds at n_neighbors=10 to 29 seconds at n_neighbors=50, with 14.6 seconds at the default of 30. That's less than half of Voyager's 34 seconds at M=12, though the two algorithms and their parameters aren't equivalent.

The first query is slow. Before it can answer, PyNNDescent prepares its search structures (pruning the graph and building a search tree) and compiles its search function with Numba. That took about 8 seconds in our run. We run a warmup query before timing, and if cold-start latency matters in your deployment you should factor it in.

Recall also responds to a query-time setting, `epsilon`, which trades accuracy for search cost. The default of 0.1 gave 0.997 on the n_neighbors=30 index, while 0.0 gave 0.985 and 0.3 gave 0.9995.

One detail is worth knowing if you run your own evaluation. As with Voyager, the index returns the query itself as its top result, while the Day 1 ground truth excludes it. We ask for 11 results and drop the query before scoring. Without that step recall can't exceed 0.9, and our first attempt stalled at 0.900 at n_neighbors=50.

| Config | Recall@10 | Latency (ms) | Build time |
|---|---|---|---|
| n_neighbors=10 | 0.958 | 0.062 | 5.8s |
| n_neighbors=20 | 0.991 | 0.037 | 9.1s |
| n_neighbors=30 | 0.997 | 0.033 | 14.6s |
| n_neighbors=50 | 0.999 | 0.056 | 29.2s |
| Voyager M=12, ef=200 (reference) | 0.980 | 0.290 | -- |
| FAISS IndexHNSWFlat (reference, single query) | 1.000 | 0.060 | -- |

The FAISS recall comes from a single query. The Voyager and PyNNDescent figures average 1,000 queries.

## Query result quality

The query results matched what FAISS and Voyager returned. PyNNDescent's top ten for our sample burglary query are the same ten incidents, in the same order and at the same distances, as Voyager's in Day 2. Rank 1 is the query incident itself, so there are really nine neighbors: eight burglaries and one Other at rank 10, at distance 0.1615. This isn't an error. It reflects a genuine linguistic similarity between certain crime descriptions that crosses the categorical boundary. A forced entry incident described in similar language to a burglary will score high cosine similarity regardless of how it was categorized.

## The scikit-learn integration

PyNNDescent provides a `PyNNDescentTransformer` that's a drop-in replacement for scikit-learn's `KNeighborsTransformer`. This means we can swap PyNNDescent into scikit-learn pipelines that use nearest neighbors without changing our pipeline code (the PyNNDescent documentation shows it feeding Isomap, for example). We demonstrated the transformer on a 5,000-record subset, producing a sparse 5,000 by 5,000 neighbor graph in 0.6 seconds.

## What PyNNDescent does not do

PyNNDescent can add data to an existing index through its `update` method, but it isn't cheap. Adding 1,000 vectors to the 500,000-record index took 9.4 seconds, against 0.1 seconds for Voyager in Day 2, and all 1,000 new vectors were found in their own top ten afterward. That's quicker than rebuilding (about 15 seconds to build plus about 8 for the first-query preparation at n_neighbors=30), but it's not the cheap insert Voyager offers.

The index can be pickled to disk with Python's standard `pickle` module, which is simpler than FAISS's `write_index`. The file is large, though: about 1.8GB for our 500,000-record index, against 788MB for Voyager. Loading it back took 1.5 seconds, and the first query on the loaded index was immediate in the same session. We haven't tested loading it in a fresh process.

Build time at 500,000 vectors ranged from 6 to 29 seconds depending on n_neighbors. We haven't compared it with FAISS here. For datasets that grow continuously, the cost of each update needs to be factored into operational planning.

## When to look elsewhere

PyNNDescent is the wrong choice if:

- You can't tolerate a cold start - the first query on a freshly built index took about 8 seconds
- Your dataset changes frequently - `update` works but took about 9 seconds per batch at this scale, and Voyager inserts are far cheaper
- You need metadata filtering - look at Chroma (Day 6) or LanceDB (Day 7)
- You need a language-agnostic library - PyNNDescent is Python only
