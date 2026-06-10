---
layout: section
---

# Database Design

Proximity search · Spatial indexing · Schema · Materialized views · Performance

---

# The Scale Challenge

RUFA's database is not just a data store — it is the performance backbone.

<div class="grid grid-cols-2 gap-4 mt-6">
<div><strong>7.2M</strong> tree detector records</div>
<div><strong>43M</strong> tree inventory records</div>
<div><strong>702</strong> city boundaries plus ZIPs and tracts</div>
<div>Every tree needs place, ZIP, and tract assignment, though census place is the primary key</div>
<div>Dashboard reads target <strong>&lt; 200 ms</strong></div>
</div>

Every design decision — indexing strategy, normalization level, refresh mechanism flows from these five requirements.

<!--
- Before diving into the specific techniques, it is worth establishing what the database actually has to do.
- It is not enough to just store records correctly — it has to support three very different workloads simultaneously.
- First, spatial ingestion: assigning millions of points to polygons.
- Second, analytical aggregation: computing canopy cover, TD-50, and the other metrics over those assignments.
- Third, low-latency serving: answering dashboard queries in under 200 milliseconds.
- A naive relational design can handle any one of these, but handling all three at once requires careful design.
- The 200ms target comes directly from the indexed benchmark result I will show at the end of this section.
-->

---

# Naive Proximity Search

The simple idea: draw a box around the area we care about, then ask the database for trees inside it.

<div class="grid grid-cols-3 gap-4 mt-6">
<div class="p-4 rounded bg-green-50">
<strong>1. Draw a box</strong><br />
Limit the search to a rough area.
</div>
<div class="p-4 rounded bg-green-50">
<strong>2. Check distance</strong><br />
Remove points outside the radius.
</div>
<div class="p-4 rounded bg-green-50">
<strong>3. Return matches</strong><br />
Show trees near the target.
</div>
</div>

<div class="mt-5">
This works for small data. At RUFA scale, the box can still include a huge number of trees, so the database may still check far too many rows.
</div>

<!--
- RUFA at it's core boils down to a problem of proximity search
- we are given sets of dynamics points and are looking to see if they fit within a static polygon

- The baseline approach for proximity search and what most reach for first is a bounding box or circle.
- The bounding box is cheap to express as a range query on lat and lon columns.
- At small scale — say, a single city's trees — this works fine.
- At RUFA scale, step 2 is the bottleneck.
- In a dense city like Los Angeles, a bounding box around a neighborhood could contain tens of thousands of candidate rows before ST_Distance gets a chance to filter.
- If the lat and lon columns have ordinary B-tree indexes, each index prunes one dimension but not both simultaneously.
- The result is that even a moderately selective lat range still leaves a huge candidate set for the lon filter to scan.
- That is the 1D index saturation problem.
- This is whats unique to the rufa problem as most indexes in databases design revolve optimization around a single field
-->

---

# Why Separate Lat/Lon Indexes Fail

Adding indexes on latitude and longitude helps, but it does not fully solve the problem.

The database can quickly find a latitude range or a longitude range. But a map search is really asking a two-dimensional question: "which trees are inside this shape?"

That is why RUFA needs a real **spatial index**.

Instead of searching rows one by one, the database first narrows the map down to a small candidate area.

<!--
- This is a subtle but important point.
- B+ tree indexes are designed for ordered 1D data.
- They are excellent at answering "give me all rows where latitude is between 34.0 and 34.1" — that is a single contiguous range in the index.
- But the geographic query you actually want — "give me all rows within 500 meters of this point" — is a 2D circle, not a 1D range.
- You can decompose a circle into lat and lon ranges, but the intersection of two B-tree range scans is inefficient in most query planners.
- The engine will use one index, scan that result set, and then apply the other condition as a filter on the already-fetched rows.
- For a city like San Francisco with enormous tree density per square kilometer, even a narrow lat band contains enough rows to make this painful.
- Spatial indexes solve this by encoding both dimensions into a single index entry.
-->

---

# Two Families of Spatial Index

<div class="grid grid-cols-2 gap-10 mt-6">
<div>

**Hash-Based**

