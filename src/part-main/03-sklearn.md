# Day 3: Scikit-learn NearestNeighbors

Scikit-learn's `NearestNeighbors` is the vector search library you already have. If you have scikit-learn installed - and most Python data science and machine learning projects do - you have a capable nearest neighbor search implementation ready to use without any additional dependencies.

Unlike FAISS and Voyager which implement HNSW graph-based search, scikit-learn offers three distinct algorithms: brute force, ball tree and kd-tree. Each has different characteristics and the library selects the most appropriate one automatically based on your data if you ask it to. This makes scikit-learn a useful starting point for understanding what is actually happening inside a nearest neighbor search before reaching for a specialized library.

## When to reach for scikit-learn

Scikit-learn NearestNeighbors is the right choice when:

- You are already using scikit-learn and want to avoid adding dependencies
- Your dataset is small enough that brute force is fast enough - typically under 50,000 vectors at 384 dimensions
- You want exact results rather than approximate and FAISS feels like overkill
- You are teaching or learning nearest neighbor search and want to experiment with different algorithms side by side

## A practical note before we start

Scikit-learn's cosine distance computation has numerical issues with float32 embeddings at high dimensionality - you'll see divide by zero and overflow warnings from the underlying `matmul`. The fix is to cast embeddings to float64 before fitting. All timing loops also need to be wrapped in `warnings.catch_warnings()` to suppress the noise. We encountered this during development and the notebook handles it, but it's worth knowing about if you adapt the code.

## What we found

We evaluated all three algorithms on a 50,000-record subset of the Nordvik dataset. Using the full 500,000 records was not practical - even brute force search becomes prohibitively slow at that scale for a library that is not designed for it.

The results confirmed what the theory predicts. Ball tree and kd-tree offered no meaningful advantage over brute force at 384 dimensions. All three algorithms took around 25 to 27ms per query - essentially identical. This is the curse of dimensionality in action: tree-based algorithms that are fast in low dimensions (under 20) degrade toward O(n) query time as dimensionality increases. At 384 dimensions they are just a more complicated way to do the same linear scan.

| Algorithm | Recall@10 | Latency (ms) | Dataset size |
|---|---|---|---|
| Brute force (cosine) | 0.158 | 17.53 | 50k |
| Ball tree (euclidean) | 0.158 | 25.84 | 50k |
| FAISS IndexHNSWFlat (reference) | 1.00 | 0.06 | 500k |

The recall figure of 0.158 looks alarming but is explained by the same distance metric mismatch we discussed in the dataset chapter. The ground truth was computed with FAISS using L2 distance; scikit-learn uses cosine distance. The two algorithms agree with each other perfectly - both return exactly the same results - which confirms scikit-learn is internally consistent. The discrepancy is with the FAISS ground truth, not within scikit-learn itself.

The scaling numbers are the most telling result. Brute force latency is essentially flat from 10,000 to 25,000 records - the M4's parallelization via `n_jobs=-1` saturates around there - then grows to 30ms at 100,000 records. Meanwhile FAISS HNSW at 500,000 records takes 0.06ms. That is a 450x difference at ten times the data volume.

## What scikit-learn does not do

Scikit-learn's NearestNeighbors has no index persistence, no approximate search and no GPU support. Its brute force implementation is well-optimized with multi-core support via `n_jobs=-1`, but O(n) scaling makes it impractical above roughly 50,000 vectors at 384 dimensions for latency-sensitive applications.

There is also no metadata filtering, no document storage and no path from scikit-learn to a production serving layer. It's a research and development tool, not a deployment target.

## The honest conclusion

Scikit-learn NearestNeighbors is the right answer to a specific question: "I need nearest neighbor search, I already have scikit-learn and my dataset is small." For anything larger or more demanding, the libraries in the remaining chapters are the better choice. The value of this chapter is not in the performance numbers - which are unimpressive - but in seeing clearly where scikit-learn's limits are and why they exist.

## When to look elsewhere

Scikit-learn is the wrong choice if:

- Your dataset has more than 50,000 vectors at high dimensionality - latency will be too high for interactive use
- You need approximate search to trade recall for speed - scikit-learn only does exact search
- You need index persistence - scikit-learn has no built-in save and load for neighbor indexes
- You need the lowest possible latency - every other library in this book is faster at scale
