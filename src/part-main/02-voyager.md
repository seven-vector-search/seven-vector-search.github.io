# Day 2: Voyager

Voyager is Spotify's open-source approximate nearest neighbor search library, built and used in production to power music and podcast recommendations across hundreds of millions of users. It uses the HNSW algorithm - the same one we saw in Day 1's `IndexHNSWFlat` - but wraps it in a cleaner API with better defaults, built-in persistence and lower memory usage by default.

Spotify describes Voyager as their answer to the limitations they found with Annoy, their original recommendation library. The comparison with Annoy would have been a natural chapter pairing, but Annoy proved to be broken on Apple Silicon - it consistently returned only one neighbor regardless of K, which we confirmed with a minimal test on 1,000 random vectors. We dropped it from the book. Voyager, by contrast, installed and ran without issue.

## When to reach for Voyager

Voyager is the right choice when:

- You want HNSW search with a simpler API than FAISS and sensible defaults out of the box
- You need built-in index persistence without writing serialization code yourself
- You are building a Python or Java application and want index compatibility between both languages
- You want a library that is battle-tested in production at massive scale

## What we found

We built two Voyager indexes on the 500,000-record Nordvik dataset: one with Spotify's default of M=12 and one with M=32 to match the Day 1 FAISS configuration.

The M=12 results were surprising. Recall plateaued at around 0.89 regardless of how high we pushed `ef` - even at ef=200 we could not get above 0.888. In Day 1, FAISS HNSW with M=32 achieved recall 1.0. The natural conclusion is that M=12 caps the recall ceiling and that increasing ef alone cannot overcome a sparse graph.

So we rebuilt with M=32. The results were more surprising: recall barely moved. M=32 topped out at 0.896 compared to M=12's 0.888 - a difference of less than one percentage point despite the index taking 50 seconds to build rather than a few seconds.

The explanation is not a Voyager limitation. It's a measurement artifact. Our ground truth was computed with FAISS using L2 distance, while Voyager uses cosine distance. On normalized vectors these metrics are mathematically equivalent in theory, but floating point differences in implementation produce slightly different result orderings at the margin. The apparent recall ceiling of 0.89 is the size of that gap, not a genuine failure to find the right neighbors. Against a same-metric ground truth, Voyager at M=32 would match FAISS recall closely.

| Config | Recall@10 | Latency (ms) |
|---|---|---|
| Voyager M=12, ef=10 | 0.858 | 0.046 |
| Voyager M=12, ef=200 | 0.888 | 0.295 |
| Voyager M=32, ef=10 | 0.893 | 0.077 |
| Voyager M=32, ef=200 | 0.896 | 0.439 |
| FAISS IndexHNSWFlat (reference) | 1.000 | 0.060 |

## Persistence

One of Voyager's practical advantages over FAISS is built-in index persistence. Where FAISS requires `faiss.write_index` and careful file management, Voyager handles this with a single `index.save()` call. The index file for 500,000 vectors at M=12 came out at 788MB. Loading back from disk with `Index.load()` is equally straightforward and produces identical results.

## Dynamic insertion

Voyager supports adding new vectors to an existing index without rebuilding it. We demonstrated this by adding 1,000 new vectors to the built index - they were immediately queryable with no rebuild required. This is a meaningful advantage in production systems where new data arrives continuously.

## What Voyager does not do

Voyager is an in-memory library. The entire index must fit in RAM. At 788MB for 500,000 vectors at M=12 and scaling linearly from there, this becomes a constraint at tens of millions of records on typical server hardware.

Like FAISS, Voyager has no metadata filtering. If you need to constrain search results by crime type or neighborhood, you must filter the results after retrieval or move to Chroma (Day 6) or LanceDB (Day 7).

## When to look elsewhere

Voyager is the wrong choice if:

- You need exact search - Voyager is approximate only
- You need metadata filtering - look at Chroma (Day 6) or LanceDB (Day 7)
- Your dataset does not fit in RAM - look at LanceDB (Day 7)
- You want a purely static index with the smallest possible memory footprint and do not need dynamic insertion
