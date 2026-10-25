# Gata

Natural-language Q&A over Uttar Pradesh land records (Khatauni). Ask about a plot or an owner in plain language and get an accurate answer with the source record cited.

Example:

> Q: Who owns Gata 138 and what is each share?
> A: Gata 138 (1.439 ha) has 2 co-owners: Kailash Singh (1/2, 0.7195 ha) and Dushyant Singh (1/2, 0.7195 ha). Source: Khatauni Nainu Nangal, Gata 138, columns 2, 7, 8.

## Why this exists

- Portfolio project for moving into AI Engineer roles. Applications start in early December 2026.
- The owner knows these records first-hand, so wrong answers are easy to spot.
- The data is hard in realistic ways: image-only PDFs, Devanagari text, OCR errors in numbers, many co-owners per plot, mutation chains.

## Concepts this project covers

| Concept | Where | Depth |
|---|---|---|
| OCR on scanned documents | M1 | Deep |
| Table structure detection | M1 | Working |
| Text normalization and OCR error correction | M2 | Deep |
| Transliteration (Devanagari ↔ Roman) | M2 | Working |
| Entity resolution and fuzzy matching | M2 | Deep |
| Chunking strategy (one chunk per Gata) | M3 | Deep |
| Metadata filtering | M3–M4 | Deep |
| Embeddings (multilingual) | M3 | Working |
| Vector database (ChromaDB) | M3 | Working |
| Keyword search (BM25) | M3–M4 | Deep |
| Hybrid search and Reciprocal Rank Fusion | M4 | Deep |
| Query routing (filter / hybrid / SQL) | M4 | Deep |
| RAG and grounded generation | M4 | Deep |
| Prompt engineering for citations and refusals | M4 | Deep |
| Hallucination control | M4–M5 | Deep |
| Evals: golden sets, LLM-as-judge, regression tracking | M5 | Deep |
| Retrieval and faithfulness metrics | M5 | Deep |
| Browser agents with human-in-the-loop | M6 | Working |

Depth: Working = can use it to solve real problems. Deep = knows the failure modes, can debug and teach it.

Each concept should leave a trace in the writeup: what broke, what was tried, what the numbers showed.

## Scope

- District Bijnor, tehsil Chandpur, village Nainu Nangal (village code 114412).
- Source: Khatauni copies downloaded from upbhulekh.gov.in.
- Later: 3–5 neighbouring villages, so retrieval is actually tested on a few hundred Gatas.

## What we know about the data (M0 findings)

- PDFs are made by jsPDF and hold only JPEG screenshots of the Bhulekh page (3141×2154 px per page). There is no text layer, so pdfplumber gets nothing and OCR is required.
- All text is Hindi (Devanagari) except the numbers.
- Tesseract with `hin` reads names and headings well.
- Tesseract with `hin` drops the leading "1" in numbers: 138 → 38, 1.439000 → .439000, 114412 → 74472. Number cells need their own OCR pass.
- A red popup overlay ("कृपया उक्त खसरे…") covers column 5 on the first page.
- A diagonal "अप्रमाणित प्रति" watermark sits over the table.
- Files 1, 2 and 3 are nearly identical and may be duplicates of the same khata. Deduplicate by Gata number and unique code.
- The table layout is fixed, so cells can be located by their grid position.

### Khatauni columns

| Col | Meaning |
|---|---|
| 1 | Khasra / Gata number |
| 2 | Owners: name / father or husband / caste code / address |
| 3 | Basis of ownership: court, case or order number, date |
| 4 | Year (fasli) |
| 5 | Gata number with 15-digit unique code, e.g. 138 (114412013800012) |
| 6 | Total area of the Gata (ha) |
| 7 | Share fraction per owner |
| 8 | Area per owner (ha) |
| 9 | Land revenue owed (Rs) |
| 10 | Mutation basis: court, order, date, digital signature |
| 11–13 | Removed: owner, Gata, area |
| 14–16 | Added: owner, Gata, area |

The page header also holds district, tehsil, village, village code, agricultural year, khatedar name and shreni (land category).

## Architecture

```
PDF → page images → table grid detection → cell crops
    → OCR (hin for text cells, eng digits for number cells)
    → raw record per Gata
    → normalize (abbreviations, OCR fixes, transliteration, name parsing, fuzzy index)
    → clean record per Gata
    → one chunk per Gata + metadata → ChromaDB + BM25 + SQLite
    → query router (ID filter / hybrid search / SQL aggregation)
    → LLM answer with citations
```

## Milestones

