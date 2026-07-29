---
layout: section
---

# Clustering

K-means · SuperCluster · Voronoi

---

# What UFEI Wanted on the Map

When RUFA was first proposed, UFEI already had a **Google BigQuery** dashboard with a working clustered tree map:

<div class="flex justify-center mt-3">
<img src="../public/ventura_result.gif" class="rounded shadow max-h-[48vh] max-w-full object-contain mx-auto" />
</div>

- **Zoom-aware clusters** — density summaries at state, city, and neighborhood scale
- **Per-region metrics** — tree density and species diversity visible as you explore
- **Interactive pan and zoom** — without reloading or recomputing the full dataset

<!--
When RUFA was scoped, we were pointed to the BigQuery UI UFEI was already using

where tree points aggregated into clusters as you zoomed, with enough detail to see forest patterns across California.

That prototype proved the product vision. Our job was to reproduce and extend that clustering behavior inside the new RUFA stack.
-->

---

# Why This Is Hard

<div class="grid grid-cols-2 gap-4 mt-6">
<div>Browser maps degrade around <strong>10k–50k</strong> visible elements</div>
<div><strong>500k</strong> markers is unusable at city scale</div>
<div>Overlapping dots hide spatial patterns that may be relevant</div>
<div>Clusters must update instantly on pan and zoom; no full-dataset recomputation</div>
</div>

**Requirements**: zoom-adaptive aggregation, preserved forest metrics per region, fast re-render on pan/zoom.

<!--
The BigQuery UI hid the hard part — it ran on Google's infrastructure. In RUFA we had to deliver the same visual on a MySQL-backed API with millions of points per city.

Web maps saturate around tens of thousands of rendered elements. 

At statewide scale you need regional clusters; at neighborhood scale you need local density — and switching zoom levels has to feel instant.

That is why this chapter walks through the K-means, hierarchical clustering, and finally SuperCluster with Voronoi cells solutions.
-->

---

# K-Means: First Approach

K-means groups nearby trees together by repeatedly answering one question:

> *"Which cluster center is each tree closest to?"*

<div class="grid grid-cols-3 gap-4 mt-6">
<div class="p-4 rounded bg-green-50">
<strong>1. Pick centers</strong><br />
Drop <code>k</code> starting points on the map.
</div>
<div class="p-4 rounded bg-green-50">
<strong>2. Assign trees</strong><br />
Each tree joins its nearest center.
</div>
<div class="p-4 rounded bg-green-50">
<strong>3. Move centers</strong><br />
Each center shifts to the middle of its trees. Repeat.
</div>
</div>

<div class="mt-4">
<img src="../public/k_means.gif" class="rounded shadow max-h-[28vh] max-w-full object-contain mx-auto" />
</div>

<div class="mt-4">
Works on small data. At <strong>7M trees</strong> it is too slow, and one fixed <code>k</code> cannot handle both dense downtowns and sparse suburbs at the same time (Los Angeles VS Half Moon Bay)
</div>

<!--
- K-means is the first thing most people reach for when they want to cluster points.
- The intuition is simple: you decide up front how many clusters you want — that is the k — and then you let the algorithm sort the trees into those groups.
- Step one: drop k starting points anywhere on the map. These are your initial cluster centers.
- Step two: every tree looks at all the centers and joins the one it is closest to. Now every tree belongs to a cluster.
- Step three: each cluster center moves to the average position of all the trees that joined it. Then you return to step 2 — trees reassign to their new nearest center, centers move again — until nothing changes.
- It works well on small data and the clusters look sensible. The problem is scale: with 7 million trees and several hundred clusters, each pass through the data takes a long time, and you need many passes to converge.
- The other problem is that k is fixed. If you pick k = 100, that is the same level of detail for downtown Los Angeles as it is for Half Moon Bay — but those two places need very different amounts of clustering.
- So k-means was the baseline, it taught us what we needed, and we moved on to approaches that handle scale and variable density better.
-->

---

# K-Means: Density Still Breaks It

We tried making <code>k</code> a function of tree count, so bigger cities received more clusters.

<div class="grid grid-cols-2 gap-5 mt-4">
<div>
<img src="../public/rufa_kmeans_los_angeles.webp" class="rounded shadow max-h-[20rem] object-contain mx-auto" />
<p class="text-center text-sm mt-2 opacity-75">Los Angeles: dense areas still dominate the clusters</p>
</div>
<div>
<img src="../public/rufa_kmeans_san_deigo.webp" class="rounded shadow max-h-[20rem] object-contain mx-auto" />
<p class="text-center text-sm mt-2 opacity-75">San Diego: lower-density areas need different granularity</p>
</div>
</div>

<div class="mt-4">
Tree count helps choose <code>k</code>, but density still varies inside each city. One city can contain downtown blocks, suburbs, parks, and empty areas, so K-means still over-clusters some places and under-clusters others.
</div>

<!--
- These examples show why simply scaling k as a function of the number of trees present.
- Los Angeles has very dense urban regions, so a tree-count-based k tends to spend many clusters where the data is already dense.
- San Diego has a different spatial pattern, so the same rule does not translate cleanly.
- The failure mode is not just total size; it is uneven density within the same geographic area.
- This is why RUFA moves toward clustering that adapts to zoom level and spatial density instead.
-->

---

# SuperCluster: Greedy Clustering

Greedy clustering is a simple way to group points for one map zoom level.

<div class="grid grid-cols-[1fr_0.95fr] gap-6 mt-4 items-start">
<div>

