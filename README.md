# Supplementary Materials — *Beyond Spatial Data*
## Exploring Gender and Property in the Catasto Generale Toscano through the Florentia Illustrata Semantic Platform

This repository contains the supplementary materials for the paper:

> *Beyond Spatial Data: Exploring Gender and Property in the Catasto Generale Toscano through the Florentia Illustrata Semantic Platform*, submitted to AIUCD 2026.

The paper presents a usability and flexibility test of the [Florentia Illustrata](https://florentiaillustrata.net) research infrastructure, evaluating whether a semantic architecture grounded in CIDOC-CRM can support socio-economic and gender-oriented inquiry on a historical cadastral dataset. The case study focuses on female landownership in nineteenth-century Florence, drawing on the *Catasto Generale Toscano* (c. 1735–1835) as encoded in the Florentia Illustrata Knowledge Graph.

The materials collected here document the two analytical procedures described in the paper: the SPARQL queries used to extract ownership patterns from the Knowledge Graph, and the Python pipeline used to identify female landowners not captured by the primary `moglie_di` filter.

---

## Repository Structure

```
.
├── README.md                          ← this file
├── sparql_queries.md                  ← documented SPARQL queries
├── gender_filter_with_names.md        ← documented Python gender inference pipeline
├── results_sparql_queries/            ← CSV outputs of the SPARQL queries
└── results_gender_filter_with_names/  ← CSV outputs of the Python pipeline
```

---

## Data Sources

### Florentia Illustrata Knowledge Graph

The RDF Knowledge Graph underlying Florentia Illustrata is serialized in Turtle format and publicly available at:

> [https://doi.org/10.6092/unibo/amsacta/8236](https://doi.org/10.6092/unibo/amsacta/8236)

The graph models 17,681 land parcels (*appezzamenti*) from the Florentine cadastre of 1830. Parcels are instances of `crm:E24_Physical_Human_Made_Thing`; ownership relations are represented through `crm:E8_Acquisition` events; persons are modeled as `crm:E21_Person` with associated identifier nodes for given name, surname, patronymic, and relational descriptors such as `moglie_di` ("wife of"). The SPARQL endpoint is available at `https://florentiaillustrata.net/sparql`.

The original cadastral dataset underlying the Knowledge Graph is published in:

> Belli, G., Lucchesi, F., & Raggi, P. (2022). *Firenze nella prima metà dell'Ottocento: La città nei documenti del Catasto Generale Toscano*. Firenze University Press. [https://doi.org/10.36253/979-12-215-0002-8](https://doi.org/10.36253/979-12-215-0002-8)

### Italian Name–Gender Dataset

The gender inference pipeline (see `gender_filter_with_names.md`) relies on an external reference dataset of Italian given names with gender attributions and frequency statistics, derived from civil registry data:

> [https://gist.github.com/tezzutezzu/8f025345cadc5f92b9b311bf032b264d](https://gist.github.com/tezzutezzu/8f025345cadc5f92b9b311bf032b264d)

---

## Contents

### `sparql_queries.md`

Documents the six SPARQL queries used to interrogate the Florentia Illustrata Knowledge Graph on the topic of female landownership. The queries cover: basic aggregation of female owners, identity profiling with total surface area, per-parcel detail for a specific individual, property type distribution by gender, and full per-parcel detail including co-ownership type and co-owner names. Each query is accompanied by a description of its logic, the variables returned, and notes on known limitations.

The query results that inform the paper's findings (Section 3) are collected in `results_sparql_queries/`.

### `gender_filter_with_names.md`

Documents the two-step Python pipeline described in Section 2.1 of the paper, developed to identify female landowners not captured by the `moglie_di` filter — specifically unmarried women, widows, and nuns. The pipeline cross-references first names extracted from the RDF graph against the external name–gender dataset. Step 1 extracts candidates whose first name has any feminine attestation; Step 2 removes names also attested as masculine to reduce false positives. Through this process, 79 individuals were identified as likely unmarried female landowners, of whom three carried the title *suora* (nun).

The output CSVs are collected in `results_gender_filter_with_names/`.

### `results_sparql_queries/`

CSV files produced by running the queries documented in `sparql_queries.md` against the Florentia Illustrata SPARQL endpoint. File names correspond to query identifiers (Q01–Q06).

### `results_gender_filter_with_names/`

CSV files produced by the Python pipeline documented in `gender_filter_with_names.md`:

| File | Description |
|------|-------------|
| `donne_non_sposate_senza_moglie_di.csv` | Candidates without `moglie_di` whose first name has a feminine attestation (Step 1 output) |
| `donne_non_sposate_filtrato.csv` | Final list after removal of ambiguous names (Step 2 output) |

---

## How to Reproduce

### SPARQL queries

Queries can be run directly against the public endpoint at `https://florentiaillustrata.net/sparql` by copying the query text from `sparql_queries.md`. No local installation is required.

### Python pipeline

Requirements: Python 3, `pandas`, `rdflib`.

```bash
pip install pandas rdflib
```

Download the Turtle file from the DOI above and the `data.csv` from the GitHub Gist linked in the Data Sources section. Update the file paths in the scripts (marked as `path/to/...`) to match your local environment, then run the two steps in order as documented in `gender_filter_with_names.md`.

---

## Citation

If you use these materials, please cite the associated paper:

> [pending]

---

## License

The query collection and pipeline code in this repository are released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The underlying data (Turtle file and `data.csv`) are subject to the terms of their respective sources linked above.