| | Milestone | Done when |
|---|---|---|
| M0 | Setup + data check | Done. Image-only PDFs found, OCR approach chosen |
| M1 | Extraction | Every Gata has a raw JSON record; 3 hand-typed golden records pass as tests |
| M2 | Normalization | Name-recall tests pass ("Dushyant" / "दुष्यन्त" / "Dushyand" match) |
| M3 | Chunk + index | Chroma, BM25 and SQLite each answer queries on their own |
| M4 | Retrieval + answer | `gata ask "..."` returns a cited answer; first eval report saved |
| M5 | Eval loop | Before/after numbers for every change, tagged in git |
| M6 | Browser agent | `gata fetch --village nainu-nangal --khasra 138` downloads and re-indexes |

Target: M0–M4 plus first evals by 30 Nov 2026. M6 comes after applications start.

## Extraction approach (M1)

1. Render each page and remove the red overlay by masking its colour.
2. Find horizontal and vertical table lines with OpenCV and build the cell grid.
3. Crop each cell, upscaling 2–3× for small text.
4. OCR text cells with `hin` (psm 6) and number cells with `eng`, a digit whitelist and psm 7.
5. Check numbers against each other: the column 1 Gata must match the Gata in the column 5 unique code, and the column 8 shares must add up to the column 6 total. Flag a record when a check fails, never silently.
6. Keep the cell image path and raw OCR text with every field for audit.

## Retrieval

- Detect identifiers first (Gata number, 15-digit unique code, village code) and use them as exact metadata filters.
- Run BM25 and vector search in parallel and merge the results with Reciprocal Rank Fusion.
- Normalize names in the query the same way as the index (transliteration plus fuzzy match).
- Route aggregation questions ("total area", "how many plots") to SQLite, not RAG.

## Answering

- The LLM answers only from the retrieved records.
- Cite the Gata number and column or section for every fact.
- If the answer isn't in the records, say so plainly.
- Copy numbers exactly from the record. Never round or compute in the LLM; do arithmetic in code.

## Evaluation

- Use 80–100 questions written by hand from the PDFs, before building retrieval. Store them in `eval/questions.yaml`.
- Include misspelled names, ownership changes, loans, aggregations and questions with no answer.
- Metrics and targets:

| Metric | Target |
|---|---|
| Retrieval accuracy (right Gata in top 3) | ≥ 98% |
| Answer accuracy | ≥ 95% |
| Hallucination rate | < 2% |
| Numeric exact match | 100% |
| Correct refusal on no-answer questions | ≥ 95% |
| OCR field accuracy vs golden records | tracked per column |

- Record a baseline before any fix. Every change gets a report in `eval/reports/` and a git tag.

## Tech stack

| Layer | Choice |
|---|---|
| Language | Python 3.11, uv |
| PDF → image | pdfimages / PyMuPDF |
| Table grid | OpenCV |
| OCR | Tesseract 5 with `hin` (tessdata_best) and `eng` |
| Transliteration | indic-transliteration |
| Fuzzy match | rapidfuzz |
| Embeddings | paraphrase-multilingual-MiniLM-L12-v2 |
| Vector store | ChromaDB (local) |
| Keyword search | rank_bm25 |
| Structured store | SQLite |
| LLM | One provider behind a small interface, so models can be swapped |
| Browser agent | Playwright |
| Evals | LLM-as-judge + manual checks; RAGAS optional |
| Tests | pytest |

No LangChain or LlamaIndex. Every component is built by hand so its behaviour is understood.

## Repo layout

```
gata/
  data/raw/         PDFs (gitignored)
  data/pages/       page images (gitignored)
  data/extracted/   raw JSON per Gata
  data/clean/       normalized JSON per Gata
  data/index/       Chroma, BM25, SQLite
  src/gata/
    extract/        pages, grid, ocr, record
    normalize/
    index/
    retrieve/
    answer/
    fetch/
    cli.py
  eval/
    questions.yaml
    run_eval.py
    reports/
  tests/
    golden/         hand-typed records + cell images
```

## Code rules

- No code comments.
- Test-first for pure functions: OCR fixes, normalization, number checks, RRF, query routing.
- Type hints everywhere; dataclasses or pydantic for records.
- Small modules, no clever abstractions.
- Every pipeline stage reads and writes files on disk, so any stage can be re-run alone.

## Privacy

- Real PDFs, page images, extracted records and eval answers never go into a public repo.
- The README and blog use redacted or synthetic records.
- Records include co-owners' names, shares and loans. Treat them as personal data.
- Prefer a local LLM, or send only the minimum context to a hosted one.

## Out of scope for now

- Web UI (CLI first)
- Villages beyond the first 3–5
- Fine-tuning
- Legal advice or interpretation; Gata reports what the record says, nothing more