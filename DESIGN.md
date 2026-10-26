# Design: Packshot Integrity Gate

One page. This document is the Phase 1 gate for the FlyRank capstone. The scope here is
the capstone only.

## Problem

A grocery product record says one thing. The photograph attached to it says another.

Open Food Facts records this failure in their own tracker, as issue 7050. Products arrive
from GS1 with several views of the pack. Only one image carries the front mark. The record
then shows new ingredients beside an old photograph.

In the United Kingdom this is a legal risk. A shopper reads the pack photo to identify a
brand and a variant before the purchase. If the photo shows the wrong variant, the shopper
reads the wrong allergen and nutrition information. That is a compliance matter under the
Food Information Regulations 2014.

This system takes a product record and a set of candidate images. It picks the correct
front-of-pack image, or it refuses and gives the reason. The refusal is the product. A
system that guesses is worse than a system that sends the item to a person.

## Non-goals

There is no frontend. The review workflow is a set of API endpoints and a database table.
A reviewer uses curl or a database client.

This is also not a general image search engine. It ranks candidates for one known product
record.

## Data

The corpus is Open Food Facts, for the products tagged United Kingdom. 196,660 products
exist. This capstone samples about 50 images across 4 categories: soft drinks, breakfast
cereals, crisps, and dairy. The split is about 12 images per category.

The sample is chosen so that each reason code occurs at least once:

| Evaluation item | Reason code |
|---|---|
| A Coca-Cola record against a Coca-Cola Zero Sugar pack | `VARIANT_MISMATCH` |
| A cereal record against a crisps packet | `CATEGORY_MISMATCH` |
| A branded record against a supermarket own-label pack | `BRAND_MISMATCH` |
| One blurred or partial pack | `LOW_CONFIDENCE` |
| Four unanswerable products, with pools drawn from other categories | `NO_CONFIDENT_MATCH` |

The Coca-Cola pair is the demonstration pair. It plays the role that the brief gives to
the fox and the wolf. It is more difficult, because the two cans share brand, category,
and pack format.

Three pairs prove that matching works on concepts and not on words. In such a pair, the
caption of the correct image shares fewer words with the record than a wrong image does.
The pairs are chosen after the first batch run, from the captions that the vision model
wrote. A pair chosen in advance can pass without a test of anything.

The images carry the CC BY-SA license. The database carries the ODbL license. The
packaging artwork carries third-party trademark and copyright.

The repository holds `data/corpus/manifest.csv`, with a barcode, an image URL, an author,
and a license for each image. `scripts/fetch_corpus.py` downloads the rest. Five images
for the demonstration pairs are committed, so the demonstration operates with no network.
`data/corpus/LICENSE.md` records the image license and the attribution. The root `LICENSE`
is MIT and covers code only.

Manifest rows pin one image revision. When a contributor uploads a better photograph, the
Open Food Facts image changes. An unpinned corpus makes the precision number
unreproducible.

## Tenancy

A tenant is one customer account of this service, which is the data team of a retailer. A
tenant owns its product records and its images. A tenant sees nothing that belongs to
another tenant. The same barcode can exist in two tenants with different images attached.
See `docs/adr/0001-tenant-is-an-account.md`.

A request names its tenant with an `X-Tenant-Key` header. The `tenants` table stores a
hash of the key, never the key. A missing or unknown key returns 401. The key never
reaches a log line.

Product records arrive through `scripts/seed_products.py` only. The script reads the Open
Food Facts API. It assigns each product to a tenant from the `stores_tags` field, and
writes one row per store. There is no write endpoint for products. This is a deliberate
non-goal.

## Tag schema

Every vision call returns this shape. The service uses Pydantic to make sure that the
response matches the shape. An invalid response is retried, then recorded as failed. An
invalid response is never trusted.

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

If `confidence` is less than 0.6, the tag set is flagged. A flagged tag set is still
ranked, but it can never reach `ACCEPT`. Flagged describes an input. `REVIEW` describes a
verdict.

## Matching

Two embedding streams meet at the ranking step.

1. The vision caption for each image becomes an embedding, stored in `image_vectors`.
2. The product record text (name, brand, category, ingredients) becomes an embedding,
   stored in `product_vectors`.

Ranking is the cosine similarity between the product vector and each candidate image
vector. The candidate pool is every image in the corpus of the tenant. The pool is not
restricted to the images that came from the product barcode. A pool of one pack gives the
guard nothing to refuse. See `docs/adr/0003-corpus-wide-candidate-pool.md`.

Ranking alone never decides. The guard decides.

## The guard

The guard is a pure function, which is a function with no state and no side effects. It
takes a tag set, a product record, a similarity score, and the two floors. It returns a
decision with a reason code. It knows nothing about PostgreSQL and nothing about Gemini.
It is built first, with its own tests.

