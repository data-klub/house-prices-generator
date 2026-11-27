# house-prices-generator

A seeded simulator of the Polish housing market. It keeps a hidden world of properties, changes it
day by day (new listings, price cuts, sales, reposts) and serves the ads through a mock HTTP API
that looks like two real listing sites: OLX-like and Otodom-like.

It is the fake data source for **[DataKlubform](https://github.com/data-klub/DataKlubform)**, the
main project of **Data Klub**, a group of data people who learn by building production-style
pipelines together.

**Status: started.** The design is written ([docs/spec.md](docs/spec.md)) and the milestones are
in [docs/handoff.md](docs/handoff.md). The package is empty so far. The repo also holds
[Lab 01](labs/01-s3-data-lake/README.md), a small AWS data lake whose `generate.py` was the first,
much simpler version of this generator.

All data is synthetic. No real listings and no credentials are stored in this repo.

## How it fits with DataKlubform

```
house-prices-generator                         DataKlubform
seeded world ─> daily events ─> mock API  ──HTTP──>  adapter ─> bronze ─> silver ─> gold
      │                                               (config/pipeline.yaml picks the source)
      └─> truth/  (answer key for tests only)
```

DataKlubform talks to the generator **only over HTTP**. It never imports this code, and its
pipeline never reads `truth/`. That is what lets someone replace the fake data with a real source
later by changing only DataKlubform's config, adapter and parser
([ADR 0008](https://github.com/data-klub/DataKlubform/blob/main/docs/adr/0008-generator-separate-sources-by-config.md)).

What DataKlubform depends on, so a change to any of these needs a matching change there:

| What | The contract |
|---|---|
| API paths and parameters | Endpoints in spec §3, 50 cards per page, `page`, `region`, `category` / `type`, `as_of` |
| HTTP behaviour | `X-API-Key` required; 429 with `Retry-After`; 404 or 410 for a removed listing; occasional 5xx |
| Payload shapes | OLX-like and Otodom-like card, detail and development formats, pinned by golden-file tests |
| Language | English field names, Polish values and labels (spec §2.5–2.6, glossary in §2.6.1) |
| Listing ids | Text, stable for the life of a listing; a repost gets a new id |
| Determinism | Same `seed` + same date gives byte-identical responses |
| Start command | `uv run mock-api` on `localhost:8000` |

Databricks jobs that load and transform the data live in a third repo,
[databricks-jobs](https://github.com/data-klub/databricks-jobs).

Releases are git tags (`v0.1.0`, …). DataKlubform installs a pinned tag for its tests and daily
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
Discord. Tasks live in Linear, one team per repo (ask Kubi for an invite). Pick a milestone from
[docs/handoff.md](docs/handoff.md), open a branch, and send a small pull request with tests that
names its Linear issue. Decisions that affect both repos are recorded as ADRs in DataKlubform.

To try Lambdas in the club's AWS sandbox, ask Kubi for a login and follow
[infra/aws-team-access](infra/aws-team-access/README.md).
