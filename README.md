# Packshot Integrity Gate

A grocery product record says one thing. The photograph attached to it says another.

This service takes one product record and every photograph a retailer holds, and it does
three things. It picks the front-of-pack image that matches the record. It refuses the
images that do not match, and it names the rule that refused each one. When nothing matches
well enough, it answers "no confident match" instead of guessing.

The refusal is the product. A system that guesses is worse than a system that sends the
item to a person.

```
POST /jobs/vision          -> 50 images tagged in the background
GET  /products/5449000000996/images

  1. 0.81  REJECT  VARIANT_MISMATCH
     "Variant mismatch: the record says original, the pack reads zero sugar."
  2. 0.74  ACCEPT
     "Brand, category, and variant agree. Score is above the accept floor."
```

## Why it exists

Open Food Facts receives product data from GS1, and each delivery carries several
photographs of the pack. Only one photograph holds the front mark. When a maker changes the
recipe, the import overwrites the text fields and leaves the front mark where it was, so the
record ends up showing new ingredients beside an old photograph. That is issue 7050 on
`openfoodfacts-server`, and it is still open.

A UK shopper reads the pack in the photograph to identify a brand and a variant before a
purchase. A wrong photograph therefore shows the wrong allergen and nutrition information,
which is a compliance matter under the Food Information Regulations 2014. A retailer cannot
solve this by eye, because a wrong front image looks exactly like a right one, so nobody
knows which of 200,000 records to inspect.

## How it decides

Two streams meet at the ranking step, and the guard decides after it.

```
images --(batch job)--> vision model --> validated tags --> image_tags
                            |
                            +-- embed(caption) -----------> image_vectors

product record ---------- embed(record text) -------------> product_vectors

GET /products/{barcode}/images
    -> rank the whole tenant corpus by cosine similarity
        -> guard (brand + category + variant + two floors + confidence)
            -> ACCEPT  : suggested front image, with its score
            -> REVIEW  : a person sees it
            -> REJECT  : refusal with a reason code and a sentence
```

Ranking proposes, and it compares meaning rather than words, so a caption that reads "fizzy
cola with no sugar" still matches a record that reads "Zero Sugar carbonated soft drink".
Ranking alone never decides, because the best-ordered candidate is still wrong in the case
that matters most: a Coke Zero packshot under a Coca-Cola Original record shares brand,
category, and pack format with the record. The guard is a pure function that refuses it on
the variant rule.

Two floors exist because three decisions need two cut points. A higher accept floor
publishes fewer wrong images and sends more products to a person. Both numbers come from a
labeled evaluation set, and neither one is guessed.

## Status

Phase 1 is complete. The design, the glossary, the architecture decisions, and the build
plan are committed. There is no application code yet, so the commands below describe the
contract that `capstone.yaml` promises, not a running system today.

## Stack

Python, FastAPI, Pydantic, PostgreSQL in Docker, and Alembic. Vision is `gemini-3.8-flash`
and embeddings are `gemini-embedding-2-preview`, both pinned in configuration and recorded
on every call. The whole project runs on a free tier, and every call still writes a notional
cost at published paid-tier rates, so the budget guard can refuse.

There is no frontend. The review workflow is a set of API endpoints and a database table.

## Run

```
cp .env.example .env     # then add your Gemini key
docker compose up --build
docker compose exec web python -m scripts.seed      # prints the demonstration tenant key
docker compose exec web pytest
```

Every route is tenant-scoped and needs an `X-Tenant-Key` header. The seed is idempotent, so
running it twice creates no duplicate products.

## Results

Not measured yet. Two numbers go here when the evaluation set exists: top-1 precision, and
the wrong-publish rate, which is the share of `ACCEPT` decisions that name the wrong image.
The second number sets the two floors.

## The documents

| File | Holds |
|---|---|
| `DESIGN.md` | The one-page design: problem, data, guard rules, schema, API surface |
| `CONTEXT.md` | The glossary. Every term in this repository has one meaning |
| `docs/build-plan.md` | Seven days of work, each with a reading list and a gate |
| `docs/adr/` | The decisions that were hard to reverse, and why |
| `docs/dataset-research.md` | Why Open Food Facts, and what was rejected |
| `BUILDLOG.md` | Where the AI helped, where it was wrong, and what changed |
| `EVIDENCE.md` | One pasted proof per requirement in the brief |

## Data and license

The corpus is Open Food Facts, for products tagged United Kingdom. The images carry CC
BY-SA and the database carries ODbL, and the packaging artwork carries third-party
trademark and copyright. Attribution per image is in `data/corpus/LICENSE.md`.

The root `LICENSE` is MIT, and it covers the code only.
