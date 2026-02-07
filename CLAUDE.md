# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Unofficial Python wrapper for the Binance Exchange REST API v3. Provides both synchronous and asynchronous clients, WebSocket streaming, and depth cache management. Package name: `python_binance`, import as `binance`.

## Common Commands

### Install dependencies
```bash
pip install -r requirements.txt
pip install -r test-requirements.txt
```

### Run full test suite (via tox)
```bash
tox -e py
```

### Run tests directly with pytest
```bash
pytest -n 5 -v tests/ --timeout=90 --reruns 3 --reruns-delay 30
```

### Run a single test file
```bash
pytest tests/test_client.py -v --timeout=90
```

### Run a single test
```bash
pytest tests/test_client.py::test_function_name -v --timeout=90
```

### Lint
```bash
ruff check --target-version=py39 .
```

### Format
```bash
ruff format .
```

### Type check
```bash
pyright
```

### Pre-commit hooks
```bash
pre-commit run --all-files
```

## Test Environment

Tests run against Binance testnet by default. Environment variables needed (already set in CI):
- `TEST_TESTNET=true`
- `TEST_API_KEY`, `TEST_API_SECRET` — spot testnet keys
- `TEST_FUTURES_API_KEY`, `TEST_FUTURES_API_SECRET` — futures testnet keys
- `PROXY` — optional HTTP proxy

Test markers: `@pytest.mark.spot`, `@pytest.mark.futures`, `@pytest.mark.margin`, `@pytest.mark.portfolio`, `@pytest.mark.gift_card`

Tests use `requests-mock` for sync and `aioresponses` for async HTTP mocking. Tests are run with 5 parallel workers (`-n 5`) and flaky tests get 3 retries with 30-second delays.

## Architecture

### Client Hierarchy

`BaseClient` (base_client.py) — shared logic: API URLs, authentication, signature generation (HMAC/RSA/EDDSA), request parameter preparation.

`Client` (client.py, ~18K lines) — synchronous client using `requests.Session`. Created via `Client(api_key, api_secret)`.

`AsyncClient` (async_client.py, ~6K lines) — async client using `aiohttp.ClientSession`. Created via `await AsyncClient.create(api_key, api_secret)` (factory classmethod, not direct constructor).

Both clients expose the same API methods covering: spot trading, margin, USD-M futures, COIN-M futures, vanilla options, portfolio margin, and WebSocket API endpoints (prefixed `ws_`).

### API Endpoint Organization within Client

Methods are organized by API type with corresponding internal request methods:
- Spot: `_request_api()` → `https://api.binance.com/api`
- Margin/SAPI: `_request_margin_api()` → `https://api.binance.com/sapi`
- USD-M Futures: `_request_futures_api()` → `https://fapi.binance.com/fapi`
- COIN-M Futures: `_request_futures_coin_api()` → `https://dapi.binance.com/dapi`
- Options: `_request_options_api()` → `https://eapi.binance.com/eapi`
- Portfolio Margin: `_request_papi_api()` → `https://papi.binance.com/papi`

### WebSocket Layer (binance/ws/)

`BinanceSocketManager` (streams.py) — async WebSocket manager. Manages connections in `_conns` dict, supports multiplexing multiple streams.

`ReconnectingWebsocket` — handles auto-reconnect with exponential backoff (max 5 reconnects, max 60s wait). State machine: INITIALISING → STREAMING → RECONNECTING.

`KeepAliveWebsocket` — extends ReconnectingWebsocket with listen key refresh for user data streams.

`ThreadedWebsocketManager` — sync wrapper that runs BinanceSocketManager in a background thread with callback-based message delivery.

`DepthCacheManager` / `ThreadedDepthCacheManager` — maintains local order book snapshots updated via WebSocket. Variants exist for futures and options.

### Key Design Decisions

- AsyncClient uses a factory classmethod (`create()`) instead of `__init__` for proper async initialization (ping + timestamp offset calculation).
- Signatures support HMAC (default), RSA, and EDDSA via `private_key` parameter. Key can be a file path or string.
- Testnet/demo modes are toggleable via constructor flags, which swap API URLs.
- TLD parameter (`tld="us"`, `tld="jp"`) supports regional Binance domains.
- `verbose=True` enables debug logging of full request/response details.
