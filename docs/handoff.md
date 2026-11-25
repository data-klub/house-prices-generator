# Handoff

Where the generator stands and what to build next. Read `CLAUDE.md` and `docs/spec.md` first.
Update this file in the PR that finishes or changes a milestone.

Last updated: 2026-11-25

## Current state

| Area | State |
|---|---|
| Code | Package `house_prices_generator` is empty |
| Tests | `tests/test_scaffold.py` (import, config values, Lab 01 determinism) |
| CI | ruff and pytest on ubuntu, windows and macos, plus a gitleaks scan |
| Lab 01 | Working AWS lab in `labs/01-s3-data-lake/`: a daily Lambda writes 1,000 simple listings to S3 |
| AWS team access | IAM users with MFA in group `data-klub-team`; setup in `infra/aws-team-access/` (House_Prices ADR 0011) |
| Tasks | Linear, one team per repo (House_Prices ADR 0010) |

## Milestones

Each step is one small PR with tests. House_Prices milestone 5 (the mock API adapter) needs G4.

1. **G1 Geography.** `reference/locations_pl.csv`: all 16 voivodeships, about 50 cities with every
   Warsaw district, a sample of towns and villages; TERYT codes as text; base price per m² and a
   monthly trend per location (spec §2.3). Lab 01's district prices are a starting point.
2. **G2 World.** Seeded properties placed by population weight; same seed gives the same world
   (spec §2.1–2.2).
3. **G3 Daily events.** Replay day by day from `world_start`: new listings, price changes, sold,
   withdrawn, reposts, cross-posts, gaps (spec §2.4). Fast enough for 20,000 properties.
4. **G4 Renderers and API.** OLX-like and Otodom-like cards and details (spec §2.5–2.6), served by
   FastAPI with `X-API-Key`, paging, 429 and 5xx (spec §3). `uv run mock-api`. Golden-file tests.
5. **G5 Truth export.** `truth/` files for properties, listing↔property map, events and renovation
   labels (spec §2.1, §2.7).
6. **G6 Edge cases.** The 28 injected cases in spec §2.8, each with a test.
7. **Release v0.1.0.** Tag it; House_Prices pins it and removes its empty `generator/` and
   `mock_api/` packages.

## Traps

- **Determinism:** derive every random choice from `seed` and the simulated date, never from
  `random` without a seed or from today's date.
- **Windows:** use `pathlib`, pass `encoding="utf-8"` on every file read and write, and write LF.
- **Lab 01** has its own README and needs `uv sync --group lab` (boto3). Its deployed Lambda is
  separate from this package.