<div class="space-y-2">
<div><strong>1.</strong> Start with any unclustered point.</div>
<div><strong>2.</strong> Find nearby points within a radius.</div>
<div><strong>3.</strong> Form one cluster from those points.</div>
<div><strong>4.</strong> Pick another unclustered point and repeat.</div>
</div>

<div class="mt-5 p-4 rounded bg-green-50">
Doing this separately for every zoom level is too expensive. For zoom levels 0 through 18, RUFA would process the whole dataset 19 times.
</div>

</div>
<div>

<img src="../public/super_clustering_1.gif" class="rounded shadow max-h-[24rem] object-contain mx-auto" />

</div>
</div>

<!--
- This slide introduces the basic greedy clustering idea before explaining SuperCluster by MapBox
- For one zoom level, the approach is intuitive: take a point, gather nearby points, form a cluster, then move to the next unvisited point.
- The problem is repetition across zoom levels.
- A map usually has many zoom levels, often 0 through 18. If we run this clustering process from scratch for each zoom, the full dataset gets scanned again and again.
- Lower zoom levels also become harder because one cluster may need to represent an exponentially larger number of points.
- SuperCluster solves this by building a hierarchy once, then reusing it across zoom levels.
-->

---

# SuperCluster: Reuse the Work

SuperCluster avoids reclustering the full dataset at every zoom level.

<div class="grid grid-cols-[1fr_0.95fr] gap-6 mt-4 items-start">
<div>

<div class="space-y-3">
<div><strong>Start at high zoom:</strong> cluster the original tree points at z18.</div>
<div><strong>Move upward:</strong> group those clusters into z17 clusters, then z16, and so on.</div>
<div><strong>Use weighted centroids:</strong> bigger clusters carry more weight when the next level is formed.</div>
</div>

<div class="mt-5 p-4 rounded bg-green-50">
Each zoom level has its own spatial index, so RUFA can quickly answer: "which clusters are visible on this screen?"
</div>

</div>
<div>

<img src="../public/super_clustering_2.webp" class="rounded shadow max-h-[24rem] object-contain mx-auto" />

</div>
</div>

<!--
- Instead of running the same expensive radius search against the original tree points for every zoom level, SuperCluster reuses the previous level.
- At the most detailed zoom level, it clusters actual points. The next zoom level clusters those clusters using weighted centroids.
- Because every level has fewer inputs than the level below it, the amount of work drops quickly as the map zooms out.
- This idea is hierarchical greedy clustering, popularized by Dave Leaver's Leaflet. markercluster plugin.
- SuperCluster also builds a spatial index at each zoom level. That solves two expensive operations: finding nearby points within a radius and finding clusters inside the current map viewport.
- The result is fast enough for millions of points and good enough for interactive map browsing.
-->

---

# Voronoi Diagrams — Spatial Regions

SuperCluster gives **cluster centers**. Voronoi turns them into **interactive map regions**.

<div class="grid grid-cols-2 gap-6 items-center mt-2">
<div class="text-left text-base">

- Each cell = trees closest to one cluster center
- Per cell: **tree density** (trees/km²) and **TD-50** diversity
- Same `calculate_td_50` as the RUFA Score but now at map-cell scale
- Exported as **GeoJSON** for the dashboard

</div>
<div>

<img src="../public/vornoi_result.gif" class="rounded shadow max-h-[42vh] max-w-full object-contain mx-auto" />

</div>
</div>

<!--
Despite the SuperCluster solution, SuperCluster only produces cluster centroids, which must still be constrained to a meaningful spatial boundary.

This is where Voronoi serves as a useful complement.

In mathematics, a Voronoi diagram partitions a plane into regions based on proximity to a given set of points.

We apply this concept to the cluster centroids generated by SuperCluster using Fortune's algorithm.

Fortune's algorithm constructs the tessellation in O(n log n) time, making it practical for large-scale spatial datasets.

The resulting polygons provide a complete partition of the study area, allowing metrics to be assigned to geographic regions rather than isolated points.

By leveraging our generic polygon-based metric framework, canopy, diversity, density, and equity metrics can be calculated consistently for any generated region.

Together, SuperCluster and Voronoi provide a scalable approach for summarizing millions of trees into interpretable spatial units while preserving meaningful geographic patterns.
-->

---

# Clustering Performance and Output

**Complexity summary:**

| Approach | Complexity | 7M Trees (est.) | Multi-scale? |
|---|---|---|---|
| K-means | O(nkt) | ~140 min | No — rerun per zoom |
| Grid-based | O(n) | ~10 sec | Limited |
| Hierarchical | O(n³) | Infeasible | Yes |
| **SuperCluster + Voronoi** | **O(n log n)** | **~45 sec** | **Yes — single index build** |

Output capabilities:

- Valid **GeoJSON** supports Leaflet, Mapbox GL, and Deck.gl. 
- Density maps are color-coded per Voronoi cell, TD-50 is calculated per cell.

<!--
So where did all of that land us?

K-means was the obvious first try, but at seven million trees it would take on the order of two hours per run — and you would still re-run it every time someone zooms. Grid-based methods are fast but too coarse. Full hierarchical clustering does not scale at all.

SuperCluster plus Voronoi is the combination that actually fits RUFA. One index build, about forty-five seconds upfront, and then pan and zoom are cheap because we are reading from a precomputed hierarchy — not re-clustering the whole state on every interaction.

The output is plain GeoJSON: density and TD-50 per Voronoi cell, ready for the dashboard. That closes the loop back to what the BigQuery prototype had — but now it runs on our stack.
-->