Split the map into boxes, like a grid. Records get a bucket key.

Fast and easy. Cell size is the key tradeoff — too big and you over-include, too small and you fragment.

**Example: Geohash**

</div>
<div>

**Tree-Based**

Organize space hierarchically. Each level subdivides the level above.

Better for irregular density — denser areas get more subdivision automatically.

**Examples: Quadtree, R-tree**

</div>
</div>

<!--
- There are two big families of spatial index, and they take fundamentally different approaches.
- Hash-based: divide the world into a fixed grid and bucket records into cells. Geohash is the classic example.
- Tree-based: build a tree where each node represents a region of space, and subdivide regions as needed. Quadtree and R-tree both work this way.
- The next two slides go into Geohash and Quadtree because they illustrate the tradeoffs cleanly. Then we'll land on R-tree, which is what RUFA actually uses for polygon membership.
-->

---

# Geohash — Hash-Based

Encode a coordinate into a short string. Nearby strings = nearby places.

<div class="grid grid-cols-[1fr_0.9fr] gap-5 mt-4 items-start">
<div>

<div class="grid grid-cols-2 gap-4">
<div class="p-4 rounded bg-green-50">
<strong>Easy to implement</strong><br />
Just a string prefix lookup. Works in any database.
</div>
<div class="p-4 rounded bg-green-50">
<strong>Fixed precision per level</strong><br />
Each character of the string halves the cell size.
</div>
</div>

<div class="mt-4">

**Bottom-up search**: start small, expand outward until you find what you need.

> Example: "find a gas station within 10–20 miles" -> Expand the geohash precision until you've gathered enough candidates, or the radius is exhausted.

</div>

</div>
<div>

<img src="../public/geo_hash.webp" class="rounded shadow max-h-[19rem] object-contain mx-auto" />

</div>
</div>

<!--
- Geohash is the simplest hash-based spatial index. It turns a lat/lon pair into a short string
- The clever property is that strings that share a prefix are geographically close. "ABAB" is a neighborhood of "ABAA".
- That means proximity queries become string prefix lookups, which any database with a B-tree on a string column can do.
- The implementation simplicity is the main selling point. There's no special index type, no spatial library — just a string column and a regular index.
- The fixed-precision-per-level property means you choose how granular your buckets are. Each additional character roughly halves the cell dimensions.
- This is intuitive for "find me the nearest N things" queries: keep widening the bucket until N things are inside.


The main issues with this approach

Nearby points can fall into different GeoHashes, so you miss close matches unless you also check neighboring cells for example AC and CA
Cells aren’t consistent in real-world distance due to Earth curvature and latitude effects
Fixed precision levels, GeoHash uses discrete lengths which may miss points when applying polygons
-->

---

# Quadtree — Tree-Based

Recursively split each region into four quadrants. Subdivide only where data is dense.

<div class="grid grid-cols-[1fr_0.9fr] gap-5 mt-4 items-start">
<div>

<div class="grid grid-cols-2 gap-4">
<div class="p-4 rounded bg-green-50">
<strong>Adapts to density</strong><br />
Empty regions stay as big cells. Dense urban cores get finely subdivided automatically.
</div>
<div class="p-4 rounded bg-green-50">
<strong>Slightly harder to implement</strong><br />
Needs a real tree structure, not just a string column.
</div>
</div>

<div class="mt-4">

**Top-down search**: start with the whole map, descend into smaller quadrants until you have enough matches.

> Example: "find businesses near a point" -> Start at the county level, drill down into the relevant quadrant, keep splitting until the cell contains a useful candidate set.

</div>

</div>
<div>

<img src="../public/quad_tree.webp" class="rounded shadow max-h-[19rem] object-contain mx-auto" />

</div>
</div>

