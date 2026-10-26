# Dataset research: the choice of the image corpus

Date: 2026-10-14. The research searched for the candidates directly, not from memory.

The goal was an image dataset with four properties: an open license, a United Kingdom
link, at least four confusable categories or text for each image, and a real business
problem behind it.

## Verified candidates

| Dataset | UK link | Images | Text per image | Confusable pair | License | Redistribute | Status |
|---|---|---|---|---|---|---|---|
| DVM-CAR | UK car market data, built at Southampton and Glasgow | 1,451,784 across 899 models | Make, model, trim, year, color, plus 250,000 used-car adverts | BMW 3 Series against 5 Series | CC BY-NC 4.0 | Yes, non-commercial, with attribution | Verified on the project site |
| Open Food Facts (UK products) | 196,660 products tagged United Kingdom | Millions of packaging photos | Product name, brand, ingredients, allergens | Coca-Cola against Coca-Cola Zero | Images CC BY-SA, database ODbL | Yes, share-alike applies | Verified. The UK count was read from the API |
| Oxford-IIIT Pet | Built at Oxford | About 7,349 across 37 breeds | Breed label only | British Shorthair against Russian Blue | CC BY-SA 4.0 | Yes | Known, not re-examined in this session |
| Science Museum Group collection | UK national museum group | Large | Object title and description | Similar instruments | Metadata open, image rights differ for each object | Per object | Needs rights work for each item |

## Rejected, with the reason

| Candidate | Why it fails |
|---|---|
| Rightmove and UK property sets | Only scraped tables with image links. No open image corpus. |
| UK road and pothole open data | Northern Ireland surface defects is Open Government License, but tabular, not images. |
| data.gov.uk image holdings | Living England satellite data, Northern Ireland aerial survey, North Somerset Flickr links. None carry product-style categories. |
| Trap Tracker and DeepFaune UK | Released as trained models under a non-commercial license. The training images are not an open corpus. |
| TACO litter dataset | MIT licensed, and it includes London streets. But it is not a UK dataset, and the business buyer is weak. |
| Myntra fashion, Grocery Store Dataset, Amazon Berkeley Objects | Ready-made and well labeled, but Indian, Swedish, and American. |

## The finding that shaped the recommendation

Open Food Facts documents this exact failure in its own tracker, as issue 7050 on
openfoodfacts-server. Products arrive from GS1 with several views, and only the front image
carries the front mark. As a result, the database holds new ingredients and nutrition
data beside an old photo. That is a live, unsolved, image-to-record matching problem in a real
project.

## Trade-off

DVM-CAR has cleaner labels and a tie to automotive retail. But its images are 300x300 with
the background removed, which is much cleaner than a real dealer photograph. CC BY-NC also
blocks commercial reuse.

Open Food Facts has real phone photographs of real packaging. It has a share-alike license
that permits commercial use, and a UK regulatory hook through allergen labeling. The
problem it documents is still open. It has no ready-made correct-pairing labels, so you
must label the small evaluation set by hand. The capstone brief requires that work
anyway.

## Sources

- DVM-CAR: https://deepvisualmarketing.github.io/
- Open Food Facts data and licenses: https://world.openfoodfacts.org/data
- Open Food Facts images on AWS: https://registry.opendata.aws/openfoodfacts-images/
- The stale photo problem: https://github.com/openfoodfacts/openfoodfacts-server/issues/7050
- Northern Ireland surface defects: https://www.opendatani.gov.uk/dataset/surface-defects
