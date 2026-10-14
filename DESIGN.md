# Design: Packshot Integrity Gate

One page. Phase 1 gate for the FlyRank capstone. Scope here is the capstone only.

## Problem

A grocery product record says one thing. The photograph attached to it says another.

Open Food Facts records this failure in their own tracker, issue 7050. Products arrive
from GS1 with several views of the pack. Only one image gets marked as the front. The
record then shows updated ingredients next to an old photograph.

In the United Kingdom this is more than untidy. A pack photo is how a shopper reads a
brand and a variant before buying. If the photo shows the wrong variant, the shopper
reads the wrong allergen and nutrition information. That is a compliance matter under
the Food Information Regulations 2014.

This system takes a product record and a set of candidate images. It picks the correct
front-of-pack image, or it refuses and says why. The refusal is the product. A system
that guesses is worse than a system that hands the item to a human.

## Non-goal

No frontend. The review workflow is API endpoints and a database table. A reviewer uses
curl or a database client.

Second non-goal, stated for clarity: this is not a general image search engine. It ranks
candidates for one known product record.

## Data

Corpus: Open Food Facts, products tagged United Kingdom. 196,660 products exist. This
capstone samples about 50 images across at least 4 categories, such as soft drinks,
breakfast cereals, crisps, and dairy.

The sample includes confusable variant pairs on purpose. Coca-Cola against Coca-Cola
Zero is the demonstration pair. It plays the role the brief gives to the fox and the
wolf.

Images are licensed CC BY-SA. The database is licensed ODbL. Packaging artwork carries
third-party trademark and copyright, so the repository holds a small sample with
attribution, plus a download script.

## Tag schema

Every vision call returns this shape. The service validates it with Pydantic. Invalid
responses are retried, then flagged. They are never trusted.

```json
{
  "brand": "Coca-Cola",
  "product_name": "Coca-Cola Zero Sugar",
  "category": "soft drink",
  "variant_markers": ["zero sugar", "330ml", "can"],
  "pack_format": "can",
  "caption": "A black and red 330ml can of Coca-Cola Zero Sugar",
  "confidence": 0.91
}
```

If `confidence` falls below 0.6, the image is flagged for review. It is not used as an
accepted tag set.

## Matching

Two embedding streams meet at the ranking step.

1. The vision caption for each image becomes an embedding, stored in `image_vectors`.
2. The product record text (name, brand, category, ingredients) becomes an embedding,
   stored in `product_vectors`.

Ranking is cosine similarity between the product vector and each candidate image vector.
Ranking alone never decides. The guard decides.

## The guard

The guard runs on the top-ranked candidate. It returns one of three decisions with a
reason code.

| Rule | Reason code if it fails |
|---|---|
| After normalization, the detected brand matches the record brand | `BRAND_MISMATCH` |
| Detected category matches the record category | `CATEGORY_MISMATCH` |
| Detected variant markers do not contradict the record variant | `VARIANT_MISMATCH` |
| Cosine similarity is at or above the tuned threshold | `BELOW_THRESHOLD` |
| Vision confidence is at or above 0.6 | `LOW_CONFIDENCE` |

Decisions:

- `ACCEPT`. Every rule passes. The image is suggested.
- `REVIEW`. Similarity or confidence is marginal. A human sees it.
- `REJECT`. A hard rule fails, or no candidate clears the bar. Reason code
  `NO_CONFIDENT_MATCH` covers the empty case.

Every decision stores its reason in plain words. Example: "Variant mismatch: record says
zero sugar, the pack reads original."

The threshold is not guessed. It comes from the labeled evaluation set.

## Database

| Table | Holds | Indexes |
|---|---|---|
| `products` | Record text from Open Food Facts, with the barcode | unique on `barcode` |
| `images` | File path, source URL, attribution, `product_id` | `product_id` |
| `image_tags` | Validated vision output, confidence, flagged state | `image_id`, `flagged` |
| `image_vectors` | Caption embedding | `image_id` |
| `product_vectors` | Record embedding | `product_id` |
| `suggestions` | Ranked candidate, score, decision, reason code, reason text | unique on `(product_id, image_id)` |
| `reviews` | Approve or reject, reviewer, timestamp | `suggestion_id` |
| `ai_calls` | Model, tokens, cost, call type, timestamp | `created_at` |

Schema arrives as migrations. Vectors are stored as arrays at this scale, which the brief
permits for about 50 images.

## API surface

| Method and path | Does |
|---|---|
| `POST /jobs/vision` | Starts the batch vision job over unprocessed images |
| `GET /jobs/{job_id}` | Reports progress, retries, and failures |
| `GET /products/{barcode}/images` | Returns ranked candidates with decision and reason |
| `POST /suggestions/{id}/approve` | Records an approval |
| `POST /suggestions/{id}/reject` | Records a rejection with a note |
| `GET /suggestions/{id}/explain` | Returns scores, tags, and the rule that fired |
| `GET /costs` | Returns per-call cost entries and a total |

Input validation happens at the boundary. Bad input returns a 4xx response, never a 500.
Approve and reject are idempotent, so a retried call records one action.

## Layers

```
app/api/        HTTP only. Request and response models. No logic.
app/services/   Vision client, embedding client, ranking, the guard, the batch runner.
app/models/     SQLAlchemy tables and migrations.
app/schemas/    Pydantic schemas, including the tag schema above.
app/core/       Configuration, cost logging, retries.
```

The guard is built first, as its own module, with its own tests. The rest of the system
is assembled around it.

## Flow

```
Images --(batch job)--> Vision model --> validated tags --> image_tags
                             |
                             +-- embed(caption) ----------> image_vectors

Product record ---------- embed(record text) -------------> product_vectors

GET /products/{barcode}/images
    -> rank by cosine similarity
        -> guard (brand + category + variant + threshold + confidence)
            -> ACCEPT: suggested image, with its score
            -> REVIEW: sent to the review queue
            -> REJECT: "no confident match" + reason
                -> POST approve / reject
```

## How quality is measured

A labeled evaluation set of 20 products. For each product, one image is marked correct by
hand. Two numbers go in the README:

1. Top-1 precision. The share of products whose first suggested image is the correct one.
2. Wrong-publish rate. The share of `ACCEPT` decisions that are wrong.

The second number sets the threshold. Raise the threshold and fewer wrong images publish,
but more products go to a human. That trade is the point of the system.

## Stack

Python, FastAPI, Pydantic, PostgreSQL in Docker. Gemini Flash free tier for vision and
embeddings, with Ollama as the local fallback. Keys live in `.env`, never in code.
