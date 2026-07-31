# Day 5: USearch

USearch is a fast, compact vector search library developed by Unum Cloud. Like FAISS, it implements the HNSW algorithm - but that's where the similarity ends. USearch is designed around a different set of priorities: a minimal binary footprint, SIMD-optimized distance computations, support for user-defined metrics and compatibility across a wide range of platforms including iOS, Android and WebAssembly.

Where FAISS is a research-oriented library with a large C++ codebase, USearch is engineered for deployment. The entire library fits in a single header file. Its documentation claims exact brute-force search up to 20 times faster than FAISS's `IndexFlatL2` on the same hardware, thanks to SIMD-optimized similarity kernels. We'll put that claim to the test.

## When to reach for USearch

USearch is the right choice when:

- You need the smallest possible binary footprint - important for mobile, edge or embedded deployments
- You want SIMD-accelerated exact search on Intel or AMD hardware
- You need custom distance metrics - USearch supports user-defined Python functions via Numba
- You want a library that works identically across Python, JavaScript, Rust, Java and C without a separate server

## What we found

We built a USearch HNSW index with connectivity=32 and expansion_add=200 on the full 500,000-record dataset. Build time was 77 seconds - slower than both FAISS and PyNNDescent for equivalent quality. The index file came out at 864MB, larger than Voyager's 788MB at the same connectivity setting.

HNSW query recall settled at 0.897 across all `expansion_search` settings above 16, consistent with the distance metric ceiling we've seen throughout the book.

| Config | Recall@10 | Latency (ms) |
|---|---|---|
| HNSW conn=32, ef=16 | 0.897 | 0.143 |
| HNSW conn=32, ef=64 | 0.897 | 0.260 |
| HNSW conn=32, ef=256 | 0.900 | 0.887 |
| Exact search | 1.000 | 77.3 |

## The exact search result

The most significant finding in this chapter is what happened when we enabled `exact=True`. USearch's exact brute-force mode is supposed to be dramatically faster than FAISS's `IndexFlatL2` on modern hardware. On our MacBook Air M4 it was 5 times slower -- 77ms compared to FAISS's 14ms.

This is not a USearch bug. It's a platform mismatch. USearch's SIMD optimizations target Intel AVX-512 instructions, which are available on Intel Sapphire Rapids and similar x86 server processors. Apple Silicon uses ARM NEON and SVE, a different SIMD architecture, for which USearch does not have the same level of optimization. The 20x speedup claim is real on the hardware it was designed for. On an M4 Mac the picture is reversed.

This is a useful lesson about benchmarks in general: performance claims from library documentation are measured on specific hardware. Always benchmark on your target platform before committing to a library based on headline numbers.

## Custom metrics

One of USearch's most distinctive features is support for user-defined distance metrics via Numba. You write a Python function decorated with `@numba.cfunc`, compile it to a `CompiledMetric` object and pass it to the `Index` constructor. USearch uses it for both index construction and search.

We demonstrated a weighted cosine metric that upweights the first 64 dimensions of each vector - a toy example of the kind of domain-specific tuning this enables. The custom metric index built in 4.4 seconds on 50,000 vectors and produced different top results than the standard cosine index, confirming that the metric was applied.

Note that the API for custom metrics changed in USearch 2.26.0. `CompiledMetric` and `MetricSignature` moved from `usearch.compiled` to `usearch.index`. If you're adapting older examples you'll need to update your imports.

## Persistence

USearch index persistence uses the same save and load pattern as Voyager. `index.save(path)` writes the index to disk; loading requires creating an empty `Index` with matching parameters first, then calling `index.load(path)` on the instance. This changed between versions - earlier examples showing `Index.load(path)` as a class method will fail in 2.26.0.

## What USearch does not do

USearch has no metadata filtering and no document storage. Like FAISS and Voyager, it's a pure vector index. It also has no dynamic insertion in the same sense as Voyager - you can add vectors but the process is less fluid than Voyager's API suggests.

The exact search advantage is hardware-specific. On Apple Silicon it offers no performance benefit over FAISS flat search.

## When to look elsewhere

USearch is the wrong choice if:

- You are on Apple Silicon and expect the exact search speedup - it will not materialize
- You need metadata filtering - look at Chroma (Day 6) or LanceDB (Day 7)
- Your dataset does not fit in RAM - look at LanceDB (Day 7)
- You want automatic parameter tuning - USearch requires the same manual tuning as FAISS HNSW
