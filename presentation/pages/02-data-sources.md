---
layout: section
---

# Data Sources

Inventory records · CNN-detected coordinates · RUFA Score

---

# Inventory Data

The first source: a compiled California urban forest inventory.

<v-clicks>

- Aggregated from private arborist firms (e.g., West Coast Arborists, Davey Tree Company, APlus Tree Care), primarily derived from municipal maintenance and risk-assessment records
- Public sources: CAL FIRE urban forestry program and associated municipal tree inventories
- Per-tree attributes: **species, genus, family, trunk diameter, canopy spread, height**
- Geographic precision: point coordinates for each inventoried tree
- USDA Reference: https://www.fs.usda.gov/rds/archive/products/RDS-2017-0011/_metadata_RDS-2017-0011.html

</v-clicks>

<v-click>

**Key limitation**: inventory coverage is uneven. Dense urban cores tend to be well-catalogued; suburban and lower-income areas may have significant gaps.

</v-click>

<!--
- This is the high-information source in RUFA. These are not just tree dots; they are records collected through municipal inventories and arborist workflows.
- The technical value is the attribute richness: species, genus, family, DBH, height, and canopy spread let us compute metrics that require biology, not just location.
- TD-50 and tree evenness depend on species counts, so the inventory data acts as the taxonomic ground truth for the score.
- The coordinates also make the records spatially joinable. Once each tree point is assigned to a city, ZIP, or tract polygon, the same raw records can support multiple geographic rollups.
- The weakness is sampling bias. Inventory coverage follows where cities paid for surveys or maintenance, so dense and wealthier municipalities can be overrepresented.
- That means inventory data is precise where it exists, but incomplete as a statewide coverage layer. RUFA needs another source to fill the spatial gaps.
-->

---

# CNN-Based Tree Detection

The second source: a deep learning detection pipeline over aerial imagery.

<v-clicks>

- Input: **high-resolution multispectral aerial imagery** covering California urban areas
- Architecture: VGG-style convolutional neural network (fully convolutional) for tree-location heatmap regression from multispectral imagery, with peak-based extraction of individual trees (Ventura et al., 2022). https://arxiv.org/abs/2208.10607
- Output: a **confidence map** produced by regressing detected tree locations
- Result: a coordinate for every detected tree crown; no species attribute (simple lon/lat csv)

</v-clicks>

<!--
- This source solves the opposite problem from inventory. It gives broad spatial coverage, even in places where no city inventory exists.
- The model uses multispectral aerial imagery, including visible and near-infrared bands, because vegetation has a different spectral signature than pavement, roofs, or bare ground.
- The CNN is fully convolutional, so instead of classifying one image at a time, it scans imagery and produces a heatmap of likely tree crown centers.
- Peaks in that confidence map are converted into point coordinates. For RUFA, the useful output is simple: longitude and latitude for detected trees.
- The tradeoff is that the detector sees tree crowns, not species labels. It cannot tell whether a point is ash, oak, palm, or maple.
- So detected points are excellent for spatial metrics like tree density and trees per capita, but they cannot directly support TD-50 or species evenness.
- This sets up the core data story: inventory gives rich attributes with uneven coverage; CNN detection gives broad coverage with sparse attributes.
-->

---

# Why Both Sources Together

<v-clicks>

- **Inventory alone**: species-rich data, but many trees and many cities are simply missing
- **Detection alone**: comprehensive spatial coverage, but no species, size, or taxonomy
- **Combined**: RUFA can compute all four score metrics even in partially inventoried cities

</v-clicks>

<v-click>

The join works through **point-in-polygon spatial assignment**: detected coordinates get associated with place, ZIP, and tract boundaries, then enriched by any overlapping inventory records.

</v-click>

<!--
- The complementarity here is important to understand.
- Canopy cover percentage and trees per capita can be estimated from detected points alone — you just need locations and a population figure.
- But TD-50 species diversity and tree evenness require species counts, which only come from the inventory.
- So in a city with good inventory coverage, all four metrics are computed from inventory data.
- In a city with poor inventory but good aerial detection, you still get the spatial metrics, but diversity scores are limited or absent.
- The system is designed to degrade gracefully rather than refuse to compute a score entirely.
-->

---

# Score Formula

Each metric is binned 1–5 against the California distribution, then combined:

$$
\text{RUFA Score} = \left(\frac{\text{CCP}_{\text{bin}} + \text{TD}_{\text{bin}} + \text{TE}_{\text{bin}} + \text{TPC}_{\text{bin}}}{20}\right) \times 100
$$

<v-clicks>

- **Denominator 20**: four metrics × maximum bin value of 5
- **Equal weighting**: no metric dominates the composite score
- A score of **100** means top-quintile performance on all four metrics in California
- A score of **25** means bottom-quintile on all four — a signal for intervention

</v-clicks>

<!--
- The formula is deliberately simple.
- The binning step is what makes it powerful: instead of comparing raw canopy percentages across cities of wildly different sizes and geographies, you compare each city's canopy against the California distribution and assign it a 1-through-5 bin.
- A city in the second quintile on canopy gets a 2, regardless of whether that means 12% canopy or 35%.
- That normalization is what makes scores comparable across cities like Los Angeles and a small Central Valley town.
- The equal weighting is a design choice — the thesis discusses how you could weight the metrics differently for specific policy goals, but equal weighting keeps the system interpretable and neutral.
-->

---

# TD-50 — The Diversity Metric

TD-50 = the number of species needed to account for **50% of all trees** in a place.

<v-clicks>

- **Small TD-50** (e.g., 2): two species make up half the forest — high ecological dominance, high vulnerability
- **Large TD-50** (e.g., 20): many species share abundance — distributed risk, higher resilience
- **Pest vulnerability example**: a city where *Fraxinus* (ash) makes up 40% of trees is one Emerald Ash Borer outbreak away from catastrophic canopy loss

</v-clicks>

<v-click>

```python
def calculate_td_50(species_counts: dict) -> int:
    sorted_counts = sorted(species_counts.values(), reverse=True)
    total = sum(sorted_counts)
    cumulative, i = 0, 0
    while cumulative < total * 0.5:
        cumulative += sorted_counts[i]
        i += 1
    return i  # species count for top 50%
```

</v-click>

<!--
- The TD-50 metric was introduced by Natalie Love
- 2022 specifically for California urban forest contexts.
- The algorithm is O(n log n) overall — the dominant step is the descending sort by abundance — and then O(n) for the cumulative scan.
- The return value is an integer: how many distinct species did you have to count, in order from most to least abundant, before you hit 50% of all trees? A return value of 2 is a red flag.
- A return value of 20 means the forest is well-diversified.
- In the clustering section, we will see this same function applied per Voronoi cell, which gives you a spatial diversity heat map rather than just a city-level number.
- I should also note that this algorithm is re-used in both the score pipeline and the visualization pipeline — the same function runs at two different geographic scales.
-->