<!--
- Quadtree is the canonical tree-based spatial index. The idea is simple: take the whole map as one cell, split it into four equal quadrants, and recursively split any quadrant that has too many records in it.
- The result is a tree where empty rural areas stay as one big cell at the top, but dense city centers get split many times into very small cells at the bottom.
- This density-adaptive behavior is the key advantage over fixed-grid approaches like geohash. Los Angeles and a rural county can coexist in the same index without forcing a compromise on cell size.
- The implementation cost is higher because you need an actual tree data structure with parent-child pointers and rebalancing logic when records get inserted or deleted.
- Search is top-down: start at the root, find which quadrant contains the query point, descend into that quadrant, repeat until the cell is small enough to enumerate.
- For "find k nearest things" you keep descending until you have at least k candidates in your cell and possibly its neighbors.
- Quadtrees are great for points. For polygons — which is what RUFA actually needs — R-trees do this better, which is the next slide.
-->

---

# R-tree — RUFA's Choice

Each city, ZIP, and tract gets a **rectangle** drawn around it. The R-tree groups nearby rectangles into bigger rectangles.

When RUFA places a tree on the map, the R-tree quickly rejects rectangles that are nowhere near that tree.

<div class="grid grid-cols-[1fr_0.9fr] gap-5 mt-4 items-start">
<div>

<div class="grid grid-cols-2 gap-4">
<div class="p-4 rounded bg-green-50">
<strong>Built for polygons</strong><br />
Cities, ZIPs, and tracts are real shapes — not points. R-trees were designed for exactly this.
</div>
<div class="p-4 rounded bg-green-50">
<strong>Native in MySQL</strong><br />
Mark a column `SPATIAL` and MySQL builds an R-tree automatically.
</div>
</div>

</div>
<div>

<img src="../public/r_tree.webp" class="rounded shadow max-h-[19rem] object-contain mx-auto" />

</div>
</div>

<!--
- The R-tree is the spatial index RUFA actually uses, and the reason is that RUFA's main spatial workload is polygon membership: assigning a tree point to its containing city, ZIP, and tract.
- Quadtrees and geohashes are good for points. R-trees are good for shapes.
- Each polygon — say, the boundary of San Francisco — gets a minimum bounding rectangle drawn around it. That rectangle is what the R-tree stores.
- The tree then groups nearby polygon rectangles into bigger parent rectangles, and groups those into even bigger grandparent rectangles, all the way up to the root.
- When you query with a tree point, the R-tree walks down from the root and rejects any parent rectangle that doesn't contain the point. By the time you reach the leaves, you have a tiny candidate set.
- The other major reason for R-tree: MySQL supports it natively. You declare a column as SPATIAL and the engine builds the R-tree behind the scenes. No external library, no schema gymnastics.
-->

---

# RUFA Spatial Lookup — Two Phases

<div class="grid grid-cols-2 gap-6">
<div>
<h3>1. Fast guess</h3>
Use the R-tree to find the few city, ZIP, or tract polygons that might contain the tree.
</div>
<div>
<h3>2. Exact check</h3>
Use `ST_Contains` to confirm the tree is actually inside the polygon.
</div>
</div>

This gives RUFA both speed and correctness: it avoids scanning everything, but still handles messy real-world boundaries.

<!--
- The two-phase strategy is the most important algorithmic idea in the database section.
- Phase one is fast but approximate: the MBR of a polygon is larger than the polygon itself, so you can get false positives where a point is inside the MBR but outside the actual polygon.
- That is fine — the R-tree is not meant to give you exact answers, it is meant to give you a very small candidate set very quickly.
- Phase two is slow but exact: ST_Contains runs the actual point-in-polygon algorithm on the polygon's true boundary.
- Because you are only running it on 2 to 5 candidates rather than 702, the total cost is tiny.
- The correctness benefit is significant: California has many irregular polygon shapes — coastal cities, cities with holes, census tracts that follow highways.
- Without exact containment testing, you would misassign trees near boundaries.
- With the two-phase approach, you get both speed and accuracy.
-->

---

# Relational Schema

The schema separates raw data, geographic boundaries, and precomputed results.

<div class="text-[0.95em] leading-7 mt-4">

| Table | Role |
|---|---|
| `rufa_place`, `rufa_zip`, `rufa_tract` | polygon geometry |
| `tree_inventory` | raw tree records |
| `tree_detected` | CNN tree coordinates |
| `tree_stats_by_location` | ready-to-display scores |
| `mv_refresh_schedule` | what needs to be updated |
| `mv_version_control` | which version is live |