| Rule | Reason code for a failed rule |
|---|---|
| The detected brand matches a record brand after normalization | `BRAND_MISMATCH` |
| The detected category matches the record category | `CATEGORY_MISMATCH` |
| The detected variant markers do not contradict the record variant | `VARIANT_MISMATCH` |
| The similarity score is equal to or more than the review floor | `BELOW_THRESHOLD` |
| The vision confidence is equal to or more than 0.6 | `LOW_CONFIDENCE` |

Brand normalization folds case, removes punctuation, and removes the words `organic`,
`original`, and `ltd`. The record `brands` field is free text, split on commas. When the
detected brand matches any entry, the rule passes. The corpus spells one maker as `Jason's
Sourdough`, `Jasons`, and `Jason's`, so an exact match rejects correct images.

String-distance matching is deliberately not used yet. It needs a second tuned number, and
it risks a pass of `Coca-Cola` for `Coca-Cola Zero`.

Marker groups decide variant contradiction. A marker group is a set of variant markers
that cannot all be true of one pack, such as zero sugar against original. Two markers
contradict in one case only. The record and the image both name a member of one group, and
the members differ. A marker that the photograph does not show contradicts nothing. An
absent marker is not evidence of a different pack.

Decisions:

- `ACCEPT`. Every rule passes, and the score is equal to or more than the accept floor.
- `REVIEW`. The score is between the review floor and the accept floor, or the tag set is
  flagged. A person sees it.
- `REJECT`. A hard rule fails, or the score is less than the review floor. The reason code
  `NO_CONFIDENT_MATCH` covers the case where no candidate clears the bar.

Every decision stores its reason in plain words. Example: "Variant mismatch: the record
says zero sugar, the pack reads original."

Two floors exist because three decisions need two cut points. A higher accept floor and a
lower review floor are separate trade-offs. Both numbers come from the labeled evaluation
set. Neither number is guessed.

The guard writes its full input snapshot onto the `suggestions` row: the score, both
floors, the vision confidence, the compared brand and category strings, and the reason
code. `GET /suggestions/{id}/explain` reads that row and recomputes nothing. An explanation
that can drift from the decision it explains is worse than no explanation. A new run of
the pipeline writes a new suggestion, and does not change an old one.

## Database

| Table | Holds | Indexes |
|---|---|---|
| `tenants` | Name, slug, hashed API key | unique on `slug`, unique on `key_hash` |
| `products` | Record text, barcode, `tenant_id`, `front_image_id`, `front_image_set_at` | unique on `(tenant_id, barcode)` |
| `images` | File path, source URL, attribution, `tenant_id`, `source_product_id` | `tenant_id`, `source_product_id` |
| `image_tags` | Validated vision output, confidence, flagged state | `image_id`, `flagged` |
| `image_vectors` | Caption embedding | `image_id` |
| `product_vectors` | Record embedding | `product_id` |
| `suggestions` | Candidate, score, decision, reason code, reason text, decision snapshot | unique on `(product_id, image_id)` |
| `reviews` | Approve or reject, reviewer, note, timestamp | unique on `suggestion_id` |
| `jobs` | Batch status, counts, started and finished timestamps | `status` |
| `job_items` | One row per image: state, `attempts`, `last_error` | `(job_id, state)` |
| `ai_calls` | Model name, tokens, notional cost, call type, timestamp | `created_at` |

`images.source_product_id` is provenance. It records the barcode that a photograph came
from. It is never a matching filter.

The schema arrives as Alembic migrations. Read every autogenerated script before you
commit it. Autogenerate misses index and constraint changes, and the `(tenant_id,
barcode)` unique index is the kind that it gets wrong. Vectors are stored as float arrays,
which the brief permits at about 50 images.

## API surface

| Method and path | Does |
|---|---|
| `POST /jobs/vision` | Writes a job row for every unprocessed image |
| `GET /jobs/{job_id}` | Shows progress, retries, and failures from `job_items` |
| `GET /products/{barcode}/images` | Returns ranked candidates with a decision and a reason |
| `GET /suggestions?decision=REVIEW` | Lists the review queue with a score and a reason |
| `POST /suggestions/{id}/approve` | Records an approval, and sets the front image of the product |
| `POST /suggestions/{id}/reject` | Records a rejection with a note |
| `GET /suggestions/{id}/explain` | Returns the stored decision snapshot |
| `GET /costs` | Returns one notional cost entry per call, and a total |

Every route is tenant-scoped. Input validation happens at the boundary. Bad input returns
a 4xx response, never a 500.

