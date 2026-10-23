---
name: paper-collector
description: Find research papers by field, date range, keywords and target journals, then download the open-access PDFs into the user's reference folder and list the rest (paywalled) in an Excel sheet with DOI links. Use whenever the user asks to 찾아서 논문 다운로드, 레퍼런스 수집, 문헌조사, literature search, "papers from Nature/JACS/ACS Nano/Angew… on X in the last N years", or wants PDFs saved into a folder — even if they only list 분야/기간/키워드/venue without saying "skill".
---

# Paper collector

Turns a request like "분야: peptide assembly / 기간: 최근 3년 / 키워드: MD, conductivity / venue: JACS, ACS Nano… → PDF를 reference 폴더에" into:
1. a curated list of matching papers,
2. legally free PDFs saved in the user's folder,
3. an Excel list (`<topic>_<from>-<to>.xlsx`) of every paper with its status (downloaded / paywalled / failed).

## 1. Pin down the request (ask once, with AskUserQuestion, only what's missing)

- **Keyword logic.** Users usually give one *core* topic plus *aspect* keywords (e.g. "peptide assembly, MD, conductivity"). The sensible default is `core AND (aspect1 OR aspect2)`; strict AND usually returns a handful of papers and OR floods hundreds. Confirm if unclear.
- **Date range.** "최근 N년" = today minus N years → today. If their example dates disagree with N (e.g. "3년 (2024-10-23~2026-10-23)"), go with N but say so, and keep the date column so they can filter.
- **Destination folder.** A path on their computer → `device_request_folder_access` on exactly that folder.
- **Paywalled papers.** Default: download only open-access copies (publisher OA, PMC, institutional repositories, ChemRxiv/arXiv preprints) and list the rest. Never use shadow libraries (Sci-Hub, LibGen etc.); never enter the user's institutional credentials.

Expand each aspect keyword into the phrasings authors actually use — this matters more than anything else for recall:
- MD → "molecular dynamics", "MD simulation(s)", "coarse-grained", "all-atom"
- conductivity → conductivity, conductive, conducting, conductance, "charge transport", "electron transport", "proton conduction"
- assembly → assembly, "self-assembly", "self-assembled", "co-assembly", nanofiber, supramolecular

## 2. Search (OpenAlex)

Run `scripts/collect_papers.py search` (see its docstring). It resolves journal names to OpenAlex source IDs (accepts abbreviations like JACS, Angew, Nat Commun), then queries:
- `title_and_abstract.search` — precise; but ACS/Wiley abstracts are often missing in OpenAlex, so this alone misses papers.
- `--fulltext` adds the relevance `search=` over full text — high recall, lots of noise (methods sections mention "MD" in passing; "conduct" also matches "conducted"). Always use it, then curate.

Notes learned the hard way:
- Cursor paging now requires an OpenAlex API key; use `page=1..N` (the script does). 2 pages × 200 is plenty after relevance ranking.
- Pass the user's email as `--mailto` (polite pool).
- Without an API key OpenAlex has a small free daily budget per IP; when it answers "Rate limit exceeded / Insufficient budget", switch to Crossref (`https://api.crossref.org/journals/<ISSN>/works?query.bibliographic=...&filter=from-pub-date:...,until-pub-date:...`) and filter titles yourself.
- For "conductivity"-type requests, also search related electrical properties (piezoelectric, ferroelectric, proton conduction) as a separate tagged group — users in this area usually want them.

## 3. Curate (this is the step that makes the result useful)

Read every candidate's title (and abstract when present) and keep only papers where the **core topic is the subject of the paper** and the aspect keyword is actually used, not just mentioned. Drop e.g. clinical amyloid papers, protein structure papers, drug-delivery papers that merely contain a peptide, and regex false positives ("conducted"). Tag each kept paper with `keyword` (MD / conductivity / both) and keep `match` (title_abstract vs fulltext) so the user knows how strong the evidence is. Save as `curated.json`. Mention borderline groups you excluded (e.g. piezoelectric papers when they asked for conductivity) so the user can ask for them.

## 4. Download

`scripts/collect_papers.py download --in curated.json --dest <folder> --mailto <email>` tries, in order: OpenAlex `best_oa_location.pdf_url`, publisher direct-PDF patterns (nature.com `/articles/<id>.pdf`, Wiley `pdfdirect`, ACS `/doi/pdf`), then Unpaywall's other OA locations. It checks the `%PDF-` header because publishers often return an HTML page instead of a PDF, and names files `YYYY_Journal_First_Words_Of_Title.pdf`.

Then `scripts/collect_papers.py sheet --in curated.json --out <topic>_<from>-<to>.xlsx`.

### Where to run it (Cowork)
- Work happens in the cloud workspace, then results go to the user's folder with `device_commit_files` (≤50 files per call — batch it).
- **If the cloud network blocks** `api.openalex.org` / publisher hosts (curl returns 000, proxy "connect_rejected"): the search can still run through the built-in browser (allow `api.openalex.org`, then `fetch()` from the page in `javascript_tool` — same query strings as the script). PDFs cannot be moved from the browser to disk, so write the Excel list first, deliver it, and tell the user which domains to allow (Settings > Capabilities): `api.openalex.org, api.unpaywall.org, www.nature.com, www.science.org, pubs.acs.org, onlinelibrary.wiley.com, chemrxiv.org, pmc.ncbi.nlm.nih.gov, europepmc.org, www.ebi.ac.uk` — then rerun step 4.

## 5. Report

One short message: how many papers matched, how many PDFs landed in the folder, how many are paywalled (listed in the Excel with DOI links so they can grab them via their library), and any interpretation you made (date range, keyword logic, excluded borderline topics).
