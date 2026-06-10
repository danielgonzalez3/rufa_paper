---
theme: default
title: Rapid Urban Forest Assessment
info: |
  Thesis presentation for the Rapid Urban Forest Assessment (RUFA) system.
class: text-center
transition: slide-up
---

# Rapid Urban Forest Assessment

<div class="text-xl opacity-90 mt-2">
From fragmented tree data to comparable scores across California
</div>

<div class="mt-10 text-lg opacity-70">
Daniel Gonzalez · Cal Poly · Urban Forest Ecosystems Institute
</div>

<div class="mt-4 text-base opacity-50">
Master's Thesis · Computer Science
</div>

<!--
Good afternoon. My name is Daniel Gonzalez, and I am pleased to present my master's thesis on the Rapid Urban Forest Assessment system, or RUFA.

This work was developed in collaboration with the Urban Forest Ecosystems Institute at Cal Poly San Luis Obispo. I am grateful to Professor Matt Ritter and Jenn Yost for entrusting me with the UFEI infrastructure and the RUFA project, and to Dr. Alex Dekhtyar for his guidance throughout this research. I also acknowledge the contributions of Cami Pawlak (pow-lock), who provided essential RUFA data and scoring formulas, and the San Jose State University team who developed the RUFA front end.

RUFA is a web-based platform that integrates urban tree inventories and aerial tree detection to assess forest health across California's census-designated places. UFEI's mission is to connect scientific research with practical tools for urban foresters, planners, and communities. RUFA extends that mission by aggregating disparate data sources into a standardized RUFA Score and an interactive dashboard.

This thesis addresses two engineering challenges encountered in building that platform: querying and aggregating over seven million tree records in real time, and rendering spatial summaries at multiple zoom levels without recomputing cluster assignments on every map interaction.

Today, I will present the data sources and scoring metrics that define RUFA, the database design and indexing strategies that enable sub-second queries, the AWS deployment architecture, and the clustering pipeline that supports interactive map visualization.
-->

---
src: ./pages/01-intro.md
---

---
src: ./pages/02-data-sources.md
---

---
src: ./pages/03-database-design.md
---

---
src: ./pages/04-system-design.md
---

---
src: ./pages/05-clustering.md
---

---
src: ./pages/06-conclusion.md
---
