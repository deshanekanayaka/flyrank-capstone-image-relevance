# Build plan: seven days

This plan turns `DESIGN.md` into seven days of work. Each day has a goal, a short reading
list, a build list, a list of traps, and a gate. A gate is a concrete result that proves
the day is finished.

The brief estimates 35 to 50 focused hours. Seven days means about 6 hours a day. If a day
runs long, cut scope inside the day and keep the gate. Do not skip a day and do not change
the order. The order exists because day 5 changes the database migrations that day 2
writes, and learning that on one image is cheaper than learning it on eight tables.

Every day ends the same way. You paste the proof into `EVIDENCE.md`, you write what the AI
got wrong into `BUILDLOG.md`, and you commit.

## Day 1: The guard and the schemas

Goal: the decision core exists as a pure function with tests, and it touches no database
and no network.

Read first:

- `DESIGN.md`, the sections "Tag schema" and "The guard".
- `CONTEXT.md`, the entries for guard, tag set, flagged, decision, reason code, marker
  group, accept floor, and review floor.
- Section 4, part 3 of the capstone brief, which is the mismatch guard.

Build:

- `app/schemas/` with the tag set as a Pydantic model, and the decision and reason code as
  enumerations.
- `app/services/guard.py` with one pure function. It takes a tag set, a product record, a
  similarity score, and two floors. It returns a decision and a reason code.
- Brand normalization and marker groups as separate small functions.
- `tests/test_guard.py` with one test for each reason code in the table.

Watch for:

- The floors are arguments to the guard, never values that the guard reads from
  configuration. A pure function that reads the environment is not testable.
- Decide the precedence of the rules now, and write it down. A photo can fail the brand
  rule and the confidence rule at the same time, and the reason code that the user sees
  must be the same on every run.
- `LOW_CONFIDENCE` is a property of the input. `REVIEW` is a verdict. Keep the two words
  apart in the code, as `CONTEXT.md` requires.

Gate: `pytest tests/test_guard.py` passes, with one named test for `BRAND_MISMATCH`,
`CATEGORY_MISMATCH`, `VARIANT_MISMATCH`, `BELOW_THRESHOLD`, `LOW_CONFIDENCE`, and
`NO_CONFIDENT_MATCH`.

## Day 2: Database, tenancy, and the application skeleton

Goal: `docker compose up --build` starts a web container and a PostgreSQL container, the
migrations run, and a request without a tenant key returns 401.

Read first:

- `DESIGN.md`, the sections "Tenancy", "Database", and "Layers".
- `docs/adr/0001-tenant-is-an-account.md`.

Build:

- `docker-compose.yml` with a web service, a database service, and a worker service. The
  worker can stay empty until day 4.
- `app/core/config.py`, which reads `.env` with Pydantic settings.
- All eleven tables from the table in `DESIGN.md`, as SQLAlchemy models.
- The first Alembic migration, and the entrypoint that runs `alembic upgrade head`.
- `app/core/tenancy.py`, which reads `X-Tenant-Key`, hashes it, and finds the tenant.
- A FastAPI dependency that returns 401 for a missing or unknown key.

Watch for:

- Alembic autogenerate misses index and constraint changes. Read every generated script
  before you commit it, and make sure that the unique index on `(tenant_id, barcode)` is
  present.
- Store a hash of the tenant key, never the key. Make sure that no log line and no error
  message contains the key.
- Decide the vector column type now, as a float array, and accept that day 5 can change
  its length. Write the length as a variable in the migration, not as a literal.

Gate: `docker compose up --build` starts on a clean machine, the tables exist, and a curl
request with no header returns 401.

## Day 3: The corpus and the product records

Goal: about 50 images and their product records exist in the database, with attribution,
and the demonstration operates with no network.

Read first:

- `DESIGN.md`, the section "Data".
- `docs/dataset-research.md`.
- `data/corpus/LICENSE.md`.

Build:

- `data/corpus/manifest.csv` with a barcode, an image URL, an author, and a license for
  each image. Pin one image revision per row.
- `scripts/fetch_corpus.py`, which downloads the images that are not committed.
- `scripts/seed_products.py`, which reads the Open Food Facts API and writes one product
  row per store from `stores_tags`.
- The five demonstration images, committed to the repository.
- A seed step that prints the demonstration tenant key.

Watch for:

- The seed must be idempotent, which means safe to run twice. `capstone.yaml` promises
  this. Use an upsert on `(tenant_id, barcode)`.
- Make sure that the sample contains at least one case for every reason code. The table in
  `DESIGN.md` names them.
- Do not pick the three concept pairs yet. The design says that you pick them after the
  first batch run, from the captions that the model wrote.
- Record the author and the license for every image on the day you download it. You cannot
  reconstruct attribution later.

Gate: the seed runs twice and creates no duplicate rows, and `images` and `products` hold
the counts that you expect.

## Day 4: The vision pipeline

Goal: a batch job reads every image, and `image_tags` holds a validated tag set for each
one, with the cost of every call recorded.

Read first:

