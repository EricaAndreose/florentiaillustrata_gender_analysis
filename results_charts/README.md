# `results_charts` — Visualisation Notebook and Data

This folder contains the Jupyter notebook used to produce the charts included in the paper, along with its input data files and the resulting chart images.

---

## Contents

```
results_charts/
├── Florentia_visualizzazioni.ipynb        ← notebook that generates all charts
├── florentia_visualizations.png           ← main composite figure (paper)
├── florentia_visualizations_all.png       ← extended version, all property categories
├── florentia_visualizations_attivita.png  ← chart focused on activity-type properties
├── florentia_visualizations_log_scale.png ← log-scale variant for surface-area comparison
└── data/                                  ← CSV input files consumed by the notebook
    ├── macro.csv
    ├── no_mogliedi.csv
    ├── si_mogliedi.csv
    └── lista_donne_non_sposate.csv
```

---

## Input Files (`data/`)

### `macro.csv`

Lookup table that maps each `specie_pro` value (the raw property-type label from the cadastral record) to a `macro_categoria` (an aggregated category used for grouping in the charts).

| Column | Description |
|--------|-------------|
| `specie_pro` | Raw property type as recorded in the *Catasto Generale Toscano* |
| `macro_categoria` | Aggregated category (e.g. `fabbricato urbano`, `terreno`) |

---

### `si_mogliedi.csv`

Per-parcel ownership records for women identified through the `moglie_di` filter — i.e. women explicitly described as "wife of" a named individual. These correspond to the primary cohort of married female landowners extracted by the SPARQL queries (see `../sparql_queries.md`, queries Q02–Q06).

| Column | Description |
|--------|-------------|
| `nome` | Given name |
| `cognome` | Surname |
| `patronimico` | Patronymic |
| `marito` | Name of the husband (from the `moglie_di` relational descriptor) |
| `appezzamento` | URI of the land parcel in the Florentia Illustrata Knowledge Graph |
| `mq` | Surface area of the parcel in square metres |
| `via` | Street name |
| `numCivico` | Civic number |
| `speciePro` | Property type label |
| `tipoPossesso` | Ownership type (`esclusivo` / `comproprietà`) |
| `comproprietari` | Semicolon-separated list of co-owners, if any |
| `numeroAppezzamenti` | Total number of parcels held by the person |
| `superficieTotaleMq` | Total surface area held by the person across all parcels (m²) |

---

### `no_mogliedi.csv`

Per-parcel ownership records for women without a `moglie_di` descriptor — i.e. likely unmarried women, widows, or nuns. This cohort was identified through the gender-inference pipeline documented in `../gender_filter_with_names.md`. The schema is identical to `si_mogliedi.csv` but does not include a `marito` column.

| Column | Description |
|--------|-------------|
| `nome` | Given name |
| `cognome` | Surname |
| `patronimico` | Patronymic |
| `appezzamento` | URI of the land parcel |
| `mq` | Surface area of the parcel (m²) |
| `via` | Street name |
| `numCivico` | Civic number |
| `speciePro` | Property type label |
| `tipoPossesso` | Ownership type (`esclusivo` / `comproprietà`) |
| `comproprietari` | Semicolon-separated list of co-owners, if any |
| `numeroAppezzamenti` | Total number of parcels held by the person |
| `superficieTotaleMq` | Total surface area held by the person (m²) |

---

### `lista_donne_non_sposate.csv`

The final output of the gender-inference pipeline (Step 2): the list of persons inferred as unmarried female landowners after ambiguous names have been removed. This file is used by the notebook to label or filter individuals within the `no_mogliedi` cohort.

| Column | Description |
|--------|-------------|
| `uri` | Person URI in the Florentia Illustrata Knowledge Graph |
| `nome` | Given name |
| `cognome` | Surname |
| `patronimico` | Patronymic |
| `genere_inferito` | Inferred gender (`F`) |

---

## How to Run the Notebook

Requirements: Python 3, `pandas`, `matplotlib`.

```bash
pip install pandas matplotlib
```

Open `Florentia_visualizzazioni.ipynb` in Jupyter and run all cells. The notebook reads the four CSV files from `data/` using relative paths and writes the chart images to the current directory. No external data download is required.
