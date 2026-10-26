# Packshot Integrity Gate

This service takes a grocery product record and a set of candidate photographs. It then
picks the correct front-of-pack image, or it refuses and gives the reason. The refusal is
the product of the system, so the language in this glossary is exact about the meaning of a refusal.

## Language

**Product**:
A grocery item as the record describes it, identified by its barcode. The capstone brief
calls this a "post" or an "article". This repository does not.
_Avoid_: Post, article, item, SKU

**Tenant**:
One customer account of this service, which is the data team of a retailer. A tenant owns
its product records and its images, and sees nothing that belongs to another tenant.
_Avoid_: Retailer, store, organization, customer

**Packshot**:
A photograph of the physical packaging of a product.
_Avoid_: Product shot, pack image

**Front-of-pack image**:
The one packshot that shows the front face of the pack. A shopper reads this face to
identify brand and variant.
_Avoid_: Main image, primary image, hero image

**Candidate image**:
A packshot offered for one product, before the system decides anything about it.
_Avoid_: Option, match, result

**Variant**:
The version of a product within one brand and category, such as zero sugar against
original, or 330ml against 1.5 liter.
_Avoid_: Flavor, type, version

**Variant marker**:
A short phrase that the vision model reads off the pack, which identifies the variant. Two
examples are "zero sugar" and "330ml".
_Avoid_: Attribute, feature, label

**Marker group**:
A set of variant markers that cannot all be true of one pack, such as zero sugar against
original. Two markers contradict in one case only. Both markers name a member of one group,
and the members differ.
_Avoid_: Antonym set, exclusion list

**Tag set**:
The validated structured output of one vision call about one image: brand, product name,
category, variant markers, pack format, caption, and confidence.
_Avoid_: Metadata, tags, annotation

**Flagged**:
A property of a tag set whose vision confidence is less than the low-confidence threshold.
A flagged tag set is still ranked, but it can never reach `ACCEPT`. Flagged describes an
input. `REVIEW` describes a verdict.
_Avoid_: Uncertain, unreliable, pending

**Suggestion**:
One candidate image paired with one product, which carries a similarity score, a decision,
and a reason.
_Avoid_: Recommendation, pairing, match

**Decision**:
The verdict of the guard on a suggestion: `ACCEPT`, `REVIEW`, or `REJECT`.
_Avoid_: Status, state, outcome

**Reason code**:
The machine-readable name of the rule that produced a decision, such as `BRAND_MISMATCH`.
Every decision carries one, with human-readable reason text beside it.
_Avoid_: Error code, failure reason

**Guard**:
The module that applies the rules to the top-ranked candidate and returns a decision with
a reason code. Ranking proposes. The guard decides.
_Avoid_: Validator, checker, filter

**Candidate pool**:
Every image in the corpus of one tenant. Ranking considers the whole pool, never only the
images that came from the barcode of the product.
_Avoid_: Search space, index, shortlist

**Accept floor**:
The similarity score at which the guard can return `ACCEPT`. A higher score also returns
`ACCEPT`.
_Avoid_: Threshold, cutoff

**Review floor**:
The similarity score under which the guard returns `REJECT` with `BELOW_THRESHOLD`. A
score between the review floor and the accept floor returns `REVIEW`.
_Avoid_: Lower threshold, minimum score

**Notional cost**:
The cost that a call carries at published paid-tier rates. Every call in this project is
free, so no money changed hands.
_Avoid_: Cost, spend, price

**Unanswerable product**:
A product in the evaluation set whose candidate pool holds no correct image. It proves that
the system answers "no confident match" instead of a guess.
_Avoid_: Negative case, empty case

**Wrong-publish rate**:
The share of `ACCEPT` decisions that name the wrong image. This number sets the similarity
threshold.
_Avoid_: Error rate, false positive rate
