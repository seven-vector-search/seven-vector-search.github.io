# Day 3: Scikit-learn NearestNeighbors

Scikit-learn's `NearestNeighbors` is the vector search library you already have. If you have scikit-learn installed - and most Python data science and machine learning projects do - you have a capable nearest neighbor search implementation ready to use without any additional dependencies.

Unlike Voyager and FAISS's HNSW index, which use graph-based search, scikit-learn offers three distinct algorithms: brute force, ball tree and kd-tree. Each has different characteristics and the library selects the most appropriate one automatically based on your data if you ask it to. This makes scikit-learn a useful starting point for understanding what's actually happening inside a nearest neighbor search before reaching for a specialized library.

## When to reach for scikit-learn

Scikit-learn NearestNeighbors is the right choice when:

- You're already using scikit-learn and want to avoid adding dependencies
- Your dataset is small enough that brute force is fast enough - typically under 50,000 vectors at 384 dimensions
- You want exact results rather than approximate and FAISS feels like overkill
- You're teaching or learning nearest neighbor search and want to experiment with different algorithms side by side

## A practical note before we start

On Apple Silicon you may see divide by zero and overflow warnings from the underlying matrix multiplication. In our runs they came from numpy and appeared even with the embeddings cast to float64. The notebook suppresses them and checks that the similarities are finite and that each query matches itself. Note that float64 doubles the data each search scans, so scikit-learn's latencies aren't strictly like-for-like with the float32 libraries in Days 1 and 2.

## What we found

We evaluated all three algorithms on a 50,000-record subset of the Nordvik dataset. We didn't run the full 500,000 records: extrapolating the 100,000-record result linearly suggests about 150ms per query. Recall is scored against an exact ground truth computed on the same 50,000 records, using the 92 sampled queries from Day 1 that fall inside them.

The results confirmed what the theory predicts. Ball tree and kd-tree offered no meaningful advantage over brute force at 384 dimensions. All three algorithms took roughly 22 to 26ms per query - essentially identical. This is the curse of dimensionality in action: tree-based algorithms that are fast in low dimensions (under 20) degrade toward O(n) query time as dimensionality increases. At 384 dimensions they are just a more complicated way to do the same linear scan.

| Algorithm | Recall@10 | Latency (ms) | Dataset size |
|---|---|---|---|
| Brute force (cosine) | 1.000 | 24.39 | 50k |
| Ball tree (euclidean) | 1.000 | 25.78 | 50k |
| KD-tree (euclidean) | 1.000 | 25.91 | 50k |
| Voyager M=12, ef=200 | 0.980 | 0.29 | 500k |
| FAISS IndexHNSWFlat (reference, single query) | 1.000 | 0.06 | 500k |

All three algorithms scored a recall of 1.000, as exact search should. That's the meaningful recall result here: scikit-learn never misses a neighbor, it just takes a long time to find them. Two details matter if you score a subset yourself. The Day 1 ground truth covers all 500,000 records, and only about 10% of its neighbors fall inside our subset, so scoring against it caps recall near that share. And each index returns the query itself as its top result, so we ask for 11 neighbors and drop the query before scoring.

The scaling numbers are the most telling result. Brute force latency is flat at around 13.5ms from 10,000 to 25,000 records, then roughly doubles to 26ms at 50,000 and reaches 31ms at 100,000. We haven't isolated why the start is flat. A fixed per-query cost is one possible explanation. Meanwhile FAISS HNSW at 500,000 records takes 0.06ms, which is roughly 400 times faster than scikit-learn at 50,000 records and a tenth of the data.

## What scikit-learn does not do

Scikit-learn's NearestNeighbors has no dedicated index persistence (you can pickle a fitted estimator, but there's no save and load designed for neighbor indexes), no approximate search and no GPU support. Its brute force implementation is well-optimized with multi-core support via `n_jobs=-1`, but O(n) scaling makes it impractical above roughly 50,000 vectors at 384 dimensions for latency-sensitive applications.

There's also no built-in metadata filtering, no document storage and no path from scikit-learn to a production serving layer. It's a research and development tool, not a deployment target.

## The honest conclusion

Scikit-learn NearestNeighbors is the right answer to a specific question: "I need nearest neighbor search, I already have scikit-learn and my dataset is small." For anything larger or more demanding, the libraries in the remaining chapters are the better choice. The value of this chapter is not in the performance numbers - which are unimpressive - but in seeing clearly where scikit-learn's limits are and why they exist.

## When to look elsewhere

Scikit-learn is the wrong choice if:

- Your dataset has more than 50,000 vectors at high dimensionality - latency will be too high for interactive use
- You need approximate search to trade recall for speed - scikit-learn only does exact search
- You need index persistence - scikit-learn has no dedicated save and load for neighbor indexes
- You need the lowest possible latency - every other library in this book is faster at scale
