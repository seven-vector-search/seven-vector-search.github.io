# Introduction

Vector search is one of those ideas that sounds complicated until you see it working. You take a piece of text, convert it to a list of numbers and then find other pieces of text whose numbers are similar. That's it. The Python ecosystem has produced a remarkable variety of tools for doing this efficiently and choosing between them isn't always obvious.

This book is a practical guide to seven of those tools. We'll use the same dataset throughout, ask the same questions of each library and measure the results consistently so you can make informed choices rather than guessing. By the end you'll know not just how to use each library but why you would reach for one over another given your specific constraints.

## What this book is about

Each chapter introduces one library, builds an index over a 500,000-record crime incident dataset, runs similarity searches, evaluates recall and latency against a consistent ground truth and discusses honestly what the library is good at and where it falls short. The chapters are designed to be read in order but work as standalone references once you know the context.

Two details are worth knowing about. Scikit-learn runs on a 50,000-record subset, with a ground truth restricted to that subset, and Day 1 measures FAISS's recall from a single query, while every other library is averaged over 1,000. The last two chapters, Chroma and LanceDB, are embedded vector databases rather than pure libraries. We include them because they run inside your Python process.

This isn't a survey of vector search theory. We won't derive the mathematics of HNSW graphs or explain how product quantization works from first principles. There are excellent papers and textbooks for that. What we'll offer here is the practitioner's perspective: what does this library actually do when you run it, what are the gotchas and when should you use something else?

## How we measure

Every chapter uses the same 384-dimension embeddings, computed once from the `MO_TEXT` field with `all-MiniLM-L6-v2` and normalized to unit length, so cosine and L2 distance rank neighbors the same way. Every chapter also measures the same two things. Recall@10 is the share of the true ten nearest neighbors that an index returns, averaged over 1,000 sampled queries. The true neighbors come from an exact FAISS search in Day 1, which excludes the query itself. Most indexes return the query as their own first result, so each later chapter asks for 11 results and removes the query's own ID before scoring. Without that step recall can't exceed 0.9. Latency is the average time for one query, in milliseconds. These timings come from one machine, so treat them as a guide to relative order rather than precise figures.

## Who this book is for

This book is for Python developers who need vector search to work and want to understand their options before committing to one. You might be building a RAG pipeline and wondering whether you need a database or whether a library will do. You might be hitting performance limits with your current approach and wondering if there's a faster option. You might simply be curious about what the ecosystem looks like right now.

We'll assume comfort with Python and pandas. We'll use Jupyter notebooks throughout.

## The companion book

This book sits alongside *[Seven Vector Databases in Seven Days](https://seven-vector-databases.github.io/)*, which covers vector databases: PostgreSQL with pgvector, which we run locally, and six cloud services, MongoDB Atlas, Pinecone, Weaviate, Neo4j, Snowflake and Databricks. The two books address different questions. The databases book asks: what infrastructure should I run? This book asks: what Python library should I call?

If your dataset fits in memory and you want to stay in process, this book is your starting point. If you need a persistent, queryable service with replication and access control, the databases book is where to look next. Many production systems use both: a library for rapid prototyping and offline analysis, a database for the serving layer.

## Nordvik

Every chapter in this book uses the same dataset: 500,000 synthetic crime incident reports set in Nordvik, a fictional city on Bouvet Island. Bouvet Island is a Norwegian dependency in the South Atlantic and holds the distinction of being one of the most remote uninhabited islands in the world. Its coordinates are real and spatially coherent, so the incident map in the dataset notebook is geographically meaningful, but nothing we'll generate implicates any real street or person.

The dataset was generated using a template engine rather than a language model. Each record has a structured set of fields - crime type and subtype, neighborhood, property type, coordinates, timestamp, suspect description, suspect vehicle, victim details, entry method, weapon and property stolen - and a free-text `MO_TEXT` field that reads like an officer's incident report and is assembled from them using templates. We chose this approach because it's fast, reproducible and requires no external API. The trade-off is that the `MO_TEXT` has more template structure than real police data would: each crime type has ten narrative templates, which affects how the embedding model clusters results. We'll note this where it matters.

Some fields are also sampled independently of one another. Weapons, for example, are assigned to 40% of incidents whatever the crime type, so a burglary or a drug offence can carry a firearm. That's unrealistic, but it means a weapon filter cuts across every crime type.

## How to use this book

The notebooks are on GitHub alongside the book. To follow along you'll need Python 3.12, Jupyter and a machine with at least 16GB of RAM. Each chapter installs its own dependencies, pinned to exact versions, via a pip cell at the top.

Run `generate_dataset.ipynb` first to create the Nordvik dataset, then `generate_embeddings.ipynb` to compute and save the embeddings. Then run the Day 1 notebook, which computes and saves the ground truth that every later chapter scores against. You only need to run these three once.

If you want to skip straight to a specific library, each chapter notebook is self-contained once the dataset, embeddings and Day 1 ground truth files exist.
