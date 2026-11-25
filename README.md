# house-prices-generator

A seeded simulator of the Polish housing market. It keeps a hidden world of properties, changes it
day by day (new listings, price cuts, sales, reposts) and serves the ads through a mock HTTP API
that looks like two real listing sites: OLX-like and Otodom-like.

It is the fake data source for **[House_Prices](https://github.com/data-klub/House_Prices)**, the
main project of **Data Klub**, a group of data people who learn by building production-style
pipelines together.

**Status: started.** The design is written ([docs/spec.md](docs/spec.md)) and the milestones are
in [docs/handoff.md](docs/handoff.md). The package is empty so far. The repo also holds
[Lab 01](labs/01-s3-data-lake/README.md), a small AWS data lake whose `generate.py` was the first,
much simpler version of this generator.

All data is synthetic. No real listings and no credentials are stored in this repo.

## How it fits with House_Prices

```
house-prices-generator                         House_Prices
seeded world ─> daily events ─> mock API  ──HTTP──>  adapter ─> bronze ─> silver ─> gold
      │                                               (config/pipeline.yaml picks the source)
      └─> truth/  (answer key for tests only)
```

House_Prices talks to the generator **only over HTTP**. It never imports this code, and its
pipeline never reads `truth/`. That is what lets someone replace the fake data with a real source
later by changing only House_Prices' config, adapter and parser
([ADR 0008](https://github.com/data-klub/House_Prices/blob/main/docs/adr/0008-generator-separate-sources-by-config.md)).

What House_Prices depends on, so a change to any of these needs a matching change there:

| What | The contract |
|---|---|
| API paths and parameters | Endpoints in spec §3, 50 cards per page, `page`, `region`, `category` / `type`, `as_of` |
| HTTP behaviour | `X-API-Key` required; 429 with `Retry-After`; 404 or 410 for a removed listing; occasional 5xx |
| Payload shapes | OLX-like and Otodom-like card, detail and development formats, pinned by golden-file tests |
| Listing ids | Text, stable for the life of a listing; a repost gets a new id |
| Determinism | Same `seed` + same date gives byte-identical responses |
| Start command | `uv run mock-api` on `localhost:8000` |

Releases are git tags (`v0.1.0`, …). House_Prices installs a pinned tag for its tests and daily
workflow.

## Setup

Everything runs natively on macOS and Windows, with no WSL or Docker. You need
[Git](https://git-scm.com/downloads) and [uv](https://docs.astral.sh/uv/), which installs
Python 3.12 for you.

```bash
git clone https://github.com/data-klub/house-prices-generator.git
cd house-prices-generator
uv sync
uv run pre-commit install
uv run pytest
```

Everyday commands, the same on both platforms:

```bash
uv run pytest                  # tests
uv run ruff check .            # lint
uv run ruff format .           # format
```

## Layout

```
config/generator.yaml    seed, world size, API settings
docs/                    spec (world, events, formats, API) and milestones
src/house_prices_generator/
tests/
labs/01-s3-data-lake/    Lab 01: AWS warm-up (Lambda, S3, Athena)
infra/aws-team-access/   how members get their own AWS login (IAM policies)
data/, truth/            generated locally, never committed
```

## How we work

Data Klub meets every Wednesday at 19:00 (Warsaw time) in a Discord voice channel, and chats in
Discord. Pick a milestone from [docs/handoff.md](docs/handoff.md), open a branch, and send a small
pull request with tests. Decisions that affect both repos are recorded as ADRs in House_Prices.