- `DESIGN.md`, the sections "Background processing" and "Cost and budget".
- `docs/adr/0002-notional-cost-on-a-free-tier.md`.
- The Gemini structured-output documentation, for the exact way to pin a response schema.

Build:

- `app/services/vision.py`, with the real provider and a fake provider behind the same
  small interface.
- Pydantic validation of every response, with two retries on a schema failure and the
  terminal state `failed_schema`.
- The retry policy with three classes, as the table in `DESIGN.md` describes.
- `POST /jobs/vision` and `GET /jobs/{job_id}`.
- The worker, which claims a row with `SELECT ... FOR UPDATE SKIP LOCKED`.
- One `ai_calls` row per call, with the model name, the tokens, and the notional cost.
- The budget guard, which refuses the next call with a 429.

Watch for:

- `POST /jobs/vision` must write rows and return. It must never call a model inside the
  request.
- Ask the model for the fields separately, as `brand`, `category`, and `variant_markers`.
  Do not read the brand back out of the caption.
- A failed item must keep its validation message in `last_error`. That row is the evidence
  that the system never trusts an invalid response.
- Test the budget guard with a small budget value. A budget guard that you never saw
  refuse a call is a budget meter.

Gate: the batch job tags all 50 images, `GET /costs` returns a total, and at least one
deliberately broken response ends as `failed_schema` with a message.

## Day 5: Embeddings, ranking, and one product end to end

Goal: `GET /products/{barcode}/images` returns ranked candidates, each with a decision and
a reason.

Read first:

- `DESIGN.md`, the sections "Matching" and "Flow".
- `docs/adr/0003-corpus-wide-candidate-pool.md`.

Build:

- `app/services/embedding.py`, which embeds a caption and a product record text.
- `image_vectors` and `product_vectors`, written by the batch job.
- Cosine similarity as a pure function, with its own tests.
- `GET /products/{barcode}/images`, which ranks the whole tenant pool and calls the guard.
- The decision snapshot, written onto the `suggestions` row.

Watch for:

- Read the vector length from the first live response, then fix it in the migration. Do
  not guess the length.
- The candidate pool is every image of the tenant. `images.source_product_id` is
  provenance only, and it is never a filter.
- Write the score, both floors, the confidence, and the compared strings onto the
  suggestion row at the moment of the decision. An explanation that recomputes can drift
  from the decision that it explains.
- Pick the three concept pairs today, from the captions that the model wrote on day 4.

Gate: the Coca-Cola record ranks the Coca-Cola can first, and the guard refuses the Zero
Sugar can with `VARIANT_MISMATCH` and a sentence in plain words.

## Day 6: The review workflow

Goal: a person can see the queue, approve, reject, and ask why.

Read first:

- `DESIGN.md`, the section "API surface".
- Section 4, part 5 of the capstone brief, which is the review API.

Build:

- `GET /suggestions?decision=REVIEW`.
- `POST /suggestions/{id}/approve`, which writes a review and sets
  `products.front_image_id` and `products.front_image_set_at`.
- `POST /suggestions/{id}/reject`, which records a note.
- `GET /suggestions/{id}/explain`, which reads the stored snapshot and recomputes nothing.
- Boundary tests against the PostgreSQL container: a 4xx response on bad input, tenant
  isolation, and a repeated approval.

Watch for:

- Approve and reject are idempotent. The unique constraint on `reviews.suggestion_id`
  means that the first verdict wins. A repeat returns 200 with the existing review, and a
  conflicting verdict returns 409.
- Bad input returns a 4xx response, never a 500. Test an unknown suggestion id and a
  malformed body.
- Tenant isolation needs a real test. Create two tenants with the same barcode, and make
  sure that each key sees only its own rows.

Gate: an approval sets the front image, a second identical approval returns 200 and
changes nothing, and a tenant key cannot read the suggestions of another tenant.

## Day 7: Evaluation, the numbers, and the documents

Goal: the two quality numbers exist, the floors come from data, and a stranger can run the
project from the README.

Read first:

- `DESIGN.md`, the section "How quality is measured".
- Section 6 of the capstone brief, which is the requirements checklist.

Build:

- The labeled evaluation set: 20 products with one correct image marked by hand, plus 4
  unanswerable products.
- `eval/run_eval.py`, which sweeps the two floors and prints the pair that it chose.
- Top-1 precision and the wrong-publish rate, both written into the README.
- The tuned floors, committed to `.env.example`, with a comment that names the run.
- The README, with the architecture diagram, and `docs/architecture.md`.
- `EVIDENCE.md`, with one pasted proof against each box in Section 6.
- `BUILDLOG.md`, honest about where the AI helped and where it was wrong.

Watch for:

- Label the evaluation set before you look at the output of the system. A label that you
  wrote after you saw the answer measures nothing.
- Report both numbers, including the one that looks bad. The trade between the two is the
  argument of the project.
- Make sure that you can explain any two or three lines of the code that a reviewer picks.

Gate: `eval/run_eval.py` prints both floors and both numbers, the README holds them, and
every box in Section 6 has a pasted proof in `EVIDENCE.md`.