</div>

<!--
- The normalization in this slide is not just academic tidiness. It is what makes the refresh system work correctly.
- If place metadata and tree statistics were denormalized into one big table, updating a city's population count would require touching every tree record for that city.
- By separating the geographic boundary tables from the statistics tables from the inventory tables, updates are isolated and the cascade logic becomes tractable.
- The first three tables hold raw data — polygons, inventory points, detected points.
- The next table — tree_stats_by_location — is the precomputed serving layer that the dashboard actually reads from.
- The last two tables are operational metadata: which rows are stale, which version is currently being served.
- The next slide shows how all of these connect.
-->

---

# Schema Diagram

<div class="flex justify-center mt-1">
<img src="../public/rufa_database_erd_fit.png" class="max-h-[37vh] max-w-full rounded shadow object-contain" />
</div>

<!--
- This is the ERD from Chapter 3 of the thesis.
- Walk through it left to right: inventory and detector points on the far left feed into the system as raw data.
- In the center, the rufa_city table is the human-facing view of a place — it stores the precomputed stats like td_50, trees_per_km, trees_per_capita.
- The place, zip, and tract tables on the right hold the actual polygon geometry that the R-tree indexes.
- The two intersection tables in the middle — rufa_place_zip_intersection and rufa_place_tract_intersection — encode which ZIPs and tracts belong to which place. These are the relationships the cascade logic walks when a city's data changes.
- At the bottom, the zip_metrics and tract_metrics tables hold the precomputed scores for those finer geographies.
- The whole structure is normalized to 3NF — every fact lives in exactly one place, and updates propagate through the intersection and metrics tables in a controlled way.
-->

---

# Why MySQL / MariaDB

RUFA uses MySQL/MariaDB because it fits the existing Selectree system.

PostGIS would be stronger for advanced GIS work, but switching databases would mean rewriting existing tools and pipelines.

The important point: MySQL still supports the spatial index RUFA needs for fast polygon lookup.

<!--
- This is an honest engineering tradeoff.
- PostgreSQL with PostGIS is objectively more capable for spatial workloads.
- It has better index types, a much richer spatial function library, native coordinate reference system transformations, and true materialized views with concurrent refresh.
- MongoDB with geospatial indexes would give you schema flexibility for heterogeneous tree attributes.
- Oracle Spatial and ArcGIS Enterprise exist for enterprise use.
- But all of those require migrating away from the existing infrastructure, which means rewriting every Selectree integration, every data pipeline, and retraining every user.
- RUFA's thesis is that you can deliver a high-quality spatial system within the MySQL constraint — and the benchmark at the end of this section shows that the 95% performance improvement achievable with MySQL's spatial indexing is sufficient for the dashboard use case.
-->

---

# Materialized Views

RUFA should not recompute scores every time someone opens the dashboard.

For one city, the score needs several expensive metrics:

<div class="grid grid-cols-2 gap-4 mt-4">
<div class="p-3 rounded bg-green-50">Canopy cover</div>
<div class="p-3 rounded bg-green-50">Trees per capita</div>
<div class="p-3 rounded bg-green-50">TD-50 diversity</div>
<div class="p-3 rounded bg-green-50">Tree evenness</div>
</div>

Instead, RUFA computes the scores ahead of time and stores them in `tree_stats_by_location`.

The dashboard reads the saved answer quickly. The expensive recalculation happens later, in the background.

<!--
- A materialized view is a cached query result stored as a table.
- PostgreSQL has native support for them with REFRESH CONCURRENTLY, which lets you rebuild the cached table without locking out readers.
- MariaDB does not have this feature.
- What RUFA does instead is maintain regular tables that hold the same pre-computed results, and then manages the refresh lifecycle explicitly: a trigger marks a row as stale when underlying data changes, a batch job recomputes the stale rows on a schedule, and an atomic table swap publishes the new results.
- This is more operational complexity than native materialized views, but it achieves the same outcome — dashboard queries read from fast, pre-computed tables rather than running expensive aggregations on the fly.
-->

---

# Geographic Cascade Architecture

