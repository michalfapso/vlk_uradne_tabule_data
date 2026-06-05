# Data — Agent Guide

The `data/` directory is a **git submodule** (separate repository) — changes here must be
committed and pushed independently from the main repo. The main repo tracks the submodule
pointer, updated automatically by CI.

## Directory Layout

```
data/
├── scraped/                  Raw JSON from scrapers (committed)
│   ├── minv_1_regions.json
│   ├── minv_2_documents.json
│   ├── minv_2_documents_old.json
│   ├── minzp_2_documents.json
│   └── minzp_2_documents_old.json
├── docs/                     Per-document analysis tree (committed)
│   └── {source}/{kraj}/{okres}/{docId}/
│       ├── meta.json         Document metadata from scraper
│       ├── analysis.json     LLM analysis output
│       ├── text.txt          Extracted document text
│       ├── parcels.geojson   Cadastral parcel geometry (if found)
│       └── status.json       Processing errors/warnings
├── laws/                     Law registry (committed)
│   ├── registry.json         Regex patterns for identifying laws by name/number
│   ├── 543-2002.json         Full text index for nature protection law
│   └── 71-1967.json          Full text index for administrative procedure law
├── protected_areas/          GIS shapefiles — GITIGNORED, synced from CI artifacts
└── cadaster/                 Cadastral boundary data — GITIGNORED
```

Path example: `data/docs/minzp/Košický kraj/Spišská Nová Ves/ou-sn-oszp-2020-005175/`

Source is either `minv` or `minzp`. Kraj (region) and okres (district) come from the scraper.

## `meta.json` Schema

Stored at `data/docs/{source}/{kraj}/{okres}/{docId}/meta.json`.

```json
{
  "url": "https://www.minzp.sk/...",
  "source": "minzp",
  "datum": "2020-02-12",
  "nazov": "OU-SN-OSZP-2020/005175",
  "kraj": "Košický kraj",
  "okres": "Spišská Nová Ves",
  "kategoria": null
}
```

Fields: `url` (original document URL), `source` (`minv`|`minzp`), `datum` (ISO date published),
`nazov` (document title/reference number), `kraj` (region), `okres` (district), `kategoria`
(document category, may be null for minzp).

## `analysis.json` Schema

Produced by `analyzer/shared/llm_analyzer.py`. Key fields:

| Field | Type | Description |
|---|---|---|
| `cislo_konania_spisu` | string | Official case/file number (spis) |
| `cislo_rozhodnutia` | string\|null | Decision number if different from case number |
| `datum_dokumentu` | string | ISO date of document |
| `datum_zverejnenia` | string | ISO date when published on notice board |
| `faza_konania` | string | Phase of proceeding |
| `ucast_v_konani.povolena` | bool\|null | Whether public participation is allowed |
| `ucast_v_konani.lehota_na_vyjadrenie` | string\|null | Deadline for submitting comments |
| `ziadatel_navrhovatel` | string | Applicant name |
| `miesto_realizacie.kraj` | string | Region |
| `miesto_realizacie.okres` | string | District |
| `miesto_realizacie.obec` | string | Municipality |
| `miesto_realizacie.katastralne_uzemia` | array | List of cadastral zones with parcel numbers |
| `miesto_realizacie.nazov_lokality` | string | Exact locality name from document |
| `miesto_realizacie.nazov_lokality_norm` | string | Normalized locality name for Nominatim lookup |
| `typ_dokumentu` | string | Document type |
| `kategorie_vlk` | string[] | VLK-specific relevance categories |
| `typ_zasahu` | string[] | Types of intervention (e.g. "výrub drevín") |
| `typ_uzemia` | string[] | Territory type descriptions |
| `je_v_chranenom_uzemi` | bool\|null | Whether location is in a protected area |
| `dotknute_zivocichy_rastliny` | string[] | Affected species |
| `paragrafy` | array | Law paragraphs cited (each has nazov, cislo, paragraf, odsek) |
| `zhrnutie` | string | AI-generated Slovak-language summary |

## Scraped Data Structure

`data/scraped/minv_2_documents.json` — nested: list of krajov → list of okresy → list of
kategorie dokumentov → list of documents (each with `datum`, `nazov`, `url`).

`data/scraped/minzp_2_documents.json` — flat list of documents (each with `datum`, `nazov`,
`url`, `kraj`, `okres`).

## Gitignored Files

`data/protected_areas/` and `data/cadaster/` contain large GIS shapefiles that are NOT committed.
They are uploaded as CI artifacts by the process workflow and must be synced locally with:
```bash
bash analyzer/sync_github_data.sh
```
