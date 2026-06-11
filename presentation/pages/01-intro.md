---

# The Problem

Urban and Community Forestry programs need to answer:

<div class="grid grid-cols-2 gap-4 mt-6">
<div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">How much tree canopy does this city have?</div>
<div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">Are trees equitably distributed?</div>
<div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">Is the species mix resilient?</div>
<div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">Which cities need intervention?</div>
</div>

<div class="mt-8 text-left text-base opacity-90 max-w-4xl mx-auto">

**Why measurement matters**

- California has among the **lowest urban tree cover per capita** in the U.S.; schoolyards average **~6.4%** median canopy *(Green Schoolyards America, 2022–2024)*
- **~85%** of California elementary schools **lost tree canopy** (2018–2022), with severe losses in the Central Valley *(UC Davis, 2023)*
- California ranks among the **highest tree cover losses** nationwide — wildfire, drought, and pests *(Urban Forestry & Urban Greening, 2021–2024)*

</div>

<!--
Before diving into the technicals, we first must define the problem.

Effective urban forest management begins with measurement. California expanded its Urban and Community Forestry programs in response to these trends — but planners still need standardized answers to the questions on this slide.
-->

---

# Enter RUFA:

<div class="grid grid-cols-2 gap-6 items-center mt-2">
<div class="text-left text-lg">

- **RUFA Score (0–100)** for every California city
- **Interactive map** at city, ZIP, and tract levels
- **Compare** forest health across **702+** places statewide

</div>
<div>

<img src="../public/site_preview.gif" class="rounded shadow max-h-[52vh] max-w-full object-contain mx-auto" />

</div>
</div>

<!--
Enter RUFA.

RUFA is the UFEI dashboard behind these questions. A planner selects a city and gets a composite RUFA Score plus the four metrics that drive it — canopy, density, diversity, and evenness.

The map supports zoom from statewide view down to ZIP codes and census tracts, so you can compare forest health across California's census-designated places on a common scale.

The rest of the talk covers the data pipeline and engineering that make this dashboard work at scale.
-->

---

# Why RUFA?

RUFA begins with **two complementary datasets**:

- **Inventory data**: species, size, and location from municipal and arborist records
- **Detector data**: tree coordinates from CNN analysis of aerial imagery

Together they support a single platform for planners and stakeholders.

<div class="mt-6 text-lg font-semibold text-green-700">
This thesis contributes the engineering underneath: sub-second database queries over the full inventory, and multi-scale map clustering over millions of points.
</div>

<!--
So why RUFA, and how does it help solve the problem on the previous slide?

RUFA starts from two data sources we already had: detailed inventory records and broad detector output from aerial imagery. Inventory gives species and structure where cities surveyed trees; detection fills spatial gaps with coordinates across the state.

Planners still need practical answers — canopy, equity, diversity, and where to intervene — on a common scale across California.

My contribution focused on the system architecture that makes large-scale urban forestry analysis practical

optimizing queries across more than seven million records and developing multi-scale spatial clustering techniques that maintain interactive performance across all map scales.
-->

---

# Agenda

1. **Data Sources + System Design:** Inventories, CNN detection, and RUFA Score
2. **Database Design:** spatial indexing, schema, and materialized views
3. **System Design:** AWS architecture and data flow
4. **Clustering:** K-means to SuperCluster and Voronoi

<!--
- I'm going to walk through five sections today
- If you have questions as we go, feel free to hold them until the end or interrupt if something is unclear.
-->

---
class: text-lg
---

# The Four Score Metrics

Each component is binned **1–5** against the California distribution, then combined:

$$
\text{RUFA Score} = \left(\frac{\text{CCP}_{\text{bin}} + \text{TD}_{\text{bin}} + \text{TE}_{\text{bin}} + \text{TPC}_{\text{bin}}}{20}\right) \times 100
$$

<div class="text-base opacity-80 mt-2">
Equal weight · **0–100** scale · **100** = top quintile on all four · **25** = bottom quintile on all four
</div>

<div class="grid grid-cols-2 gap-x-6 gap-y-2 text-base text-left max-w-3xl mx-auto mt-6">
<div><strong>CCP</strong> — Canopy Cover Percentage</div>
<div><strong>TPC</strong> — Trees Per Capita</div>
<div><strong>TD</strong> — Tree Diversity (TD-50 index)</div>
<div><strong>TE</strong> — Tree Evenness</div>
</div>

