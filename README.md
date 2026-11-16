# Umbra

[![CI](https://github.com/Elegazia/umbra/actions/workflows/ci.yml/badge.svg)](https://github.com/Elegazia/umbra/actions/workflows/ci.yml)

A multi-asset market-data layer: records trades and order books from live feeds and bulk history, validates them, stores them as partitioned Parquet queried with DuckDB, and measures latency at every stage.

> **Status:** early development.

## Development

Requires [uv](https://docs.astral.sh/uv/).

```bash
git clone https://github.com/Elegazia/umbra.git
cd umbra
uv sync
uv run pre-commit install
```

Run the same checks as CI:

```bash
uv run ruff check
uv run ruff format --check
uv run pytest
```

## Design decisions

Recorded as [architecture decision records](docs/adr/README.md).

- [ADR 0001: Brokers, venues and data sources](docs/adr/0001-brokers-venues-and-data-sources.md)
- [ADR 0003: Run ruff in pre-commit through uv](docs/adr/0003-pre-commit-ruff-via-uv.md)
