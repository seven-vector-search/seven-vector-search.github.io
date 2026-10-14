# Day 5: USearch

USearch is a fast, compact vector search library developed by Unum Cloud. Like FAISS, it implements the HNSW algorithm - but it's designed around a different set of priorities: a minimal binary footprint, SIMD-optimized distance computations, support for user-defined metrics and compatibility across a wide range of platforms including iOS, Android and WebAssembly.

Where FAISS is a research-oriented library with a large C++ codebase, USearch is engineered for deployment. The core C++ implementation is a single header file. Its README reports exact brute-force search up to 20 times faster than FAISS's `IndexFlatL2` in a Google Colab test, thanks to SIMD-optimized similarity kernels. We'll put that claim to the test.

## When to reach for USearch

USearch is the right choice when:

- You need the smallest possible binary footprint - important for mobile, edge or embedded deployments
- You want both approximate and exact search from one library
- You need custom distance metrics - USearch supports user-defined Python functions via Numba
- You want a library that works identically across Python, JavaScript, Rust, Java and C without a separate server

## What we found

We built a USearch HNSW index with connectivity=32 and expansion_add=200 on the full 500,000-record dataset and scored it against the same 1,000 sampled queries and Day 1 ground truth we used for the other libraries. Build time was about 80 seconds, slower than Voyager (34 to 51 seconds) and PyNNDescent (6 to 29 seconds) in our runs. The saved index came out at 864MB (the library reported 1.27GB in memory), against 788MB for Voyager at M=12. These settings aren't equivalent, so treat the sizes as a rough guide.

HNSW recall was 0.9945 at expansion_search=16 and 0.9996 at 64, with little to gain beyond that (0.9997 at 256). Latency roughly doubled with each doubling of expansion_search, from 0.16ms to 0.91ms, and repeated almost exactly between two runs. That's higher recall than Voyager or PyNNDescent reached, at higher latency than PyNNDescent (0.03 to 0.06ms) and FAISS HNSW (0.06ms from a single query), and in the same range as Voyager (0.06 to 0.46ms).

| Config | Recall@10 | Latency (ms) |
|---|---|---|
| HNSW conn=32, ef=16 | 0.9945 | 0.157 |
| HNSW conn=32, ef=64 | 0.9996 | 0.335 |
| HNSW conn=32, ef=256 | 0.9997 | 0.911 |
| Exact search, single query | 0.9999 | 87.6 |
| Exact search, batch of 1,000 (per query) | 0.9999 | 14.5 |
| FAISS IndexFlatL2 (reference, single query) | 1.0000 | 15.75 |

## The exact search result

The most significant finding in this chapter is what happened when we enabled `exact=True`. USearch's README reports exact brute-force search up to 20 times faster than FAISS's `IndexFlatL2` in a Google Colab test. On our MacBook Air M4 a single exact query took 88ms, about 5.6 times the 15.75ms we measured for FAISS flat in Day 1.

The gap closes substantially when queries are batched. Sending all 1,000 queries in one call, USearch exact search took 14.5ms per query, close to FAISS flat's single-query figure. That suggests much of the single-query gap is a threading effect, since a batch can spread queries across cores. We didn't run FAISS in batch mode, so we can't say which is faster there. The 20x claim didn't reproduce in either mode, and we haven't established why. The comparison also isn't like-for-like, because USearch computed cosine distance while FAISS computed L2. Exact search scored a recall of 0.9999 against the L2 ground truth, so the metric difference itself is negligible.

This is a useful lesson about benchmarks in general: headline numbers are measured on specific hardware and workloads. Always benchmark on your target platform and with your own query pattern before committing to a library.

## Custom metrics

One of USearch's most distinctive features is support for user-defined distance metrics via Numba. You write a Python function decorated with `@numba.cfunc`, compile it to a `CompiledMetric` object and pass it to the `Index` constructor. USearch uses it for both index construction and search.

We demonstrated a weighted cosine metric that upweights the first 64 dimensions of each vector - a toy example of the kind of domain-specific tuning this enables. Those dimensions have no special meaning in our embeddings. To check the metric was really applied, we compared it with a standard cosine index built on the same 50,000 vectors with the same settings. On average 9.7 of the 10 results were the same (including the query itself), and the distances differed: for our sample query, rank 2 was 0.1629 under the custom metric and 0.1638 under standard cosine. Some of the difference in results could come from HNSW's approximation, but the changed distances confirm the metric was used. The custom index took 6.9 seconds to build against 5.3 for the standard one.

Custom metric APIs have changed between releases, so if you adapt older examples, check where `CompiledMetric` and `MetricSignature` are imported from.

## Persistence

USearch persistence starts with `index.save(path)`, which wrote our 500,000-vector index to an 864MB file in 0.6 seconds. There are two ways to load it. You can create an empty `Index` with matching parameters and call `index.load(path)`, or call the class method `Index.restore(path)`, which reads the parameters from the file. Restoring took 0.3 seconds. Passing `view=True` memory-maps the file instead of loading it, which took 0.03 seconds, and both returned the same results for our sample query.

## What USearch does not do

USearch has no document storage and, in its Python API, no filtered search. The README lists predicate-based filtering for C++, C, Rust and Swift but not Python. Like FAISS and Voyager, it's a pure vector index. From Python, if you need to constrain results by crime type or neighborhood you must filter after retrieval or move to Chroma (Day 6) or LanceDB (Day 7).

Unlike PyNNDescent, USearch copes well with changing data. Adding 1,000 vectors to the 500,000-vector index took 0.27 seconds (about 0.1 for Voyager in Day 2), all 1,000 were found in their own top ten, and removing them took effectively no time and left none in the results. Exact search didn't beat FAISS flat in our tests: 5.6 times slower for single queries and about level in batches.

## When to look elsewhere

USearch is the wrong choice if:

- You expect the headline exact-search speedup on single queries - we didn't see it on an M4 Mac, and batching only brought it level with FAISS flat
- You need metadata filtering - look at Chroma (Day 6) or LanceDB (Day 7)
- Your dataset doesn't fit in RAM - look at LanceDB (Day 7)
- You want automatic parameter tuning - USearch requires the same manual tuning as FAISS HNSW
