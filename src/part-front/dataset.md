# The Dataset

Before we can evaluate any vector search library we need something to search. This chapter covers how we built the Nordvik crime dataset and why we made the choices we did.

## Why synthetic data

Real crime data exists and some of it's publicly available. We looked at several sources including British Transport Police (BTP) incident records and Dallas Police Department reports. The BTP data lacked free-text narrative fields, which are essential for meaningful semantic search. The Dallas data was richer but comes with licensing friction that would make it awkward for readers to reproduce the experiments.

Generating synthetic data solves both problems. We control the schema, the volume and the distribution. Readers can reproduce everything without requesting access to anything. And we can tune the data to make the vector search experiments as informative as possible.

The trade-off is realism. Synthetic narratives generated from templates have more structural regularity than real officer-written reports. This affects how the embedding model clusters results - we see tighter clusters around template patterns than we would with genuine linguistic variety. We note this in the chapters where it matters most.

## Bouvet Island

The coordinates in the dataset are real. We use Bouvet Island as the geographic anchor for Nordvik, our fictional city. Bouvet Island is a Norwegian dependency in the South Atlantic, roughly 49 square kilometers in area and holds the distinction of being one of the most remote uninhabited islands in the world. If you drop any coordinate from our dataset into a mapping tool it will resolve to a glacier, not a street.

We define 15 fictional neighborhoods across the island's footprint and assign each a centroid coordinate. Individual incidents are placed within 300 meters of their neighborhood centroid using a small random offset. The result is a spatial distribution that looks like a real city and supports meaningful geographic queries and hotspot analysis.

## The MO_TEXT field

The central field for vector search is `MO_TEXT` - a free-text officer narrative describing how each crime was committed. A typical entry looks like this:

```
At approx 1632hrs, Multiple offenders described as medium build, tall, wearing face covering gained entry to Residence - Single Family in Hartley Cross. Property taken: handbag, tablet. No FO identified.
```

We generate these using a template engine with 30 templates per crime type, each drawing from randomized structured fields. The templates vary in structure - some lead with a timestamp, some with the location, some with the suspect description, some with police shorthand. We also vary the phrasing of individual components: `Property taken:` in one record becomes `Stolen:` or `Items removed include` in another.

The result is varied enough for the embedding model to produce meaningful similarity scores but structured enough that readers can see immediately whether a search result makes sense.

## Crime types and distribution

The dataset contains six crime types weighted to approximate a realistic urban distribution:

| Crime type | Share |
|---|---|
| Burglary | 35% |
| Vehicle Crime | 25% |
| Robbery | 20% |
| Assault | 10% |
| Drug Offences | 7% |
| Other | 3% |

Each crime type has its own set of subtypes, entry methods, property types and property stolen values. Burglary incidents are constrained to residential or commercial properties depending on subtype. Vehicle crime incidents use vehicle-specific entry methods. The structured fields are internally consistent even though the data are synthetic.

## Generating the dataset

The generation notebook `generate_dataset.ipynb` produces two output files:

- `nordvik_crimes.csv` - the full dataset in CSV format
- `nordvik_crimes.parquet` - the same data in Parquet format for faster loading

At 500,000 records the generation runs in seconds. There is no LLM involved, no API call and no external dependency beyond pandas. The random seed is fixed at 42 throughout so results are fully reproducible.

## Computing embeddings

Once the dataset exists we run `generate_embeddings.ipynb` to embed every `MO_TEXT` record using `all-MiniLM-L6-v2` from the sentence-transformers library. This model produces 384-dimensional embeddings and is a standard baseline for semantic search benchmarks, which makes it the right choice for a book that compares libraries rather than models.

Embedding 500,000 records at batch size 256 takes roughly 8 to 9 minutes on a MacBook Air M4. The output is saved as:

- `nordvik_embeddings.npy` - float32 array of shape (500000, 384), 732MB
- `nordvik_ids.txt` - IncidentID strings in matching row order

We normalize embeddings to unit length before saving. This means L2 distance and cosine similarity are equivalent throughout the book and we can compare results across libraries that use different default metrics without worrying about systematic differences.

## Ground truth

Every chapter measures recall against a consistent ground truth computed in Day 1 using FAISS's exact `IndexFlatL2`. Rather than computing ground truth for all 500,000 records - which would be expensive - we sample 1,000 queries at random and store their exact top-10 neighbors. This ground truth sample is saved as:

- `nordvik_ground_truth.npy` - shape (1000, 10), exact neighbor indices
- `nordvik_ground_truth_indices.npy` - the 1,000 sampled query positions

Every chapter loads these files and uses them to compute recall@10. One caveat applies throughout the book: because the ground truth was computed using L2 distance and most libraries use cosine distance, floating point differences in implementation produce slightly different result orderings at the margin. This causes recall to appear capped around 0.89 to 0.90 even for libraries that should theoretically achieve higher. We'll explain this in each chapter where it appears.

## A note on the sanity check

The embeddings notebook includes a sanity check that compares cosine similarity between two burglary incidents against cosine similarity between a burglary and an assault. The result is counterintuitive: the two burglaries score 0.41 while the burglary and assault score 0.49.

This happens because the two burglary incidents use very different templates and different neighborhoods. The assault happens to share structural phrasing with the first burglary. It's a reminder that embedding similarity reflects linguistic similarity, not categorical similarity. A search for "incidents like this burglary" will find other incidents that are described in similar language - which may or may not be the same crime type. We'll return to this observation in Day 1.
