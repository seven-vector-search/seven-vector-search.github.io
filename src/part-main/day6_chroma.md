# Day 6: Chroma

Chroma is an open-source AI-native vector database designed for building LLM applications. Unlike every library we have used so far, Chroma is not just a search index - it's a persistent, queryable store with built-in metadata filtering, document storage and collection management. It's the bridge between a vector search library and a full vector database.

Chroma runs embedded in your Python process with no server required, which is what makes it relevant in a book about Python vector search libraries. In embedded mode it feels like a library - you import it, create a collection and query it. But underneath it manages persistence, indexing and metadata automatically in a way that none of the previous libraries do.

## When to reach for Chroma

Chroma is the right choice when:

- You are building a RAG pipeline and need a persistent store that survives process restarts
- You need metadata filtering alongside vector search - filter by crime type, neighborhood or date before or during search
- You want to store documents and embeddings together rather than managing them separately
- You want a simple local development experience that scales to a hosted service without code changes

## What we found

We loaded all 500,000 Nordvik records into Chroma in batches of 5,000, storing the MO_TEXT as the document and including crime type, crime subtype, neighborhood, datetime and weapon as metadata fields. The collection persisted automatically to disk - no explicit save call required.

Basic vector search produced recall of 0.899 at 0.776ms. That is consistent recall-wise with the other approximate libraries but about ten times slower than PyNNDescent for pure vector search. The overhead comes from Chroma's persistence and metadata layers, which do real work on every query.

| Config | Recall@10 | Latency (ms) | Metadata filtering |
|---|---|---|---|
| Chroma (cosine, embedded) | 0.899 | 0.776 | Yes |
| FAISS IndexHNSWFlat (reference) | 1.000 | 0.060 | No |
| PyNNDescent (reference) | 0.900 | 0.043 | No |

## Metadata filtering

This is where Chroma earns its place. None of the previous five libraries can do what we demonstrate in this chapter. We ran three filtered queries against the same burglary incident:

**Burglary only.** Restricting results to `CrimeType = "Burglary"` returned ten burglaries from the Hartley Cross area. Without the filter, rank 10 was occasionally a different crime type at similar distance. The filter enforces the constraint directly in the search rather than requiring post-processing.

**Armed incidents only.** Filtering to `Weapon != "none"` returned results across multiple crime types - burglaries, robberies and one Other incident - all carrying weapons. The filter crossed categorical boundaries correctly, finding semantically similar incidents that shared the weapon characteristic regardless of crime type.

**Millfield only.** Restricting to a single neighborhood produced distances of 0.34 and above, compared to 0.14 and above for the unfiltered results. This is expected and informative: when you constrain the candidate set geographically, the closest semantic matches within that area are necessarily further from the query than the global nearest neighbors. The filter does not degrade the search - it answers a different question.

## A simple RAG pipeline

We built a minimal retrieval function that fetches the five most similar incidents to a query, with an optional crime type filter. This is the pattern that most LLM applications use in production: embed the query, retrieve relevant context, pass it to the language model. Chroma's document storage means the retrieved context is immediately available without a second lookup against the original dataset.

The function takes around 1ms to retrieve five filtered results - fast enough for interactive use in most applications.

## Persistence

Chroma's `PersistentClient` writes to a local directory automatically. We verified this by creating a fresh client pointing to the same path and confirming the collection loaded correctly with all 500,000 records intact. Query results were identical to the original session. For a production application this means you build the index once and serve from it across restarts without any additional code.

## A note on the API

In Chroma 1.5.9, `"ids"` is not a valid value for the `include` parameter in `collection.query()`. IDs are always returned by default and do not need to be requested explicitly. Passing `include=["ids"]` raises a `ValueError`. Use `include=[]` to return only IDs, or `include=["documents", "metadatas", "distances"]` for the full result set.

## What Chroma does not do

Chroma in embedded mode is a single-process store. It's not designed for concurrent writes from multiple processes. If you need that, Chroma offers a client-server mode with `chromadb.HttpClient` that requires no changes to your query logic - only to the client initialization.

Chroma's HNSW implementation is solid but not the fastest available. For pure vector search without filtering, PyNNDescent and FAISS are significantly faster. The metadata filtering and document storage are what justify the additional overhead. If you do not need those features, an earlier library in this book will serve you faster.

## When to look elsewhere

Chroma is the wrong choice if:

- You need the lowest possible query latency - FAISS and PyNNDescent are faster for pure vector search
- Your dataset is larger than available RAM - Chroma loads indexes into memory; look at LanceDB (Day 7)
- You need concurrent writes from multiple processes in embedded mode - use client-server mode instead
- You need SQL-style joins between your vector store and relational data - LanceDB (Day 7) has better structured data support
