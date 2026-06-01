# Corpus Preprocessing Pipeline for VOSviewer

A reproducible text preprocessing pipeline for large-scale **bibliometric co-occurrence mapping** using VOSviewer. Designed for Scopus corpora in the Life and Health Sciences, with explicit handling of the encoding artefacts, multilingual leakage, out-of-domain contamination, and critical polysemy that are common in broad biomedical literature exports.

---

## Table of Contents

- [Overview](#overview)
- [Who is this for?](#who-is-this-for)
- [Pipeline architecture](#pipeline-architecture)
- [Disambiguation strategies](#disambiguation-strategies)
- [Stopword layers](#stopword-layers)
- [Language filter](#language-filter)
- [Outputs](#outputs)
- [Requirements](#requirements)
- [Usage](#usage)
- [Input format](#input-format)
- [Reproducing the results](#reproducing-the-results)
- [Methodological rationale](#methodological-rationale)
- [Citation](#citation)
- [Author](#author)

---

## Overview

This pipeline preprocesses titles and abstracts from a Scopus CSV export and produces four VOSviewer-ready corpus files, one for the full corpus and one for each collaboration subgroup (HIC, LMIC, HIC-LMIC). It was developed for a corpus of **98,681 articles** by Brazilian authors covering all Life and Health Sciences in Scopus.

Every preprocessing decision is logged at the article level (`thesaurus_log.json`), making the entire transformation auditable for peer review or replication.

---

## Who is this for?

This pipeline is useful whenever you need to:

- Build a **keyword co-occurrence map** in VOSviewer from a large Scopus export
- Work with a **broad, cross-disciplinary biomedical corpus** where generic and polysemous terms inflate uninformative clusters
- Provide **transparent, reproducible preprocessing evidence** for a manuscript (e.g., in response to reviewer requests)
- Handle corpora with **multilingual abstract headers** (common in Latin American journals indexed in Scopus) or encoding artefacts from CSV exports

It is particularly suited to Health Sciences, Life Sciences, Public Health, Epidemiology, and related biomedical fields.

---

## Pipeline architecture

Each article's `title` and `description` (abstract) fields pass through five sequential steps:

```
Raw text
   │
   ▼
1. Encoding repair        latin-1/cp1252 artefacts → correct characters
   │
   ▼
2. Structural cleaning    HTML tags, in-text citations, p-values,
                          bare years, percentages → removed
   │
   ▼
3. Thesaurus substitution polysemous terms specified or normalised
                          via 60+ regex rules (see below)
   │
   ▼
4. Tokenisation + lemmatisation
                          NLTK WordNetLemmatizer, two passes
                          (noun then verb); compound tokens
                          produced by the thesaurus pass through
                          without lemmatisation
   │
   ▼
5. Stopword + language filtering
                          NLTK English stopwords + bibliometric
                          stopwords + multilingual stopwords +
                          English dictionary filter
   │
   ▼
Clean token string (space-separated, VOSviewer corpus format)
```

---

## Disambiguation strategies

Disambiguation follows the framework of Van Eck & Waltman (2010) and uses two strategies. No term is discarded; ambiguous terms are resolved by context.

### Specification

A polysemous term accompanied by a disambiguating qualifier is converted into an underscore-joined compound, which VOSviewer treats as a distinct node. The bare term without a qualifier is left intact.

| Original text | Canonical form | Domain |
|---|---|---|
| `polymer film` | `polymer_film` | Materials science |
| `thin film` | `thin_film` | Physics / engineering |
| `biofilm` | `biofilm` | Microbiology |
| `cell membrane` | `cell_membrane` | Cell biology |
| `drug resistance` | `drug_resistance` | Pharmacology |
| `insulin resistance` | `insulin_resistance` | Endocrinology |
| `cytotoxic agent` | `chemotherapy_drug` | Oncology |
| `cancer screening` | `cancer_screening` | Oncology |
| `disease burden` | `disease_burden` | Epidemiology |
| `extracellular matrix` | `extracellular_matrix` | Histology |
| `layer-by-layer` | `layer_by_layer_assembly` | Nanotechnology |

> **Example:** `film` alone (1,060 occurrences in the corpus) is left intact. Only `polymer film`, `thin film`, and `protective film` are qualified into compounds. This was the direct response to a reviewer comment on polysemy.

### Normalisation

Spelling variants and acronyms are mapped to a single canonical form, ensuring that different surface forms generate one node in the map.

| Variants | Canonical form | Notes |
|---|---|---|
| `WHO`, `World Health Organization` | `who` | ~8,170 articles; short token to avoid VOSviewer concatenation |
| `LMIC`, `developing country`, `low-resource setting`, `resource-limited setting` | `lmic` | ~312 articles |
| `HIC`, `high-income country` | `hic` | ~237 articles |
| `SUS` | `sus` | Sistema Único de Saúde; ~140 articles |
| `NCD`, `noncommunicable disease` | `noncommunicable_disease` | ~199 articles |
| `tumour` | `tumor` | British/American spelling |
| `leukaemia` | `leukemia` | British/American spelling |
| `paediatric` | `pediatric` | British/American spelling |
| `programme` | `program` | British/American spelling |

The four short-token anchor terms (`who`, `lmic`, `hic`, `sus`) are preserved as analytically informative nodes and should be interpreted accordingly in the VOSviewer map. They are documented in the code as `MONITOR_TERMS`.

---

## Stopword layers

Three complementary stopword sets are combined:

**1. NLTK English stopwords** (179 terms)
Standard function words.

**2. Bibliometric stopwords** (~80 terms)
Generic abstract-structure words that appear at high frequency in scientific literature but carry no thematic signal:
- Abstract section headers: `study`, `background`, `objective`, `method`, `result`, `conclusion`
- Generic quantifiers and comparatives: `high`, `significantly`, `associated`, `compared`
- Generic research terms: `data`, `analysis`, `model`, `sample`, `group`, `factor`
- Common temporal and spatial terms: `year`, `area`, `region`, `population`

**3. Multilingual stopwords** (~30 terms)
Portuguese and Spanish words that leak into abstracts through structured-abstract headers common in Latin American journals:
- PT: `objetivo`, `metodo`, `resultado`, `conclusao`, `foram`, `para`
- ES: `estudio`, `fueron`, `conclusiones`

---

## Language filter

After lemmatisation, each token is evaluated against three layers in order:

1. **Underscore token** (produced by the thesaurus) → pass through unchanged
2. **English dictionary** (`wordfreq` top-100,000 most frequent English words) → keep
3. **Scientific exceptions list** (~450 terms) → keep
4. Anything else → discard

The `SCIENTIFIC_EXCEPTIONS` set was built automatically from the 450 most frequent corpus tokens absent from the English dictionary and verified manually. It covers:

- Pathogen and parasite names (e.g., `cruzi`, `leishmania`, `schistosoma`)
- Tropical disease and vector terms (e.g., `trypomastigotes`, `lutzomyia`)
- Brazilian biome terms used as outcomes (e.g., `caatinga`, `cerrado`, `pantanal`)
- Specialised method acronyms (e.g., `qpcr`, `gwas`, `maldi`)
- Drug and molecule names (e.g., `trastuzumab`, `benznidazole`, `malondialdehyde`)
- Taxonomic and ecological terms (e.g., `microsatellites`, `phylogeographic`, `arbuscular`)

---

## Outputs

| File | Description |
|---|---|
| `vosviewer_total.csv` | All articles — full corpus |
| `vosviewer_hic.csv` | Articles led by high-income-country authors |
| `vosviewer_lmic.csv` | Articles led by low/middle-income-country authors |
| `vosviewer_hic_lmic.csv` | Articles with HIC + LMIC co-leadership |
| `corpus_preprocessed_full.csv` | Raw and clean fields side by side (audit file) |
| `thesaurus_log.json` | Per-article substitution log keyed by EID (methodological evidence) |

Each VOSviewer CSV follows the **corpus file format**: two columns, `label` (EID) and `text` (preprocessed token string).

---

## Requirements

```
python >= 3.9
pandas
nltk
tqdm
wordfreq
```

Install all dependencies:

```bash
pip install pandas nltk tqdm wordfreq
```

NLTK resources are downloaded automatically on first run:

```python
import nltk
for resource in ("stopwords", "wordnet", "omw-1.4"):
    nltk.download(resource, quiet=True)
```

---

## Usage

1. Clone the repository:

```bash
git clone https://github.com/PriAlbuquerque/corpus-preprocessing-pipeline.git
cd corpus-preprocessing-pipeline
```

2. Open `preprocess_vosviewer.ipynb` in Jupyter.

3. In **Cell 2**, set the paths to your input file and output directory:

```python
INPUT_FILE = "path/to/your_corpus.csv"
OUTPUT_DIR = "path/to/output_folder"
```

4. Run all cells in order.

5. Validate the pipeline by running **Cell 8** (sample test on 20 articles) before the full run in Cell 9.

6. Import any of the four output CSVs into VOSviewer:
   - *Create → Create a map based on text data → Corpus file*
   - Label column: `label` / Text column: `text`
   - Counting method: **Binary** (recommended)
   - Minimum occurrences: 10–20 for a corpus of ~100k articles; 5–10 for ~10k articles

---

## Input format

The pipeline expects a CSV file with at least the following columns:

| Column | Description |
|---|---|
| `eid` | Unique article identifier (Scopus EID) |
| `title` | Article title |
| `description` | Abstract |
| `score<hic>` | Binary flag: 1 if article is in the HIC subgroup |
| `score<lmic>` | Binary flag: 1 if article is in the LMIC subgroup |
| `score<hic lmic>` | Binary flag: 1 if article is in the HIC-LMIC subgroup |

The three score columns are used to generate the four output datasets. If you only need a single corpus file (no subgroups), the score columns are not required — simply comment out the subgroup export loop in Cell 12 and export only `df[["eid", "text_vos"]]`.

The pipeline expects **latin-1 encoding** (the default Scopus CSV export encoding). If your file uses UTF-8, change `encoding="latin-1"` to `encoding="utf-8"` in Cell 7.

---

## Reproducing the results

Every preprocessing decision is logged in `thesaurus_log.json`. Each entry has the structure:

```json
{
  "2-s2.0-85XXXXXXXXX": [
    {
      "field": "title",
      "pattern": "\\bdrug\\s+resistance\\b",
      "replacement": "drug_resistance",
      "strategy": "specification",
      "occurrences": 1,
      "rationale": "drug resistance"
    }
  ]
}
```

The disambiguation report (Cell 10) summarises total substitution counts per replacement term and per article, and includes a specific verification for the reviewer-flagged term `film`.

The audit CSV (`corpus_preprocessed_full.csv`) places raw and clean fields side by side for any article, enabling manual spot-checks of any substitution.

---

## Methodological rationale

| Step | Tool | Decision |
|---|---|---|
| Encoding repair | Regex (Python) | latin-1 artefacts mapped to correct characters |
| Structural cleaning | Regex (Python) | Remove HTML, citations, p-values, percentages |
| Disambiguation | Custom thesaurus (60+ rules) | Two strategies: specification and normalisation (Van Eck & Waltman, 2010) |
| Stopwords | NLTK + custom | 179 NLTK EN + bibliometric + multilingual (PT/ES) |
| Lemmatisation | NLTK WordNetLemmatizer | Two passes: noun (`pos='n'`) then verb (`pos='v'`) |
| Language filter | wordfreq top-100k + scientific exceptions | Removes non-English tokens not in the scientific vocabulary list |

**Out-of-domain contamination addressed:**

| Domain | Share of corpus | Approach |
|---|---|---|
| Agriculture / ecology | 16% | Terms specified or filtered via stopwords |
| Animal models | 9.7% | Terms filtered via stopwords |
| Materials science | 7% | Polysemous terms specified (`polymer_film`, `thin_film`, etc.) |

**Reference:**
Van Eck, N.J., & Waltman, L. (2010). Software survey: VOSviewer, a computer program for bibliometric mapping. *Scientometrics*, 84(2), 523–538. https://doi.org/10.1007/s11192-009-0146-3

---

## Author

**Priscila Albuquerque**  
GitHub: [@PriAlbuquerque](https://github.com/PriAlbuquerque)  
Grupo de Redes, Informação e Dados em Saúde (GRID)  
Centro de Desenvolvimento Tecnológico em Saúde (CDTS) / Fiocruz, Rio de Janeiro, Brazil

---

## License

MIT License. See `LICENSE` for details.
