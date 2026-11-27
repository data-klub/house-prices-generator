# CLAUDE.md — house-prices-generator

Guidance for AI coding assistants working in this repo. Read this file first, then
`docs/spec.md` (the design), then `docs/handoff.md` (milestones).

## What this project is

A seeded simulator of the Polish housing market that serves OLX-like and Otodom-like listings
through a mock HTTP API. It is the fake data source for
[DataKlubform](https://github.com/data-klub/DataKlubform), which crawls the API daily. Both are
Data Klub learning projects.

## Hard rules

- **Determinism:** same `seed` + same date ⇒ byte-identical output. Never use unseeded randomness
  or wall-clock time inside the generator.
- **The API and payload formats are a contract with DataKlubform.** Change them only together with
  the DataKlubform parsers, and keep golden-file tests for them.
- **`truth/` is an answer key for tests.** DataKlubform pipeline code never reads it.
- **Only synthetic data.** No real scraped listings, names, phones or emails, ever.
- **No secrets in code, config or chat.** The API key comes from the `MOCK_API_KEY` environment variable.

## Team and platforms

- Several contributors on macOS and Windows; everything runs natively (no WSL, no Docker).
- Use `pathlib`; text files are UTF-8 with LF (`.gitattributes`).
- Field and column names are in English; generated content (titles, descriptions, labels) is in
  Polish (ADR 0006). Polish text keeps diacritics (Łódź, Białołęka). Prices are in PLN.

## Tooling

Python 3.12 with uv; ruff; pytest; pre-commit (ruff, end-of-file, gitleaks). FastAPI and uvicorn
for the mock API, once milestone G4 adds them.

```bash
uv sync
uv run pytest
uv run ruff check . && uv run ruff format .
```

## Conventions

- Small PRs, one milestone step each; every PR has tests. Commit style: `feat:`, `fix:`, `test:`,
  `docs:`, `chore:`.
- Public functions have type hints and a one-line docstring.
- Decisions that affect both repos get an ADR in DataKlubform `docs/adr/`.

## Teaching mode

Some contributors are learning. When asked to explain, explain the *why* before the *how*, and
prefer guiding with steps and hints over writing the full solution, unless asked for the code.
