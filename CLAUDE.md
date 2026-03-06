# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Home Assistant custom integration (HACS) that monitors IMAP email accounts for package tracking and delivery notifications. Supports USPS, Amazon, UPS, FedEx, DHL, PostNL, DHL NL, DPD NL, bol.com, Amazon.nl, Walmart, and generic shippers. Processes emails locally without external services.

**Domain:** `mail_and_packages`
**Min HA version:** 2025.5.0
**Python targets:** 3.13, 3.14
**Config version:** 14 (with migration support in `__init__.py`)

## Commands

### Testing
```bash
pytest                                          # Run all tests with coverage
pytest tests/test_helpers.py                    # Run a specific test file
pytest tests/test_helpers.py::test_func_name    # Run a single test
pytest -x                                       # Stop on first failure
tox -e py313                                    # Run tests via tox
```

Tests use `pytest-asyncio` with `asyncio_mode = "auto"` (no manual `@pytest.mark.asyncio` needed). Timeout is 30 seconds per test.

### Linting & Formatting
```bash
ruff check .                # Lint
ruff check --fix .          # Lint with auto-fix
ruff format .               # Format
ruff format --check .       # Check formatting
tox -e lint                 # Lint check via tox
tox -e format               # Auto-fix + format via tox
tox -e mypy                 # Type checking
```

### Pre-commit
```bash
pre-commit run --all-files  # Run all hooks (ruff lint/format, codespell, yaml check, trailing whitespace)
```

## Architecture

### Core Integration Files (`custom_components/mail_and_packages/`)

- **`__init__.py`** — Entry point. `async_setup_entry()` creates a `MailDataUpdateCoordinator` that calls `process_emails()` on a configurable interval. Contains config migration logic (versions 1-14).
- **`helpers.py`** (~101KB) — Core business logic. IMAP login/search/fetch, email parsing per carrier, Amazon image downloading, USPS GIF generation, tracking number extraction. This is where most feature work happens.
- **`const.py`** (~51KB) — All constants: config keys, defaults, sensor descriptions, carrier-specific email patterns/regex, Amazon domain lists, multi-language search strings.
- **`config_flow.py`** — Multi-step UI configuration: IMAP credentials, carrier selection, image storage options, Amazon forwarding.
- **`camera.py`** — Camera entities for delivery images (USPS mail, Amazon, UPS, FedEx, Walmart, Generic). Service: `mail_and_packages.update_image`.
- **`sensor.py`** — `PackagesSensor` (counts for in-transit/delivered/exceptions) and `ImagePathSensors` (USPS mail image paths/URLs).
- **`binary_sensor.py`** — Update detection sensors (image changed vs "no mail" placeholder).
- **`entity.py`** — Custom entity description dataclass.
- **`diagnostics.py`** — Redacts sensitive data for diagnostic exports.

### Data Flow

1. `MailDataUpdateCoordinator._async_update_data()` triggers on interval
2. Calls `helpers.process_emails()` which logs into IMAP and processes emails per carrier
3. Returns dict of sensor values (counts, tracking numbers, image paths)
4. Coordinator updates binary sensors by comparing image hashes to detect deliveries
5. Sensor/camera entities read from coordinator data

### Test Structure (`tests/`)

- **`conftest.py`** — Extensive fixtures mocking IMAP connections, email fetch/search, and HA components
- **`const.py`** — Test constants, fake config entries, mock data
- **`test_helpers.py`** (6K+ lines) — Email parsing, image generation, carrier-specific logic
- **`test_config_flow.py`** (6K+ lines) — Configuration UI flow testing
- **`test_camera.py`** — Camera entity tests
- **`test_emails/`** — Sample email files used by tests

## Key Conventions

- Branch from `dev` for contributions; PRs target `dev`
- Ruff is the sole linter/formatter (replaced black/isort/flake8). Config in `pyproject.toml`
- Line length: 88 characters
- Codespell skips `tests/test_emails/` and `translations/`
- Adding a new carrier requires patterns in `const.py`, processing logic in `helpers.py`, and corresponding test coverage
