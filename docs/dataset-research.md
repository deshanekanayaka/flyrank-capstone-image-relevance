# Dataset research — choosing the image corpus

Date: 2026-10-14. Researched by running the candidate search directly, not from memory.

Goal: an existing, openly licensed image dataset with a United Kingdom link, at least
four confusable categories or per-image text, and a real business problem behind it.

## Verified candidates

| Dataset | UK link | Images | Text per image | Confusable pair | Licence | Redistribute | Status |
|---|---|---|---|---|---|---|---|
| DVM-CAR | UK car market data, built at Southampton and Glasgow | 1,451,784 across 899 models | Make, model, trim, year, colour, plus 250,000 used-car adverts | BMW 3 Series against 5 Series | CC BY-NC 4.0 | Yes, non-commercial, with attribution | Verified on the project site |
| Open Food Facts (UK products) | 196,660 products tagged United Kingdom | Millions of packaging photos | Product name, brand, ingredients, allergens | Coca-Cola against Coca-Cola Zero | Images CC BY-SA, database ODbL | Yes, share-alike applies | Verified; UK count read from the API |
| Oxford-IIIT Pet | Built at Oxford | About 7,349 across 37 breeds | Breed label only | British Shorthair against Russian Blue | CC BY-SA 4.0 | Yes | Known, not re-checked this session |
| Science Museum Group collection | UK national museum group | Large | Object title and description | Similar instruments | Metadata open, image rights vary per object | Per object | Needs per-item rights work |

## Rejected, with the reason

| Candidate | Why it fails |
|---|---|
| Rightmove and UK property sets | Only scraped tables with image links. No open image corpus. |
| UK road and pothole open data | Northern Ireland surface defects is Open Government Licence but tabular, not images. |
| data.gov.uk image holdings | Living England satellite data, Northern Ireland aerial survey, North Somerset Flickr links. None carry product-style categories. |
| Trap Tracker and DeepFaune UK | Released as trained models under a non-commercial licence. The training images are not an open corpus. |
| TACO litter dataset | MIT licensed and includes London streets, but it is not a UK dataset and the business buyer is weak. |
| Myntra fashion, Grocery Store Dataset, Amazon Berkeley Objects | Ready-made and well labelled, but Indian, Swedish, and American. |

## Finding that shaped the recommendation

Open Food Facts documents this exact failure in its own tracker, issue 7050 on
openfoodfacts-server: products arrive from GS1 with several views, only the front image
is marked, so the database ends up with updated ingredients and nutrition beside an old
photo. That is a live, unsolved, image-to-record matching problem in a real project.

## Trade-off

DVM-CAR has cleaner labels and ties to automotive retail, but its images are 300x300 with
the background removed, which is far cleaner than a real dealer photograph, and CC BY-NC
blocks commercial reuse.

Open Food Facts has real phone photographs of real packaging, a share-alike licence that
permits commercial use, a UK regulatory hook through allergen labelling, and a documented
problem that no one has solved. It has no ready-made correct-pairing labels, so the small
evaluation set must be hand-labelled, which the capstone brief requires anyway.

## Sources

- DVM-CAR: https://deepvisualmarketing.github.io/
- Open Food Facts data and licences: https://world.openfoodfacts.org/data
- Open Food Facts images on AWS: https://registry.opendata.aws/openfoodfacts-images/
- Stale photo issue: https://github.com/openfoodfacts/openfoodfacts-server/issues/7050
- Northern Ireland surface defects: https://www.opendatani.gov.uk/dataset/surface-defects
