# Florentia Illustrata — SPARQL Query Collection
## Female Property Ownership in the Florentine Cadastre of the 19th century

This repository collects SPARQL queries designed to explore the **Florentia Illustrata** dataset. The dataset is modelled using the [CIDOC-CRM](http://www.cidoc-crm.org/) ontology and published as RDF/Turtle.

The queries in this collection focus specifically on **female property ownership**: identifying women in the records, measuring their holdings, characterising the types of property they owned, and detecting co-ownership patterns. Women appear in the cadastre mostly with a distinctive identifier (`moglie_di`, "wife of"), which this collection uses as the primary gender filter.

---

## Dataset & SPARQL Endpoint

The data is available at the DOI:

```
https://doi.org/10.6092/unibo%2Famsacta%2F8236
```

The base namespace used throughout is:

```
https://florentiaillustrata.net/resource/
```

The ontology prefix used is CIDOC-CRM:

```
PREFIX crm: <http://www.cidoc-crm.org/cidoc-crm/>
```

---

## How Women Are Identified

Women are identified by the presence of a `moglie_di` identifier node among the `crm:P1_is_identified_by` values of a `crm:E21_Person`. Men are identified conversely by the presence of `patronimico` and the absence of `moglie_di`. Several queries use `STRSTARTS` filters on URI prefixes to enforce this distinction:

```sparql
# Women
FILTER(STRSTARTS(STR(?id), "https://florentiaillustrata.net/resource/moglie_di/"))

# Men (proxy: have patronimico, no moglie_di)
FILTER(STRSTARTS(STR(?n), "https://florentiaillustrata.net/resource/nome/"))
FILTER(STRSTARTS(STR(?c), "https://florentiaillustrata.net/resource/cognome/"))
FILTER(STRSTARTS(STR(?p), "https://florentiaillustrata.net/resource/patronimico/"))
```

> ⚠️ **Note on the male proxy (Q05):** women who also have a `patronimico` node will match the male filter unless a `FILTER NOT EXISTS { ... moglie_di ... }` clause is added explicitly.

---

## Query Index

| # | Title | Returns |
|---|-------|---------|
| [Q01](#q01--female-property-owners--basic-aggregation) | Female property owners — basic aggregation | Owner, marital ref, parcel count, parcel list |
| [Q02](#q02--identity-profile-per-woman-with-total-surface) | Identity profile per woman with total surface | Full identity + parcel count + total m² |
| [Q03](#q03--property-detail-for-a-specific-woman) | Property detail for a specific woman | Parcel, via, civico, specie_pro |
| [Q04](#q04--property-type-distribution-for-women) | Property type distribution for women | specie_pro, count, % of total |
| [Q05](#q05--property-type-distribution-for-men) | Property type distribution for men | specie_pro, count, % of total |
| [Q06](#q06--full-per-parcel-detail-with-co-owners) | Full per-parcel detail with co-owners | Row per parcel + ownership type + co-owner names |

---

## Queries

---

### Q01 — Female property owners — basic aggregation

Retrieves all female property owners (identified via `moglie_di`) with a count of their parcels and the list of parcel URIs. Ownership is reconstructed through `crm:E8_Acquisition` events: a parcel links to an acquisition event via `crm:P24i_changed_ownership_through`, and a person acquires title through it via `crm:P22i_acquired_title_through` (legal ownership).

**Returns:** owner label, marital reference, patronymic(s), number of owned parcels, list of parcel URIs — ordered by parcel count descending.

```sparql
PREFIX crm: <http://www.cidoc-crm.org/cidoc-crm/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT
    ?owner_name
    ?moglie_di
    (COUNT(DISTINCT ?subject) AS ?num_proprieta)
    (GROUP_CONCAT(DISTINCT ?subject; SEPARATOR=", ") AS ?subjects)
    (GROUP_CONCAT(DISTINCT ?patronimico; SEPARATOR=", ") AS ?patronimici)
WHERE {
    ?subject crm:P2_has_type crm:E24_Physical_Human_Made_Thing .
    ?subject crm:P24i_changed_ownership_through ?event .
    ?owner crm:P22i_acquired_title_through ?event .
    ?owner rdfs:label ?owner_name .
    ?owner crm:P1_is_identified_by ?patronimicoRes .
    ?patronimicoRes crm:P2_has_type <https://florentiaillustrata.net/resource/patronimico> .
    ?patronimicoRes rdfs:label ?patronimico .
    ?owner crm:P1_is_identified_by ?moglieDiRes .
    ?moglieDiRes crm:P2_has_type <https://florentiaillustrata.net/resource/moglie_di> .
    ?moglieDiRes rdfs:label ?moglie_di .
}
GROUP BY ?owner_name ?moglie_di
ORDER BY DESC(?num_proprieta)
```

---

### Q02 — Identity profile per woman with total surface

Builds a structured identity for each woman by extracting nome, cognome, patronimico, and husband's name from separate identifier nodes. Returns aggregated ownership statistics per person. Surface values are stored as `rdfs:label` on dimension nodes typed with `crm:P91_has_unit`; the query casts them to `xsd:decimal` before aggregating.

> ⚠️ Women lacking a `patronimico` node are silently excluded. See Q06 for a version that handles this with `OPTIONAL`.

**Returns:** nome, cognome, patronimico, marito, number of parcels, total surface in m² — ordered by total surface descending.

```sparql
PREFIX crm:  <http://www.cidoc-crm.org/cidoc-crm/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd:  <http://www.w3.org/2001/XMLSchema#>

SELECT
  ?nome
  ?cognome
  ?patronimico
  ?marito
  (COUNT(DISTINCT ?appezzamento) AS ?numeroAppezzamenti)
  (SUM(?mq) AS ?superficieTotaleMq)
WHERE {
  ?donna
      crm:P2_has_type crm:E21_Person ;
      crm:P1_is_identified_by ?id_nome ;
      crm:P1_is_identified_by ?id_cognome ;
      crm:P1_is_identified_by ?id_patronimico ;
      crm:P1_is_identified_by ?id_moglie ;
      crm:P22i_acquired_title_through ?acq .
  FILTER(STRSTARTS(STR(?id_nome),
    "https://florentiaillustrata.net/resource/nome/"))
  FILTER(STRSTARTS(STR(?id_cognome),
    "https://florentiaillustrata.net/resource/cognome/"))
  FILTER(STRSTARTS(STR(?id_patronimico),
    "https://florentiaillustrata.net/resource/patronimico/"))
  FILTER(STRSTARTS(STR(?id_moglie),
    "https://florentiaillustrata.net/resource/moglie_di/"))
  BIND(STRAFTER(STR(?id_nome),        "/nome/")        AS ?nome)
  BIND(STRAFTER(STR(?id_cognome),     "/cognome/")     AS ?cognome)
  BIND(STRAFTER(STR(?id_patronimico), "/patronimico/") AS ?patronimico)
  BIND(STRAFTER(STR(?id_moglie),      "/moglie_di/")   AS ?marito)
  ?appezzamento
      crm:P24i_changed_ownership_through ?acq ;
      crm:P43_has_dimension ?dimension .
  ?dimension
      crm:P91_has_unit <https://florentiaillustrata.net/resource/metri_quadrati> ;
      rdfs:label ?sup .
  BIND(xsd:decimal(?sup) AS ?mq)
}
GROUP BY ?nome ?cognome ?patronimico ?marito
ORDER BY DESC(?superficieTotaleMq)
```

---

### Q03 — Property detail for a specific woman

Retrieves all parcels owned by a single individual, with address and property type. The example uses **Luisa Strozzi** (wife of Bernardino Panciatichi Ximenes d'Aragona); adapt the three `crm:P1_is_identified_by` values to query any other person.

**Returns:** parcel URI, surface (m²), street name, street number, property type.

```sparql
PREFIX crm:  <http://www.cidoc-crm.org/cidoc-crm/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT
  ?appezzamento
  ?mq
  ?via
  ?numCivico
  ?speciePro
WHERE {
  ?donna
      crm:P1_is_identified_by <https://florentiaillustrata.net/resource/nome/Luisa> ;
      crm:P1_is_identified_by <https://florentiaillustrata.net/resource/cognome/Strozzi> ;
      crm:P1_is_identified_by <https://florentiaillustrata.net/resource/moglie_di/Bernardino_Panciatichi_Ximenes_d'Aragona> ;
      crm:P22i_acquired_title_through ?acq .
  ?appezzamento
      crm:P24i_changed_ownership_through ?acq ;
      crm:P43_has_dimension ?dimension ;
      crm:P53_has_former_or_current_location ?place ;
      crm:P2_has_type ?specieProURI .
  ?dimension
      crm:P91_has_unit <https://florentiaillustrata.net/resource/metri_quadrati> ;
      rdfs:label ?mq .
  ?place crm:P1_is_identified_by ?idVia , ?idNum .
  FILTER(STRSTARTS(STR(?idVia),
    "https://florentiaillustrata.net/resource/toponomastica/"))
  FILTER(STRSTARTS(STR(?idNum),
    "https://florentiaillustrata.net/resource/num_civico/"))
  BIND(STRAFTER(STR(?idVia), "/toponomastica/") AS ?via)
  BIND(STRAFTER(STR(?idNum), "/num_civico/")    AS ?numCivico)
  FILTER(STRSTARTS(STR(?specieProURI),
    "https://florentiaillustrata.net/resource/specie_pro/"))
  BIND(STRAFTER(STR(?specieProURI), "/specie_pro/") AS ?speciePro)
}
```

---

### Q04 — Property type distribution for women, with percentage

Counts how many parcels owned by women fall into each `specie_pro` category, expressed both as an absolute count and as a percentage of all parcels owned by women. A scalar subquery computes the denominator.

**Returns:** specie_pro, parcel count, number of distinct women, percentage — top 20 by count.

```sparql
PREFIX crm:  <http://www.cidoc-crm.org/cidoc-crm/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT
  ?speciePro
  (COUNT(DISTINCT ?appezzamento) AS ?conteggioAppezzamenti)
  (COUNT(DISTINCT ?donna)        AS ?numeroDonne)
  ( (COUNT(DISTINCT ?appezzamento) / ?totaleAppezzamenti) * 100 AS ?percentuale)
WHERE {
  {
    SELECT (COUNT(DISTINCT ?appTot) AS ?totaleAppezzamenti)
    WHERE {
      ?donnaTot crm:P2_has_type crm:E21_Person ;
                crm:P1_is_identified_by ?n ;
                crm:P1_is_identified_by ?c ;
                crm:P1_is_identified_by ?p ;
                crm:P1_is_identified_by ?m ;
                crm:P22i_acquired_title_through ?acqTot .
      FILTER(STRSTARTS(STR(?n), "https://florentiaillustrata.net/resource/nome/"))
      FILTER(STRSTARTS(STR(?c), "https://florentiaillustrata.net/resource/cognome/"))
      FILTER(STRSTARTS(STR(?p), "https://florentiaillustrata.net/resource/patronimico/"))
      FILTER(STRSTARTS(STR(?m), "https://florentiaillustrata.net/resource/moglie_di/"))
      ?appTot crm:P24i_changed_ownership_through ?acqTot .
    }
  }
  ?donna crm:P2_has_type crm:E21_Person ;
         crm:P1_is_identified_by ?n ;
         crm:P1_is_identified_by ?c ;
         crm:P1_is_identified_by ?p ;
         crm:P1_is_identified_by ?m ;
         crm:P22i_acquired_title_through ?acq .
  FILTER(STRSTARTS(STR(?n), "https://florentiaillustrata.net/resource/nome/"))
  FILTER(STRSTARTS(STR(?c), "https://florentiaillustrata.net/resource/cognome/"))
  FILTER(STRSTARTS(STR(?p), "https://florentiaillustrata.net/resource/patronimico/"))
  FILTER(STRSTARTS(STR(?m), "https://florentiaillustrata.net/resource/moglie_di/"))
  ?appezzamento crm:P24i_changed_ownership_through ?acq ;
                crm:P2_has_type ?specieProURI .
  FILTER(STRSTARTS(STR(?specieProURI),
    "https://florentiaillustrata.net/resource/specie_pro/"))
  BIND(STRAFTER(STR(?specieProURI), "/specie_pro/") AS ?speciePro)
}
GROUP BY ?speciePro ?totaleAppezzamenti
ORDER BY DESC(?conteggioAppezzamenti)
LIMIT 20
```

---

### Q05 — Property type distribution for men, with percentage

Identical in structure to Q04, but filters for male owners. Intended as a comparison baseline for Q04.

**Returns:** specie_pro, parcel count, number of distinct men, percentage — top 20 by count.

```sparql
PREFIX crm:  <http://www.cidoc-crm.org/cidoc-crm/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT
  ?speciePro
  (COUNT(DISTINCT ?appezzamento)                AS ?conteggioAppezzamenti)
  (COUNT(DISTINCT ?uomo)                        AS ?numeroUomini)
  ( (COUNT(DISTINCT ?appezzamento) / ?totaleAppezzamentiUomini) * 100 AS ?percentuale)
WHERE {
  {
    SELECT (COUNT(DISTINCT ?appTot) AS ?totaleAppezzamentiUomini)
    WHERE {
      ?uomoTot crm:P2_has_type crm:E21_Person ;
               crm:P1_is_identified_by ?n ;
               crm:P1_is_identified_by ?c ;
               crm:P1_is_identified_by ?p ;
               crm:P22i_acquired_title_through ?acqTot .
      FILTER(STRSTARTS(STR(?n), "https://florentiaillustrata.net/resource/nome/"))
      FILTER(STRSTARTS(STR(?c), "https://florentiaillustrata.net/resource/cognome/"))
      FILTER(STRSTARTS(STR(?p), "https://florentiaillustrata.net/resource/patronimico/"))
      ?appTot crm:P24i_changed_ownership_through ?acqTot .
    }
  }
  ?uomo crm:P2_has_type crm:E21_Person ;
        crm:P1_is_identified_by ?n ;
        crm:P1_is_identified_by ?c ;
        crm:P1_is_identified_by ?p ;
        crm:P22i_acquired_title_through ?acq .
  FILTER(STRSTARTS(STR(?n), "https://florentiaillustrata.net/resource/nome/"))
  FILTER(STRSTARTS(STR(?c), "https://florentiaillustrata.net/resource/cognome/"))
  FILTER(STRSTARTS(STR(?p), "https://florentiaillustrata.net/resource/patronimico/"))
  ?appezzamento crm:P24i_changed_ownership_through ?acq ;
                crm:P2_has_type ?specieProURI .
  FILTER(STRSTARTS(STR(?specieProURI),
    "https://florentiaillustrata.net/resource/specie_pro/"))
  BIND(STRAFTER(STR(?specieProURI), "/specie_pro/") AS ?speciePro)
}
GROUP BY ?speciePro ?totaleAppezzamentiUomini
ORDER BY DESC(?conteggioAppezzamenti)
LIMIT 20
```

---

### Q06 — Full per-parcel detail with co-owners

The most complete query in the collection. Returns one row per parcel for each female owner, combining full identity data, address, property type, ownership type (`esclusivo` / `comproprietà`), and a semicolon-separated list of co-owner names where applicable.

Three subqueries handle: (1) per-woman aggregates, (2) owner count per acquisition event for efficient co-ownership detection, (3) an `OPTIONAL` `GROUP_CONCAT` for co-owner labels. The patronymic is wrapped in `OPTIONAL` to avoid excluding women who lack it.

**Returns:** nome, cognome, patronimico, marito, parcel URI, m², via, civico, specie_pro, tipoPossesso, comproprietari, total parcel count, total surface — one row per parcel, ordered by surname and name.

```sparql
PREFIX crm:  <http://www.cidoc-crm.org/cidoc-crm/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd:  <http://www.w3.org/2001/XMLSchema#>

SELECT
  ?nome
  ?cognome
  ?patronimico
  ?marito
  ?appezzamento
  ?mq
  ?via
  ?numCivico
  ?speciePro
  ?tipoPossesso
  ?comproprietari
  ?numeroAppezzamenti
  ?superficieTotaleMq
WHERE {

  # ==================================================
  # SUBQUERY 1: aggregazione per donna
  # ==================================================
  {
    SELECT
      ?donna ?nome ?cognome ?patronimico ?marito
      (COUNT(DISTINCT ?app) AS ?numeroAppezzamenti)
      (SUM(?mq2) AS ?superficieTotaleMq)
    WHERE {
      ?donna crm:P2_has_type crm:E21_Person ;
             crm:P1_is_identified_by ?n ;
             crm:P1_is_identified_by ?c ;
             crm:P1_is_identified_by ?m ;
             crm:P22i_acquired_title_through ?acq2 .
      FILTER(STRSTARTS(STR(?n),
        "https://florentiaillustrata.net/resource/nome/"))
      FILTER(STRSTARTS(STR(?c),
        "https://florentiaillustrata.net/resource/cognome/"))
      FILTER(STRSTARTS(STR(?m),
        "https://florentiaillustrata.net/resource/moglie_di/"))
      OPTIONAL {
        ?donna crm:P1_is_identified_by ?p .
        FILTER(STRSTARTS(STR(?p),
          "https://florentiaillustrata.net/resource/patronimico/"))
      }
      BIND(STRAFTER(STR(?n), "/nome/")    AS ?nome)
      BIND(STRAFTER(STR(?c), "/cognome/") AS ?cognome)
      BIND(
        IF(BOUND(?p),
           STRAFTER(STR(?p), "/patronimico/"),
           ""
        ) AS ?patronimico
      )
      BIND(STRAFTER(STR(?m), "/moglie_di/") AS ?marito)
      ?app crm:P24i_changed_ownership_through ?acq2 ;
           crm:P43_has_dimension ?d2 .
      ?d2 crm:P91_has_unit
            <https://florentiaillustrata.net/resource/metri_quadrati> ;
          rdfs:label ?s2 .
      BIND(xsd:decimal(?s2) AS ?mq2)
    }
    GROUP BY ?donna ?nome ?cognome ?patronimico ?marito
  }

  # ==================================================
  # SUBQUERY 2: numero proprietari per acquisition
  # ==================================================
  {
    SELECT ?acq (COUNT(DISTINCT ?persona) AS ?numProprietari)
    WHERE {
      ?persona crm:P22i_acquired_title_through ?acq ;
               crm:P2_has_type crm:E21_Person .
    }
    GROUP BY ?acq
  }

  # ==================================================
  # SUBQUERY 3: nomi dei comproprietari (OPTIONAL)
  # ==================================================
  OPTIONAL {
    SELECT ?acq ?donna
      (GROUP_CONCAT(DISTINCT ?labelAltro; separator="; ") AS ?comproprietari)
    WHERE {
      ?donna crm:P22i_acquired_title_through ?acq .
      ?altraPersona crm:P22i_acquired_title_through ?acq ;
                    crm:P2_has_type crm:E21_Person ;
                    rdfs:label ?labelAltro .
      FILTER(?altraPersona != ?donna)
    }
    GROUP BY ?acq ?donna
  }

  # ==================================================
  # RIGHE DETTAGLIATE
  # ==================================================
  ?donna crm:P22i_acquired_title_through ?acq .
  ?appezzamento
      crm:P24i_changed_ownership_through ?acq ;
      crm:P43_has_dimension ?dimension ;
      crm:P53_has_former_or_current_location ?place ;
      crm:P2_has_type ?specieProURI .
  ?dimension
      crm:P91_has_unit
        <https://florentiaillustrata.net/resource/metri_quadrati> ;
      rdfs:label ?sup .
  BIND(xsd:decimal(?sup) AS ?mq)
  ?place crm:P1_is_identified_by ?idVia , ?idNum .
  FILTER(STRSTARTS(STR(?idVia),
    "https://florentiaillustrata.net/resource/toponomastica/"))
  FILTER(STRSTARTS(STR(?idNum),
    "https://florentiaillustrata.net/resource/num_civico/"))
  BIND(STRAFTER(STR(?idVia), "/toponomastica/") AS ?via)
  BIND(STRAFTER(STR(?idNum), "/num_civico/")    AS ?numCivico)
  FILTER(STRSTARTS(STR(?specieProURI),
    "https://florentiaillustrata.net/resource/specie_pro/"))
  BIND(STRAFTER(STR(?specieProURI), "/specie_pro/") AS ?speciePro)
  BIND(
    IF(?numProprietari > 1,
       "comproprietà",
       "esclusivo"
    ) AS ?tipoPossesso
  )
}
ORDER BY ?cognome ?nome ?appezzamento
```

---

## Notes on the Data Model

- **Ownership** is always mediated through acquisition events (`crm:E8_Acquisition`). A parcel links to an event via `crm:P24i_changed_ownership_through`; a person acquires title via `crm:P22i_acquired_title_through`. Co-ownership means two or more persons sharing the same event node.
- **Surface** (`crm:P43_has_dimension`) is stored as a string label on a dimension node; cast with `xsd:decimal()` before arithmetic.
- **Address** is attached to a place node (`crm:P53_has_former_or_current_location`) and decomposed into `toponomastica` (street name) and `num_civico` (street number) identifier nodes.
- **Property type** (`specie_pro`) is encoded as a `crm:P2_has_type` URI on the parcel, not as a literal.
- **Patronimico** may be absent for some women; Q02 silently excludes them, while Q06 handles this case with `OPTIONAL`.

---

*Queries developed as part of the Florentiae Illustrata project. CIDOC-CRM ontology: http://www.cidoc-crm.org/cidoc-crm/*
