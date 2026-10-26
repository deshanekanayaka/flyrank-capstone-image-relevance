# The candidate pool is the whole tenant corpus

Open Food Facts supplies images for each barcode. Thus the obvious design restricts the
candidates for a product to the images already linked to it. That design makes ranking a
choice between four photographs of the same pack. The guard then has nothing to refuse,
because no wrong-category image is ever a candidate. We rank every image in the corpus of
the tenant instead.

## Consequences

`images.source_product_id` records the source of a photograph, and is never used as a
matching filter. A reader who sees that column and assumes that it scopes the search reads
the ranking code as wrong. That is the reason to write this decision down. The labels for
the evaluation set come free from the corpus, because the correct image for a product is the
one whose `source_product_id` matches it. At about 50 images a full scan costs a few
milliseconds, so no vector index is needed.
