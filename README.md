# Developing a Rapid Urban Forest Assessment System for Sustainable City Greenification (RUFA)

A LaTeX thesis project for Cal Poly documenting RUFA — a scalable, automated urban forest assessment system that integrates aerial imagery with urban tree records to inform green infrastructure planning.

## Abstract
This project introduces an innovative approach to urban forestry management by developing an automated tree assessment system that integrates aerial imagery with urban tree records. The aim is to provide a scalable and highly accurate solution for urban planners, foresters, and environmental agencies to monitor and manage urban green spaces efficiently. The system leverages points extracted from convolutional neural networks (CNNs) combined with inventoried data to enable automated assessments and real-time data integration. The methodology includes evaluating existing technologies, designing, and implementing the proposed system. Key outputs include percentage-based scores for tree density, canopy cover, tree diversity (TD-50), and trees per capita, tailored for census-designated places in California. Additionally, the system's outputs can be augmented with detailed tree metrics to generate comprehensive evaluations of urban forestry. These insights are designed to inform actionable recommendations and metrics for Urban and Community Forestry (U&CF) programs.

- **Author**: Daniel Gonzalez
- **Degree/Field**: Master of Science, Computer Science
- **Planned Graduation**: January 2025

## Repository Structure
- `main.tex` — root document (no need to edit in most cases)
- `cpthesis.cls` — Cal Poly thesis class (do not modify unless fixing class-level issues)
- `frontmatter/` — thesis metadata and pre-chapter pages
  - `information.tex` — title, author, degree info, committee, keywords
  - `abstract.tex`, `acknowledgments.tex`, `dedication.tex`
  - `listings.tex` — toggles for lists of figures/tables/algorithms/code
  - `nomenclature.tex` — List of Symbols entries (optional)
  - `preamble.tex` — additional packages/config for this thesis
- `chapters/` — chapter sources and `outline.tex` to include them
- `appendices/` — appendices and `outline.tex` (optional)
- `figures/` — figures and graphics
- `bibliography/` — `references.bib` and `bib_info.tex` (biblatex + biber)
- `latexmkrc` — helper for nomenclature (auto-makeindex)

## Build (Local)
Prerequisites:
- TeX distribution (TeX Live 2023+ recommended) with `latexmk`, `biber`, and `makeindex`
  - On Debian/Ubuntu: `sudo apt install texlive-full latexmk biber`

Commands:
```bash
# Build PDF
latexmk -pdf main.tex

# Continuous preview (rebuild on change)
latexmk -pdf -pvc main.tex

# Clean aux files
latexmk -C
```
Results are written to `main.pdf`. The included `latexmkrc` automatically runs `makeindex` for nomenclature when needed.

## Editing Guide
- Update thesis info in `frontmatter/information.tex` (title, author, degree year, committee, keywords).
- Write the abstract in `frontmatter/abstract.tex`.
- Optional pages: `frontmatter/acknowledgments.tex`, `frontmatter/dedication.tex`.
- If using a List of Symbols, add entries to `frontmatter/nomenclature.tex` using `\nomenclature`.
- Add/organize chapters in `chapters/` and include them from `chapters/outline.tex`.
- Add appendices in `appendices/` and include from `appendices/outline.tex` (use `\appendix[<n>]` when applicable).
- Place figures in `figures/` and reference with standard LaTeX figure environments.
- Manage citations in `bibliography/references.bib` (biblatex with biber).
  - Switch between Bibliography vs References in `bibliography/bib_info.tex` (toggle `\nocite{*}` and `\bibname` as desired).
- Customize packages/settings specific to this thesis in `frontmatter/preamble.tex`.

Notes:
- You generally should not edit `main.tex` or `cpthesis.cls`.
- If a list (e.g., List of Algorithms) would be empty, remove its corresponding command in `frontmatter/listings.tex`.

## Overleaf
You can also compile this project on Overleaf. Upload the entire repository and set `main.tex` as the root document.

## License
This project is licensed under the MIT License. See `LICENSE` for details.
