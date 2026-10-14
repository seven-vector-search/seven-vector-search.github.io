# Day 2: Voyager

Voyager is Spotify's open-source approximate nearest neighbor search library, built and used in production to power music and podcast recommendations across hundreds of millions of users. It uses the HNSW algorithm - the same one we saw in Day 1's `IndexHNSWFlat` - but wraps it in a cleaner API with better defaults and built-in persistence.

Spotify describes Voyager as their answer to the limitations they found with Annoy, their original recommendation library. The comparison with Annoy would have been a natural chapter pairing, but Annoy proved to be broken on Apple Silicon - it consistently returned only one neighbor regardless of K, which we confirmed with a minimal test on 1,000 random vectors. We dropped it from the book. Voyager, by contrast, installed and ran without issue.

## When to reach for Voyager

Voyager is the right choice when:

- You want HNSW search with a simpler API than FAISS and sensible defaults out of the box
- You need built-in index persistence without writing serialization code yourself
- You're building a Python or Java application and want index compatibility between both languages
- You want a library that is battle-tested in production at massive scale

## What we found

We built two Voyager indexes on the 500,000-record Nordvik dataset: one with Spotify's default of M=12 and one with M=32 to match the Day 1 FAISS configuration. Both were scored against the same 1,000 sampled queries and ground truth we used in Day 1, with recall and latency averaged over those queries.

M=12 reached a recall of 0.946 at ef=11 and climbed slowly to 0.980 at ef=200. Most of that gain arrived by ef=20 (0.964), so raising ef helps but with diminishing returns. M=32 started higher, at 0.990 at ef=11, and reached 0.997 at ef=200. The denser graph is worth about two to four percentage points of recall, at the price of a longer build: 51 seconds against 34 for M=12.

In Day 1, FAISS HNSW with M=32 achieved recall 1.0, so Voyager at M=32 lands very close. That Day 1 figure came from a single query though, so it isn't strictly comparable with these 1,000-query averages. Latency stayed between roughly 0.06 and 0.46ms across every setting. Differences between neighboring rows, such as M=12 at ef=11 being slower than at ef=20, are mostly timing noise.

One detail is worth knowing if you run your own evaluation. Voyager returns the query itself as its top result, while the Day 1 ground truth excludes it. If you score Voyager's top ten directly against that ground truth, each result set holds the query plus only nine true neighbors, so recall can never exceed 0.9. Our first attempt did exactly that and both indexes stalled near 0.89 whatever M and ef we used. The fix is to ask for 11 results and drop the query before scoring. Because we ask for 11 results, the smallest usable ef is 11.

| Config | Recall@10 | Latency (ms) |
|---|---|---|
| Voyager M=12, ef=11 | 0.946 | 0.157 |
| Voyager M=12, ef=200 | 0.980 | 0.290 |
| Voyager M=32, ef=11 | 0.990 | 0.093 |
| Voyager M=32, ef=200 | 0.997 | 0.458 |
| FAISS IndexHNSWFlat (reference, single query) | 1.000 | 0.060 |

## Persistence

One of Voyager's practical advantages over FAISS is built-in index persistence. Where FAISS requires `faiss.write_index` and careful file management, Voyager handles this with a single `index.save()` call. The index file for 500,000 vectors at M=12 came out at 788MB. Loading back from disk with `Index.load()` is equally straightforward and returned identical results for our sample query.

## Dynamic insertion

Voyager supports adding new vectors to an existing index without rebuilding it. We demonstrated this by adding 1,000 new vectors to the built index in about 0.1 seconds. They were searchable straight away with no rebuild: when we queried with each new vector, about 98% were found in their own top ten. That's a meaningful advantage in production systems where new data arrive continuously.

## What Voyager does not do

Voyager is an in-memory library. The entire index must fit in RAM. At 788MB for 500,000 vectors at M=12, extrapolating linearly would make this a constraint at tens of millions of records on typical server hardware (we didn't measure beyond 500,000).

Like FAISS, Voyager has no built-in metadata filtering. If you need to constrain search results by crime type or neighborhood, you must filter the results after retrieval or move to Chroma (Day 6) or LanceDB (Day 7).

## When to look elsewhere

Voyager is the wrong choice if:

- You need exact search - Voyager is approximate only
- You need metadata filtering - look at Chroma (Day 6) or LanceDB (Day 7)
- Your dataset doesn't fit in RAM - look at LanceDB (Day 7)
- You want a purely static index with the smallest possible memory footprint and don't need dynamic insertion
