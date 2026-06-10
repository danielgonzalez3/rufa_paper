---

# Agenda

1. **Introduction:** urban forest assessment problem
2. **Data Sources:** inventories, CNN detection, and RUFA Score
3. **Database Design:** spatial indexing, schema, and materialized views
4. **System Design:** AWS architecture and data flow
5. **Clustering:** K-means to SuperCluster and Voronoi

<!--
- I'm going to walk through five sections today
- If you have questions as we go, feel free to hold them until the end or interrupt if something is unclear.
-->

---

# The Problem

Urban and Community Forestry programs need to answer:

<div class="grid grid-cols-2 gap-4 mt-6">
<v-click><div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">How much tree canopy does this city have?</div></v-click>
<v-click><div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">Are trees equitably distributed?</div></v-click>
<v-click><div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">Is the species mix resilient?</div></v-click>
<v-click><div class="p-4 rounded bg-green-800 text-white text-center flex items-center justify-center min-h-20">Which cities need intervention?</div></v-click>
</div>

<!--
The project really spawns from forest services

According to Green Schoolyards America (2022–2024), California has some of the lowest urban tree cover per capita in the U.S., with schoolyards averaging about 6.4% median canopy.

A UC Davis study (2023) covering 2018–2022, ~85% of California elementary schools lost tree canopy, with severe losses in the Central Valley.

Urban Forestry & Urban Greening research (2021–2024), California has had some of the highest tree cover losses in the U.S., driven by wildfire, drought, and pests.

Due to this California expanded its Urban and Community Forestry programs to increase tree planting and maintenance in cities and schoolyard

Though in order to solve such a broad problem as urban forestry you first need to measure it

Some of the questions looks to solve is

Tree Canopy Coverage

How are tree distributed

Is this set of tree data mix resilient?

Which cities need intervention? where Should our efforts go
-->

---

# The Four Score Metrics

RUFA bins each metric **1–5** based on its California-wide distribution:

<v-clicks>

- **Canopy Cover Percentage (CCP)** — proportion of the city's land area under tree canopy; higher canopy correlates with lower urban heat island effect and better stormwater management
- **Trees Per Capita (TPC)** — total detected trees divided by census population; measures equity and urban forest density
- **Tree Diversity (TD-50)** — number of species required to account for 50% of all trees; measures ecological resilience
- **Tree Evenness (TE)** — how uniformly tree counts are distributed across represented species; low evenness means a few species dominate

</v-clicks>

<!--
- Each of these metrics captures a different dimension of forest health.
- CCP tells you about physical coverage — is there enough canopy to provide shade and cooling?
- TPC tells you about distribution — does everyone have access to trees, or are they concentrated in wealthy neighborhoods?
- TD-50 and TE together describe the species composition — a forest can have many species but still be dominated by one, and TE catches that.
- By combining all four into a single score, RUFA gives you a single number for comparison while still preserving the ability to drill into which dimension is the problem.
-->

---

# What RUFA Produces

A single, repeatable pipeline from raw data to actionable output:

<div class="grid grid-cols-2 gap-4 mt-6">
<div><strong>Score:</strong> 0–100 composite for each California census-designated place</div>
<div><strong>Updates:</strong> automatic refresh when inventory data changes</div>
<div><strong>Rollups:</strong> city, ZIP code, and census tract levels</div>
<div><strong>Map:</strong> multi-scale tree density and diversity layers</div>
</div>

<!--
- Additionally we are going to produce a RUFA Score allowing city planners to compare their cities overall urban forestry health to other cities 
- When I say RUFA produces a score, I want to be clear about what that means in practice.
- A planner in Sacramento can look at their city's score — say, 68 out of 100 — and immediately compare it to Fresno or Stockton on the same scale.
- They can drill into which of the four component metrics is dragging the score down.
- They can see which ZIP codes inside their city are underperforming.
- And when they take action — planting trees, updating the inventory — the score updates automatically through the refresh pipeline we will talk about in the database section.
- The score is the output, but the data infrastructure is the actual contribution.
-->

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
- (free ball))
- This diagram is the through-line for the entire talk.
- Every section we cover maps onto one stage of this pipeline.
- Raw inventory records and CNN-detected tree coordinates come in from the left.
- They get spatially assigned to geographic boundaries using an R-tree lookup — that is the database design section.
- Scores are computed and cached in materialized views — also the database section.
- Those scores get served to the dashboard.
- And the millions of individual tree points get clustered so the map is usable — that is the clustering section.
- Keep this mental model in mind as we go through each piece.
-->

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
- This diagram is the through-line for the entire talk.
- Every section we cover maps onto one stage of this pipeline.
- Raw inventory records and CNN-detected tree coordinates come in from the left.
- They get spatially assigned to geographic boundaries using an R-tree lookup — that is the database design section.
- Scores are computed and cached in materialized views — also the database section.
- Those scores get served to the dashboard.
- And the millions of individual tree points get clustered so the map is usable — that is the clustering section.
- Keep this mental model in mind as we go through each piece.
-->
