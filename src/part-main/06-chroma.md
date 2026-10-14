# Day 6: Chroma

Chroma is an open-source AI-native vector database designed for building LLM applications. Unlike every library we've used so far, Chroma isn't just a search index - it's a persistent, queryable store with built-in metadata filtering, document storage and collection management. It's the bridge between a vector search library and a full vector database.

Chroma runs embedded in your Python process with no server required, which is what makes it relevant in a book about Python vector search libraries. In embedded mode it feels like a library - you import it, create a collection and query it. But underneath it manages persistence, indexing and metadata automatically in a way that none of the previous libraries do.

## When to reach for Chroma

Chroma is the right choice when:

- You're building a RAG pipeline and need a persistent store that survives process restarts
- You need metadata filtering alongside vector search - filter by crime type, neighborhood or weapon alongside vector search
- You want to store documents and embeddings together rather than managing them separately
- You want a simple local development experience, with client-server and hosted options if you outgrow it (we didn't test those)

## What we found

We loaded all 500,000 Nordvik records into Chroma in batches of 5,000, storing the MO_TEXT as the document and including crime type, crime subtype, neighborhood, datetime, weapon and property stolen as metadata fields. The collection persisted automatically to disk with no explicit save call. Loading took 321.5 and 414.8 seconds in our two runs, slower than any index build in the earlier chapters, although it also includes storing the documents and metadata, so it isn't a like-for-like comparison.

Vector search recall was 0.9967 at 0.532ms, averaged over the same 1,000 sampled queries and Day 1 ground truth we used for the other libraries. As before, we asked for 11 results and removed the query's own ID before scoring. The query itself came back first for 997 of the 1,000 queries, and we didn't investigate the other three. That recall is in the same range as USearch at ef=16 and above Voyager, but Chroma is roughly an order of magnitude slower than FAISS HNSW and PyNNDescent for pure vector search, and slower than Voyager and USearch at low ef settings. We didn't measure where the extra time goes.

| Config | Recall@10 | Latency (ms) | Metadata filtering |
|---|---|---|---|
| Chroma (cosine, embedded) | 0.9967 | 0.532 | Yes |
| FAISS IndexHNSWFlat (reference) | 1.000 | 0.060 | No |
| PyNNDescent n_neighbors=50 (reference) | 0.9988 | 0.056 | No |
| Voyager M=12, ef=200 (reference) | 0.9797 | 0.290 | No |
| USearch HNSW ef=16 (reference) | 0.9945 | 0.157 | No |

## Metadata filtering

This is where Chroma earns its place. The previous five libraries gave us no metadata filtering in our examples, so we'd have had to filter the results ourselves. We ran three filtered queries against the same burglary incident, NVK-000004, and checked that every result satisfied its filter, for these three examples and for 50 further queries per filter:

**Burglary only.** Restricting results to `CrimeType = "Burglary"` returned ten burglaries, all from Hartley Cross. In the unfiltered search rank 10 was an incident of type Other (NVK-331119, distance 0.1615). With the filter that slot went to another burglary (NVK-025362, distance 0.1638). In both searches rank 1 is the query incident itself.

**Armed incidents only.** Filtering to `Weapon != "none"` returned seven burglaries, two robberies and one Other incident, all carrying weapons. The filter crossed crime types, finding similar incidents that shared the weapon characteristic.

**Millfield only.** Restricting to a single neighborhood produced distances from 0.3428 to 0.3702, compared with 0.1416 for the nearest unfiltered neighbor (excluding the query itself). This is expected and informative: when you constrain the candidate set geographically, the closest semantic matches within that area are further from the query than the global nearest neighbors. The filter doesn't degrade the search - it answers a different question.

**Hybrid search.** We also combined a metadata filter with a keyword condition on the document text: burglaries whose MO_TEXT contains `1632hrs`, a word taken from the query incident itself. Ten results came back, all satisfying both conditions. Distances beyond the query itself ranged from 0.3376 to 0.4429, larger than in the plain Burglary filter because the keyword narrows the candidates further.

**What filtering costs.** Filtering wasn't cheap. Averaged over 200 queries, an unfiltered search took 0.575ms. Filtering to Burglary took 328.1ms, armed incidents 426.4ms and Millfield 126.4ms, roughly 220 to 740 times slower. Every filtered result was correct, so the cost shows up in latency, not accuracy. The three filters match very different shares of the records: about 7% for Millfield, 35% for Burglary and about 40% for armed incidents. Latency rose in the same order, which is consistent with cost tracking the number of matching records, although we didn't test that directly. If you plan to filter at this scale, measure it on your own data.

**Is the filter applied during the search?** To check, we picked the rarest combination of crime type and neighborhood with at least 20 records (Other incidents in Northgate, 941 of the 500,000 records) and asked for ten results. Chroma returned all ten, all matching, even though none of those records appeared in the unfiltered top 100 for that query. So the filter is applied during the search rather than to the top results afterward.

## A simple RAG pipeline

We built a minimal retrieval function that fetches the five most similar incidents to a query, with an optional crime type filter. It excludes the query incident itself so it doesn't appear in its own context. This is a common pattern in LLM applications: embed the query, retrieve relevant context, pass it to the language model. We stopped short of the model call. Chroma's document storage means the retrieved context is immediately available without a second lookup against the original dataset.

With the Burglary filter the function took 384.9ms on average over 100 queries. That's close to the cost of the filtered search itself, so the filter rather than the formatting is the likely cause, but we didn't time the two separately. Whether that's acceptable depends on your application.

## Persistence

Chroma's `PersistentClient` writes to a local directory automatically. We verified this by creating a second client pointing to the same path, in the same process rather than after a full restart, and confirming the collection loaded with all 500,000 records. Fifty queries returned identical results from both collections. For a production application this means you build the collection once and can reopen it later without any additional code, though we didn't test a full restart. The directory took 3,639 MiB on disk, about five times the 732 MiB of raw embeddings, because it also holds the documents, metadata and index. We didn't break the total down.

## A note on the API

`"ids"` isn't a valid value for the `include` parameter of `collection.query()`. IDs are always returned and don't need to be requested explicitly. Use `include=[]` to return only IDs or `include=["documents", "metadatas", "distances"]` for the full result set.

## What Chroma does not do

Chroma in embedded mode is a single-process store. It's not designed for concurrent writes from multiple processes. If you need that, Chroma offers a client-server mode with `chromadb.HttpClient`. We didn't test it, but only the client initialization should need to change, not your query logic.

In our tests Chroma's HNSW search reached high recall but was slower than the other libraries' approximate indexes, and filtered search was far slower (126 to 426ms). For pure vector search without filtering, FAISS and PyNNDescent are significantly faster. If you need metadata filtering and document storage, the extra cost may be worth paying. If you don't, an earlier library in this book will serve you faster.

## When to look elsewhere

Chroma is the wrong choice if:

- You need the lowest possible query latency - FAISS and PyNNDescent are faster for pure vector search
- Your dataset is larger than available RAM - we didn't test this case, but our collection took 3,639 MiB on disk, so plan for memory use before scaling up; look at LanceDB (Day 7)
- You need fast filtered search at this scale - our filtered queries took 126 to 426ms
- You need concurrent writes from multiple processes in embedded mode - use client-server mode instead
- You need SQL-style joins between your vector store and relational data - LanceDB (Day 7) has better structured data support
