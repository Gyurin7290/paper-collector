# paper-collector

A Claude skill that finds research papers by **field, date range, keywords and target journals**, downloads the legally open-access PDFs into a folder, and writes an Excel list of every paper (downloaded / paywalled / failed) with DOI links.

- Search: OpenAlex (title/abstract + full-text), Crossref fallback
- PDFs: OpenAlex OA locations, publisher OA links (Nature, Wiley, ACS), Unpaywall (PMC, repositories, preprints)
- No shadow libraries, no credential entry

## Install
Copy this folder into your skills directory, or upload the packaged `.skill` file in Claude.

## Script usage
```bash
python scripts/collect_papers.py search --field peptide --core "assembly,self-assembly" \
  --any "molecular dynamics,conductivity,piezoelectric" \
  --venues "ACS Nano,JACS,Angew,Nat Commun,Science,Adv Mater,Nature Chemistry" \
  --from 2023-10-23 --to 2026-10-23 --fulltext --out candidates.json --mailto you@example.org
python scripts/collect_papers.py download --in curated.json --dest ./reference --mailto you@example.org
python scripts/collect_papers.py sheet --in curated.json --out paper_list.xlsx
```
See `SKILL.md` for the full workflow (including the curation step).
