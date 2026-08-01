# Introduction

Vector search is one of those ideas that sounds complicated until you see it working. You take a piece of text, convert it to a list of numbers and then find other pieces of text whose numbers are similar. That's it. The Python ecosystem has produced a remarkable variety of tools for doing this efficiently and choosing between them is not always obvious.

This book is a practical guide to seven of those tools. we'll use the same dataset throughout, ask the same questions of each library and measure the results consistently so you can make informed choices rather than guessing. By the end you'll know not just how to use each library but why you would reach for one over another given your specific constraints.

## What this book is

Each chapter introduces one library, builds an index over a 500,000-record crime incident dataset, runs similarity searches, evaluates recall and latency against a consistent ground truth and discusses honestly what the library is good at and where it falls short. The chapters are designed to be read in order but work as standalone references once you know the context.

This is not a survey of vector search theory. We won't derive the mathematics of HNSW graphs or explain how product quantization works from first principles. There are excellent papers and textbooks for that. What we'll offer here is the practitioner's perspective: what does this library actually do when you run it, what are the gotchas and when should you use something else?

## Who this book is for

This book is for Python developers who need vector search to work and want to understand their options before committing to one. You might be building a RAG pipeline and wondering whether you need a database or whether a library will do. You might be hitting performance limits with your current approach and wondering if there is a faster option. You might simply be curious about what the ecosystem looks like right now.

We'll assume comfort with Python and pandas. We'll use Jupyter notebooks throughout.

## The companion book

This book sits alongside *[Seven Vector Databases in Seven Days](https://seven-vector-databases.github.io/)*, which covers hosted and embedded vector databases: PostgreSQL with pgvector, MongoDB Atlas, Pinecone, Weaviate, Neo4j, Snowflake and Databricks. The two books address different questions. The databases book asks: what infrastructure should I run? This book asks: what Python library should I call?

If your dataset fits in memory and you want to stay in process, this book is your starting point. If you need a persistent, queryable service with replication and access control, the databases book is where to look next. Many production systems use both: a library for rapid prototyping and offline analysis, a database for the serving layer.

## Nordvik

Every chapter in this book uses the same dataset: 500,000 synthetic crime incident reports set in Nordvik, a fictional city on Bouvet Island. Bouvet Island is a Norwegian dependency in the South Atlantic and holds the distinction of being one of the most remote uninhabited islands in the world. Its coordinates are real and spatially coherent, which means our maps and geographic queries work correctly, but nothing we'll generate implicates any real street or person.

The dataset was generated using a template engine rather than a language model. Each record has a structured set of fields - crime type, suspect description, entry method, property stolen, weapon, neighborhood and timestamp - and a free-text `MO_TEXT` field that reads like an officer's incident report. We chose this approach because it's fast, reproducible and requires no external API. The trade-off is that the `MO_TEXT` has more template structure than real police data would, which affects how the embedding model clusters results. We'll note this where it matters.

## How to use this book

The notebooks are on GitHub alongside the book. To follow along you'll need Python 3.12 or later, Jupyter and a machine with at least 16GB of RAM. Each chapter installs its own dependencies via a pip cell at the top.

Run `generate_dataset.ipynb` first to create the Nordvik dataset, then `generate_embeddings.ipynb` to compute and save the embeddings. Every subsequent chapter loads the same two files from disk. You only need to run the generators once.

If you want to skip straight to a specific library, each chapter notebook is self-contained once the data files exist.
