# CLAUDE.md

Behavioral guidelines to reduce common LLM coding mistakes (after Andrej Karpathy's observations), followed by project rules for Gata. Full product spec: `.claude/product.md`. Feature tracking: `TODO.md`.

Tradeoff: these bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

Don't assume. Don't hide confusion. Surface tradeoffs.

- State assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them. Don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop, name what's confusing, and ask.

## 2. Simplicity First

Minimum code that solves the problem. Nothing speculative.

- No features beyond what was asked.
- No abstractions for single-use code.
- No flexibility or configurability that wasn't requested.
- No error handling for impossible scenarios.
- If 200 lines could be 50, rewrite it.

Test: would a senior engineer call this overcomplicated? If so, simplify.

## 3. Surgical Changes

Touch only what you must. Clean up only your own mess.

- Don't "improve" adjacent code, formatting, or naming.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it. Don't delete it.
- Remove imports/variables/functions that your changes made unused.

Every changed line should trace directly to the request.

## 4. Goal-Driven Execution

Define success criteria. Loop until verified.

- "Fix the bug" → write a test that reproduces it, then make it pass.
- "Add validation" → write tests for invalid inputs, then make them pass.
- "Refactor X" → tests pass before and after.

For multi-step tasks, state a brief plan with a verify step for each:

```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak ones ("make it work") force constant clarification.

## Project: Gata

Natural-language Q&A over UP land records (Khatauni) from scanned, image-only PDFs. Pipeline: PDF → page images → cell grid → OCR → normalize → chunk per Gata → Chroma + BM25 + SQLite → query router → cited LLM answer.

### Code rules

- No code comments.
- Test-first for pure functions: OCR fixes, normalization, number checks, RRF, query routing.
- Type hints everywhere; dataclasses or pydantic for records.
- Small modules, no clever abstractions.
- Every pipeline stage reads and writes files on disk, so any stage can be re-run alone.
- No LangChain or LlamaIndex. Build each component by hand.
- Python 3.11, managed with uv. Run tests with `uv run pytest`.

### Data and OCR rules

- Never trust OCR numbers silently. Tesseract `hin` drops leading "1" (138 → 38, 1.439 → .439). Number cells get their own `eng` digit-whitelist pass.
- Cross-check: column 1 Gata equals the Gata in the column 5 unique code; column 8 shares sum to the column 6 total. Flag failing records, never fix them silently.
- Keep cell image path and raw OCR text with every field for audit.
- Deduplicate by Gata number and unique code (source files 1–3 may overlap).

### Answering rules

- LLM answers only from retrieved records.
- Cite Gata number and column/section for every fact.
- If the answer isn't in the records, say so plainly.
- Copy numbers exactly. Never round or compute in the LLM; arithmetic happens in code.
- Aggregations go to SQLite, not RAG.
- Report what the record says; no legal advice or interpretation.

### Evals

- Questions live in `eval/questions.yaml`, written by hand before building retrieval.
- Record a baseline before any fix. Every change gets a report in `eval/reports/` and a git tag.

### Privacy

- `data/` and real records never go into a public repo. README, blog and tests use redacted or synthetic records.
- Records contain personal data (names, shares, loans). Send the minimum context to hosted LLMs; prefer local.

### Out of scope

Web UI, villages beyond the first 3–5, fine-tuning.

## Working agreement

- Work milestone by milestone (M1 → M6). Don't start a milestone's features before the previous one's "done when" is met.
- When a feature is finished, tick it in `TODO.md`.
- Each concept should leave a trace for the writeup: what broke, what was tried, what the numbers showed. Add a line to `docs/notes.md` when something notable happens.