<!--
So that score from earlier is defined via the canopy cover percentage, trees per capita, tree diversity, and the tree evenness metrics

each metric is ranked against every other place in California, binned from 1 to 5, and those four bins combine into a score out of 100.

That is why a Sacramento planner can look at 68, Fresno at 55, and Stockton at 71 and know immediately who is ahead — and drill into whether the gap is canopy, density, diversity, or uneven distribution before zooming to a ZIP code.

The next four slides define each metric a little further
-->

---
class: text-lg
---

# Canopy Cover (CCP)

**Canopy Cover Percentage** — proportion of land area covered by tree canopy

- **How:** aggregate CNN-detected tree locations and canopy extent within each place, ZIP, or tract boundary (Ventura et al.)
- **Why it matters:** shade and urban cooling, stormwater retention, reduced heat island effect

<!--
CCP is primarily an aerial-detection metric. The Ventura pipeline identifies individual trees and their canopy extent from multispectral imagery; RUFA sums that coverage as a percentage of land area inside each boundary.

Inventory canopy spread fills gaps where cities have ground-truthed measurements. This is coverage, not species composition — it answers how much of the city is actually under canopy.

This is valuable as it provides shade, urban cooling, and reduced heat island effects
-->

---
class: text-lg
---

# Trees Per Capita (TPC)

**Trees Per Capita** — tree count relative to population size

- **How:** total trees in a geographic area ÷ U.S. Census population for that area
- **Tree count:** California Urban Forest Inventory records **plus** Ventura detections, deduplicated and assigned to the place, ZIP, or tract polygon
- **Population:** census `human_population` stored in the RUFA database, propagated to ZIP and tract rollups

<!--
trees per capita is a density-equity metric, not total tree count. A large city can have many trees but still score low if the population is high and canopy is uneven.

Both data sources feed the numerator; census data feeds the denominator. That pairing is what lets RUFA compare a Central Valley town to Los Angeles on the same scale.

human population derives from recent census data
-->

---
class: text-lg
---

# Tree Diversity (TD-50)

**Tree Diversity (TD)** — TD-50 index (Love et al., 2022)

- **How:** rank species by abundance; count how many species are needed to reach **≥ 50%** of all trees in the area
- **Source:** inventory species counts only — the detector provides coordinates, not species labels
- **Low TD-50** (e.g., 2): a few species dominate — high vulnerability to pests and disease
- **High TD-50** (e.g., 20): abundance spread across many species — greater ecological resilience

<!--
TD-50 is the smallest k where the cumulative share of the k most abundant species reaches half the forest. 

RUFA evaluates this from manual inventory records assigned to each geographic unit at refresh time.

If an Arizona Ash makes up 40% of a city's trees, TD-50 stays low and one pest outbreak can wipe out a large share of canopy. 

Higher TD-50 means no single species carries disproportionate risk.
-->

---
class: text-lg
---

# Tree Evenness (TE)

**Tree Evenness** — how uniformly trees are distributed **spatially** within a place

- **How:** subdivide the place into **census tracts**; compute tree density (trees/km²) in each block
- **TE = standard deviation** of those block-level densities
- **Higher TE → less even access** — blocks deviate further from the city-wide average density
- **Example:** TE of 250 means blocks typically sit about **±250 trees/km²** above or below the city average

<!--
TE is not species evenness. 

It measures whether residents across neighborhoods experience similar tree density, or whether trees cluster in one part of the city.

A place can have acceptable city-wide trees per capita but high tree evenness if canopy concentrates in parks or wealthier districts. TE surfaces that spatial imbalance inside the boundary using the census tract data.
-->

---
class: text-lg
---

# System Pipeline

```
Inventory Records          CNN Aerial Detection
(species, size, location)  (detected tree coordinates)
         │                          │
         └──────────┬───────────────┘
                    ▼
         Point-in-Polygon Assignment
         (R-tree spatial lookup)
                    │
         ┌──────────┴──────────────┐
         ▼                         ▼
   Relational Schema          Materialized Views
   (3NF, MySQL/MariaDB)       (cached score rollups)
         │
         ▼
   RUFA Score (0–100)
   per city / ZIP / tract
         │
         ▼
   Interactive Map
   (SuperCluster + Voronoi)
```

<!--
explain diagram
-->

---

# AWS Architecture Overview

<div class="flex justify-center mt-4">
<img src="../public/rufa_system.png" class="max-h-[35vh] max-w-full rounded shadow object-contain" />
</div>

<!--
explain diagram from UI to data
-->
