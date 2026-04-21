# Gender Inference Pipeline
## Identifying unmarried women in the Florentia Illustrata dataset

In the Florentia Illustrata knowledge graph, married women are reliably identified by the presence of a `moglie_di` identifier node. However, this marker is absent for women who were unmarried at the time of the cadastral survey — widows, nuns, and single women — who therefore risk being excluded from gender-based analyses if `moglie_di` is used as the sole filter.

This pipeline recovers those women by cross-referencing the first names found in the RDF graph against an external Italian name–gender frequency dataset. It proceeds in two steps: first extracting all persons without `moglie_di` whose first name appears in the feminine name list (Step 1), then removing any name that is also attested as masculine to reduce false positives (Step 2).

---

## External Data Sources

The Florentia Illustrata RDF graph in Turtle format is available for download at: https://doi.org/10.6092/unibo/amsacta/8236

The pipeline relies also on an external CSV file containing Italian first names with associated gender and frequency statistics, compiled from Italian civil registry data.

**Source:** [`data.csv`](https://gist.github.com/tezzutezzu/8f025345cadc5f92b9b311bf032b264d)

The file has the following structure:

```
name,rank,year,count,gender,percent
ANDREA,1,1999,10336,m,3.914617704
FRANCESCO,2,1999,9494,m,3.595721796
...
```

Each row represents a name as recorded in a given year, with its frequency rank, absolute count, gender (`m` or `f`), and percentage share among all names registered that year. A single name may appear multiple times across different years or with different gender values.

Download the file and place it at the path referenced in the scripts before running (or update the `DATA_CSV` / `CSV_PATH` constants to match your local path).


---

## Input Files

| File | Description | Link |
|------|-------------|------|
| `florentiaIllustrata_KG_20250310_v01.ttl` | The Florentiae Illustrata RDF graph in Turtle format | download: https://doi.org/10.6092/unibo/amsacta/8236 |
| `data.csv` | External Italian name–gender frequency dataset | download: https://gist.github.com/tezzutezzu/8f025345cadc5f92b9b311bf032b264d |

---

## Output Files

| File | Produced by | Description |
|------|-------------|-------------|
| `donne_non_sposate_senza_moglie_di.csv` | Step 1 | All persons without `moglie_di` whose first name matches a feminine name in `data.csv` |
| `donne_non_sposate_filtrato.csv` | Step 2 | The above list with any name also attested as masculine removed |

---

## Step 1 — Extract feminine-name candidates from the RDF graph

This step reads the Turtle file directly using `rdflib`, iterates over all `crm:E21_Person` entities, and filters out anyone who already has a `moglie_di` identifier. Among the remaining persons, it retains only those whose first name (`/nome/` URI segment) appears in the set of names marked `gender = "f"` in `data.csv`.

**Dependencies:** `rdflib`, `pandas`

**Input:** `florentiaIllustrata_KG_20250310_v01.ttl`, `data.csv`  
**Output:** `donne_non_sposate_senza_moglie_di.csv`

```python
import pandas as pd
from rdflib import Graph, Namespace

# =========================
# 1. CARICAMENTO CSV NOMI
# =========================

CSV_PATH = "path/to/data.csv"

df = pd.read_csv(
    CSV_PATH,
    encoding="latin1"
)

# normalizzazione
df["name"] = df["name"].str.strip().str.lower()

# teniamo SOLO nomi chiaramente femminili
df_f = df[
    (df["gender"] == "f")
]

# set dei nomi femminili affidabili
nomi_femminili = set(df_f["name"])

print(f"Nomi femminili affidabili: {len(nomi_femminili)}")

# =========================
# 2. CARICAMENTO RDF
# =========================

RDF_PATH = "path/to/florentiaIllustrata_KG_20250310_v01.ttl"

g = Graph()
g.parse(RDF_PATH, format="turtle")

CRM = Namespace("http://www.cidoc-crm.org/cidoc-crm/")

# =========================
# 3. TUTTE LE PERSONE
# =========================

persone = set(
    g.subjects(CRM.P2_has_type, CRM.E21_Person)
)

print(f"Persone totali: {len(persone)}")

# =========================
# 4. PERSONE CON 'moglie_di'
# =========================

persone_con_moglie_di = set()

for person in persone:
    for ident in g.objects(person, CRM.P1_is_identified_by):
        if "/moglie_di/" in str(ident):
            persone_con_moglie_di.add(person)
            break

# =========================
# 5. CANDIDATI SENZA 'moglie_di'
# =========================

candidati = persone - persone_con_moglie_di

print(f"Candidati senza moglie_di: {len(candidati)}")

# =========================
# 6. FUNZIONI DI ESTRAZIONE
# =========================

def estrai_valore(person, chiave):
    for ident in g.objects(person, CRM.P1_is_identified_by):
        uri = str(ident)
        if f"/{chiave}/" in uri:
            return uri.split(f"/{chiave}/")[-1].replace("_", " ")
    return None

# =========================
# 7. FILTRO DONNE
# =========================

donne_non_sposate = []

for person in candidati:
    nome = estrai_valore(person, "nome")
    if not nome:
        continue

    nome_norm = nome.lower()

    if nome_norm in nomi_femminili:
        donne_non_sposate.append({
            "uri": str(person),
            "nome": nome,
            "cognome": estrai_valore(person, "cognome"),
            "patronimico": estrai_valore(person, "patronimico"),
            "genere_inferito": "F"
        })

# =========================
# 8. ESPORTAZIONE CSV
# =========================

output_df = pd.DataFrame(donne_non_sposate)

OUTPUT_PATH = "path/to/donne_non_sposate_senza_moglie_di.csv"
output_df.to_csv(OUTPUT_PATH, index=False)

print(f"Donne non sposate trovate: {len(output_df)}")
print(f"CSV scritto in: {OUTPUT_PATH}")
```

### Output columns

| Column | Description |
|--------|-------------|
| `uri` | Full URI of the person node in the RDF graph |
| `nome` | First name extracted from the `/nome/` URI segment |
| `cognome` | Surname extracted from the `/cognome/` URI segment |
| `patronimico` | Patronymic, if present |
| `genere_inferito` | Always `"F"` at this stage |

---

## Step 2 — Remove ambiguous (masculine-attested) names

The name–gender dataset contains names that appear under both `"f"` and `"m"` genders across different years (e.g. Andrea, which is predominantly masculine in Italian despite also having feminine attestations). Step 2 loads the CSV produced in Step 1 and removes any candidate whose first name appears at least once with `gender = "m"` in `data.csv`.

**Dependencies:** `pandas`

**Input:** `donne_non_sposate_senza_moglie_di.csv`, `data.csv`  
**Output:** `donne_non_sposate_filtrato.csv`

```python
import pandas as pd

# =========================
# 1. CARICAMENTO data.csv
# =========================

DATA_CSV = "path/to/data.csv"

df_nomi = pd.read_csv(DATA_CSV, encoding="latin1")
df_nomi["name"] = df_nomi["name"].str.strip().str.lower()

# =========================
# 2. IDENTIFICA NOMI CHE APPAIONO COME MASCHILI
# =========================

# Tutti i nomi che hanno ALMENO UNA riga con gender = "m"
nomi_maschili_da_escludere = set(
    df_nomi[df_nomi["gender"] == "m"]["name"]
)

print(f"Nomi maschili da escludere: {len(nomi_maschili_da_escludere)}")

# =========================
# 3. CARICAMENTO donne_non_sposate_senza_moglie_di.csv
# =========================

INPUT_CSV = "path/to/donne_non_sposate_senza_moglie_di.csv"

df_donne = pd.read_csv(INPUT_CSV)

# Normalizza i nomi per confronto
df_donne["nome_lower"] = df_donne["nome"].str.strip().str.lower()

# =========================
# 4. FILTRO: ESCLUDI MASCHILI
# =========================

df_filtrato = df_donne[
    ~df_donne["nome_lower"].isin(nomi_maschili_da_escludere)
]

# Rimuovi la colonna temporanea
df_filtrato = df_filtrato.drop(columns=["nome_lower"])

# =========================
# 5. ESPORTAZIONE
# =========================

OUTPUT_CSV = "path/to/donne_non_sposate_filtrato.csv"

df_filtrato.to_csv(OUTPUT_CSV, index=False)

print(f"\nRighe originali: {len(df_donne)}")
print(f"Righe dopo filtro: {len(df_filtrato)}")
print(f"Righe rimosse: {len(df_donne) - len(df_filtrato)}")
print(f"\nCSV filtrato salvato in: {OUTPUT_CSV}")
```

---

## Full Pipeline Diagram

```
florentiaIllustrata_KG_20250310_v01.ttl
        │
        │   (rdflib)
        ▼
  All crm:E21_Person nodes
        │
        │   exclude persons with /moglie_di/ identifier
        ▼
  Candidates without moglie_di
        │                          data.csv  ──► set of feminine names (gender = "f")
        │   keep if nome ∈ feminine names
        ▼
  donne_non_sposate_senza_moglie_di.csv      [Step 1 output]
        │                          data.csv  ──► set of masculine names (gender = "m")
        │   remove if nome ∈ masculine names
        ▼
  donne_non_sposate_filtrato.csv             [Step 2 output / final]
```

---

## Limitations and Caveats

**Name coverage.** The `data.csv` dataset is based on Italian civil registry records from 1999 onward. Names common in early nineteenth-century Florence but rare or absent in modern records may not appear in the list, causing some women to be missed. Conversely, archaic names shared across genders may not be flagged correctly.

**Ambiguous names.** Step 2 uses a conservative exclusion rule: any name with *at least one* masculine attestation in the dataset is removed. This reduces false positives (men incorrectly labelled as women) but may also exclude genuinely feminine names that occasionally appear in masculine records. The threshold can be tightened by filtering on the `percent` column instead (e.g. exclude only names where the masculine percentage exceeds a given value).

**No validation against the graph.** The pipeline infers gender from name alone and does not cross-check against other properties in the RDF graph (e.g. the absence of `patronimico`, or the presence of ecclesiastical titles for nuns). Manual review of the final CSV is recommended before integrating results into the dataset.

---

*Part of the Florentia Illustrata project.*
