# Gata

Natural-language Q&A over Uttar Pradesh land records (Khatauni). Ask about a plot or an owner in plain language and get an accurate answer with the source record cited.

> Q: Who owns Gata 138 and what is each share?
> A: Gata 138 (1.439 ha) has 2 co-owners: Kailash Singh (1/2, 0.7195 ha) and Dushyant Singh (1/2, 0.7195 ha). Source: Khatauni Nainu Nangal, Gata 138, columns 2, 7, 8.

The examples in this repo use redacted or synthetic records. Real records are personal data and are not published.

## Status

Planned. M0 (setup and data check) is done. See [TODO.md](TODO.md) for progress and [.claude/product.md](.claude/product.md) for the full spec.

## Why this is hard

- Source PDFs are image-only JPEG screenshots with no text layer.
- Text is Devanagari; numbers need a separate OCR pass because Tesseract `hin` drops leading digits.
- A popup overlay and a diagonal watermark cover parts of the table.
- Many co-owners per plot, mutation chains, and name spelling variants.

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

No LangChain or LlamaIndex. Every component is built by hand.

## Milestones

| | Milestone | Done when |
|---|---|---|
| M0 | Setup + data check | Done |
| M1 | Extraction | Every Gata has a raw JSON record; 3 golden records pass as tests |
| M2 | Normalization | Name-recall tests pass |
| M3 | Chunk + index | Chroma, BM25 and SQLite each answer queries alone |
| M4 | Retrieval + answer | `gata ask "..."` returns a cited answer; first eval report saved |
| M5 | Eval loop | Before/after numbers for every change, tagged in git |
| M6 | Browser agent | `gata fetch --village nainu-nangal --khasra 138` downloads and re-indexes |

## Evaluation targets

| Metric | Target |
|---|---|
| Retrieval accuracy (right Gata in top 3) | ≥ 98% |
| Answer accuracy | ≥ 95% |
| Hallucination rate | < 2% |
| Numeric exact match | 100% |
| Correct refusal on no-answer questions | ≥ 95% |

## Setup

Prerequisites: Python 3.11, [uv](https://docs.astral.sh/uv/), Tesseract 5 with `hin` and `eng` data.

```
brew install tesseract tesseract-lang
uv sync
uv run pytest
```

Place source PDFs in `data/raw/` (gitignored).

## Usage (planned)

```
gata extract
gata normalize
gata index
gata ask "Who owns Gata 138?"
gata fetch --village nainu-nangal --khasra 138
```

## Repo layout

```
data/        raw, pages, extracted, clean, index (gitignored where private)
src/gata/    extract, normalize, index, retrieve, answer, fetch, cli.py
eval/        questions.yaml, run_eval.py, reports/
tests/       golden records and cell images
```

## Scope

District Bijnor, tehsil Chandpur, village Nainu Nangal (code 114412), then 3–5 neighbouring villages. Gata reports what the record says and gives no legal advice.
