# TODO

Target: M0–M4 plus first evals by 30 Nov 2026. M6 after applications start (early Dec 2026).

## M0 Setup + data check

- [x] Inspect PDFs (image-only, jsPDF, 3141×2154 JPEG pages)
- [x] Test Tesseract `hin` on names and numbers
- [x] Document Khatauni columns and data quirks
- [x] Choose OCR approach

## Project setup

- [ ] `uv init`, Python 3.11, dependencies in pyproject.toml
- [ ] Create repo layout (data/, src/gata/, eval/, tests/)
- [ ] `git init`, first commit
- [ ] Install Tesseract 5 with `hin` (tessdata_best) and `eng`
- [ ] Decide LLM provider and local-vs-hosted policy
- [ ] Redacted/synthetic sample record for README

## Eval prep (before retrieval)

- [ ] Write 80–100 questions in `eval/questions.yaml`
- [ ] Cover misspelled names, ownership changes, loans, aggregations, no-answer
- [ ] `eval/run_eval.py` skeleton

## M1 Extraction

- [ ] Render pages from PDFs (PyMuPDF / pdfimages)
- [ ] Remove red popup overlay by colour mask
- [ ] Handle diagonal watermark
- [ ] Detect table lines with OpenCV and build cell grid
- [ ] Crop cells with 2–3× upscaling
- [ ] OCR text cells (`hin`, psm 6)
- [ ] OCR number cells (`eng`, digit whitelist, psm 7)
- [ ] Parse col 2 owners (name / father or husband / caste / address)
- [ ] Parse cols 1, 3–10 and the removed/added cols 11–16
- [ ] Parse page header (district, tehsil, village, code, year, khatedar, shreni)
- [ ] Check: col 1 Gata equals Gata in col 5 unique code
- [ ] Check: col 8 shares sum to col 6 total
- [ ] Flag failed checks in the record
- [ ] Store cell image path and raw OCR text per field
- [ ] Deduplicate files 1–3 by Gata number and unique code
- [ ] Hand-type 3 golden records and tests
- [ ] Raw JSON per Gata in `data/extracted/`
- [ ] Per-column OCR accuracy report vs golden

## M2 Normalization

- [ ] OCR number fixes (dropped leading "1", etc.)
- [ ] Abbreviation expansion
- [ ] Name parsing (name / father / husband)
- [ ] Transliteration Devanagari ↔ Roman
- [ ] Fuzzy name index (rapidfuzz)
- [ ] Name-recall tests: "Dushyant" / "दुष्यन्त" / "Dushyand"
- [ ] Clean JSON per Gata in `data/clean/`

## M3 Chunk + index

- [ ] One chunk per Gata with metadata
- [ ] SQLite schema and loader
- [ ] BM25 index
- [ ] Multilingual embeddings + ChromaDB
- [ ] Each store answers queries on its own

## M4 Retrieval + answer

- [ ] Identifier detection (Gata, 15-digit code, village code) as exact filters
- [ ] Query name normalization same as index
- [ ] Hybrid search with Reciprocal Rank Fusion
- [ ] Query router (filter / hybrid / SQL)
- [ ] LLM interface (swappable provider)
- [ ] Answer prompt: citations, refusals, exact numbers
- [ ] Arithmetic in code, not LLM
- [ ] `gata ask "..."` CLI
- [ ] First eval report in `eval/reports/`

## M5 Eval loop

- [ ] Retrieval and faithfulness metrics
- [ ] LLM-as-judge
- [ ] Baseline report and git tag
- [ ] Before/after report for each change
- [ ] Hallucination and refusal analysis

## M6 Browser agent (after applications start)

- [ ] Playwright flow on upbhulekh.gov.in
- [ ] Human-in-the-loop for captcha/login
- [ ] `gata fetch --village nainu-nangal --khasra 138`
- [ ] Auto re-index after fetch

## Later

- [ ] Add 3–5 neighbouring villages
- [ ] Blog writeup per concept: what broke, what was tried, numbers
- [ ] Polished README with redacted demo
