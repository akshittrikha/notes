Filled in incrementally as we talk through the project. Each package gets its own section once discussed — not pre-written.

## Runtime (`requirements.txt`)
### fastapi

### uvicorn

### sqlalchemy

### psycopg2-binary

### redis

### kafka-python

### pydantic

### prometheus-fastapi-instrumentator

## Dev / test (`requirements-dev.txt`)
### pytest

### httpx

### ruff

## Standard library (no pip install — ships with Python)
### argparse
Python's built-in CLI argument parser — declare flags (name, type, default, help text), it handles reading `sys.argv`, type validation, and `--help` generation.

**Where**: `scripts/simulate_device.py:7` (import), `:34-43` (parser setup) — defines `--devices`, `--interval`, `--anomaly-rate`, `--count` for the device simulator CLI.

**Why this and not Click/Typer**: only four flags, no subcommands — stdlib is the right amount of tooling here. Pulling in a third-party CLI framework for something this small would be an unjustified dependency. Not in `requirements.txt` at all since it ships with Python itself.