When one place changes, nearby summary levels may need to change too.

Example: if a city's tree inventory changes, RUFA may also need to update its ZIP codes, census tracts, and county totals.

The `location_relationships` table stores those connections so updates can cascade automatically.

<!--
- The cascade requirement comes from how the dashboard presents data.
- A user looking at a map can see city-level scores, then zoom into ZIP codes, then census tracts — all on the same screen.
- If a city's tree inventory was just updated but the ZIP scores haven't refreshed yet, the user would see inconsistent numbers: the city score says 72 but the ZIPs inside it average to 58.
- That mismatch destroys trust in the data.
- The location_relationships table is what prevents this: it is a pre-computed mapping from "this city" to "all the ZIPs and tracts that intersect it," built at database setup time.
- When the refresh stored procedure processes a stale city, it looks up all related geographic levels in location_relationships and marks them for refresh too.
- The cascade is automatic and the dashboard always sees a logically consistent snapshot.
-->

---

# Refresh Strategy

Refresh pipeline: **mark stale → recompute → publish**.

<div class="grid grid-cols-3 gap-4 mt-6">
<div class="p-4 rounded bg-green-50">
<strong>1. Mark stale</strong><br />
Tree data changed here.
</div>
<div class="p-4 rounded bg-green-50">
<strong>2. Recompute</strong><br />
Batch job rebuilds scores.
</div>
<div class="p-4 rounded bg-green-50">
<strong>3. Publish</strong><br />
Dashboard gets the finished version.
</div>
</div>

<!--
- Let me walk through each layer.
- The trigger fires automatically after any INSERT into tree_inventory — it simply sets a boolean flag in mv_refresh_schedule.
- This is cheap: it is a single UPDATE to one row.
- The flag accumulates over time as inventory records come in.
- The batch job — which runs on a cron schedule — opens a cursor over all rows where needs_refresh is TRUE and calls CalculateRUFAScore for each one.
- This is where the expensive computation happens, but it happens off the critical path, not during a dashboard request.
- The atomic swap in layer three is the most technically interesting part: RENAME TABLE in MySQL acquires a metadata lock and swaps the table names in a single atomic DDL operation.
- Dashboard queries will either see the old complete table or the new complete table — they can never see a half-populated temporary table.
- This eliminates transient inconsistency entirely.
-->

---

# Performance Benchmark

Test environment: AWS EC2 **t3.large** (2 vCPU, 8 GB RAM), **MariaDB 10.5.23** on Amazon RDS.
Dataset: **7.2 million records**, 702 cities. Each scenario executed **100 times**.

| Storage Strategy | Avg. Latency | Std. Dev | Throughput |
|---|---:|---:|---:|
| S3 — single CSV file | 8,450 ms | ±1,230 ms | 0.12 q/s |
| S3 — per-city CSV files | 3,120 ms | ±480 ms | 0.32 q/s |
| MySQL table, no index | 2,890 ms | ±290 ms | 0.35 q/s |
| **MySQL table, B-tree index** | **145 ms** | **±25 ms** | **6.90 q/s** |

<v-click>

Main takeaway: putting the data in MySQL and adding the right index brings dashboard reads under the 200 ms target.

That is why RUFA precomputes the hard work, then serves simple indexed reads.

</v-click>

<!--
- Let me walk through the four strategies.
- The S3 single-CSV approach requires downloading and parsing the entire 7.2 million record file for every query — hence the 8.4 second average and the huge standard deviation, which reflects network latency variance.
- The per-city CSV files are better: you only download the relevant city's file.
- But you still pay file system overhead and CSV parsing on every request.
- The unindexed MySQL table is slightly faster than per-city CSVs because there is no file system overhead — but without an index, every city-name query is an O(n) full table scan across 7.2 million rows.
- Adding a B-tree index on the city name column brings the latency down to 145 milliseconds — a 95% reduction — because the index turns a full scan into an O(log n) tree traversal followed by a small set of row fetches.
- The standard deviation of 25 milliseconds is low, which means consistent performance even under load.
- The storage overhead of 300 megabytes for a 95% latency improvement is an obvious engineering win.
-->
