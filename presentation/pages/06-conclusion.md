---
layout: section
---

# Conclusion

Contributions · Questions

<!--
To wrap up: this thesis is not the RUFA Score formula itself but engineering underneath.

On the data side, we normalized inventory and detector records, assigned them to census boundaries with R-tree pruning plus containment checks, and precomputed scores in materialized views so the dashboard reads indexed tables instead of running heavy aggregations live.

The benchmark showed indexed MySQL bringing city queries under two hundred milliseconds.

On the map side, we reproduced the clustered BigQuery starting from K-means, learning where it broke, and landing on SuperCluster.

Together, that is a system that ingests two data sources, computes the four-component RUFA Score, and serves it interactively at statewide scale.

If asked about next steps: AllomeTree for structural estimates on aerial-only trees, incremental refresh as inventory grows, mobile field capture, and geographic expansion — the architecture is not California-specific.

Thank you — happy to take questions, or walk through the live demo.
-->

---

# References

<div class="grid grid-cols-2 gap-6 text-[0.58rem] leading-3.5 mt-2">
<div>

- Cattell, R. (2011). Scalable SQL and NoSQL data stores. *ACM SIGMOD Record*.
- Codd, E. F. (1970). A relational model of data for large shared data banks. *Communications of the ACM*.
- Finkel, R. A., & Bentley, J. L. (1974). Quad trees: A data structure for retrieval on composite keys. *Acta Informatica*.
- Guttman, A. (1984). R-trees: A dynamic index structure for spatial searching. *SIGMOD*.
- Livesley, S. J., McPherson, E. G., Calfapietra, C., et al. (2016). The urban forest and ecosystem services. *Journal of Urban Ecology*.
- Love, N. L. R., & McPherson, E. G. (2022). Diversity and structure in California's urban forest. *Urban Forestry & Urban Greening*.
- McPherson, E. G., Berry, A. M., Studer, S. M. L., Chen, W., & van Doorn, N. S. (2018). Performance testing to identify climate-ready trees. *Arboriculture & Urban Forestry*.

</div>
<div>

- Niemeyer, G. (2008). *Geohash*. http://geohash.org
- PostGIS Project. (2005). *PostGIS Documentation*. https://postgis.net
- Samet, H. (2006). *Foundations of Multidimensional and Metric Data Structures*. Morgan Kaufmann.
- Sanusi, D. J., et al. (2016). Street orientation and side of street influence microclimatic benefits of street trees. *Journal of Environmental Quality*.
- Veach, E., et al. (2017). *S2 Geometry Library*. https://s2geometry.io
- Ventura, J., Pawlak, C., Honsberger, M., et al. (2022). Individual tree detection in large-scale urban environments using high-resolution multispectral imagery. *arXiv*.
- Xiao, Q., & McPherson, E. G. (2016). Surface water storage capacity of twenty tree species in Davis, California. *Journal of Environmental Quality*.

</div>
</div>