Approve and reject are idempotent, which means safe to send again. The unique constraint
on `reviews.suggestion_id` means that the first verdict wins. A retried call returns 200
with the existing review. A conflicting verdict on the same suggestion returns 409.

An approval sets `products.front_image_id` and `products.front_image_set_at`. This is the
output of the system. The record now shows the right photograph. An approval of a different
suggestion for the same product replaces the front image. That is a change, not a retry.

## Background processing

`POST /jobs/vision` writes rows and returns. It never calls a model.

A worker process operates as a second container in `docker-compose.yml`. It claims a
pending `job_items` row with `SELECT ... FOR UPDATE SKIP LOCKED`. This prevents a claim on
the same image by two workers, and it makes a double trigger harmless.

There are three retry classes:

| Class | Policy |
|---|---|
| A 429 or a 5xx response from the provider | Exponential backoff at 2, 8, and 32 seconds, with respect for `Retry-After`, up to 5 attempts |
| A response that fails schema validation | 2 attempts with the same prompt. The item then ends as `failed_schema`, and the image is flagged |
| Any other 4xx response, or a missing image file | Terminal at the first attempt |

Google does not publish free-tier rate limits, and the limits are account-specific. As a
result, the worker reacts to a 429 instead of a rate in configuration.

A failed item keeps its validation message in `last_error`. That row is the evidence that
invalid responses are never trusted. A batch that ends with any failed item writes a
`WARNING` log line, and exits non-zero.

## Cost and budget

Every vision call and every embedding call writes an `ai_calls` row, with the model name,
the token counts, and `notional_cost_usd`. The free tier charges nothing, so the cost comes
from published paid-tier rates held in configuration. See
`docs/adr/0002-notional-cost-on-a-free-tier.md`.

When the notional cost for the month gets to `MONTHLY_BUDGET_USD`, the next call is refused
with a 429 and a reason. A budget guard that does not refuse is a budget meter.

## Layers

```
app/api/        HTTP only. Request and response models. No logic.
app/services/   Vision client, embedding client, ranking, the guard, the batch runner.
app/models/     SQLAlchemy tables and Alembic migrations.
app/schemas/    Pydantic schemas, including the tag schema in this document.
app/core/       Configuration, cost logging, retries, tenant resolution.
```

## Flow

```
Images --(batch job)--> Vision model --> validated tags --> image_tags
                             |
                             +-- embed(caption) ----------> image_vectors

Product record ---------- embed(record text) -------------> product_vectors

GET /products/{barcode}/images
    -> rank the whole tenant corpus by cosine similarity
        -> guard (brand + category + variant + floors + confidence)
            -> ACCEPT: suggested image, with its score
            -> REVIEW: sent to the review queue
            -> REJECT: "no confident match" + reason
                -> POST approve -> products.front_image_id is set
                -> POST reject  -> recorded with a note
```

## How quality is measured

The labeled evaluation set holds 20 products, each with one image marked correct by hand,
plus 4 unanswerable products. Two numbers go in the README:

1. Top-1 precision. This is the share of products whose first suggested image is the
   correct one.
2. Wrong-publish rate. This is the share of `ACCEPT` decisions that name the wrong image.

The second number sets the floors. A higher accept floor publishes fewer wrong images, but
it sends more products to a person. That trade is the point of the system.

The evaluation script prints both floors that it tuned. The tuned values are committed to
`.env.example`, with a comment that names the run that produced them.

## Testing

The guard, brand normalization, marker groups, and the ranking mathematics are pure
functions. They are tested with no database and no network. A fake vision provider returns
fixed tag sets, so every decision test operates offline.

A second, smaller set of tests operates against the `docker compose` PostgreSQL instance.
It covers the boundary only: a 4xx response on bad input, tenant isolation, and an
idempotent approval.

## Stack

The stack is Python, FastAPI, Pydantic, PostgreSQL in Docker, and Alembic. Vision is
`gemini-3.8-flash`. Embeddings are `gemini-embedding-2-preview`. Both model names are
pinned in configuration, and recorded on every `ai_calls` row. `gemini-embedding-2-preview`
is a preview model that can be withdrawn, and a precision number is unattributable without
the model that produced it. The vector length is read from the first live response, and
fixed in the migration.

One provider is built. A fake provider exists for tests. The README names Ollama as an
untested alternative path. The brief treats model comparison as a stretch, and a provider
that no one operates is a claim that the tests cannot support.

Keys live in `.env`, never in code.

## Build order

The guard comes first, as its own module, with its own tests. Then one image goes end to
end: vision, embedding, ranking, guard, and one endpoint. Then the system widens. The
vector length and the real caption shape both change the migrations. To learn that on image
one is cheaper than to learn it after eight tables exist.
