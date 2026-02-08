# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Unofficial Python wrapper for the Binance Exchange REST API v3. Provides both synchronous and asynchronous clients, WebSocket streaming, and depth cache management. Package name: `python_binance`, import as `binance`. Targets Python 3.8–3.12 (CI tests all versions; ruff targets py39).

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

### Pre-commit hooks (ruff + pyright + trailing whitespace + YAML check)
```bash
pre-commit run --all-files
```

## Test Environment

Tests run against Binance testnet by default. Environment variables needed (already set in CI):
- `TEST_TESTNET=true`
- `TEST_API_KEY`, `TEST_API_SECRET` — spot testnet keys
- `TEST_FUTURES_API_KEY`, `TEST_FUTURES_API_SECRET` — futures testnet keys
- `PROXY` — optional HTTP proxy

Test markers: `@pytest.mark.spot`, `@pytest.mark.futures`, `@pytest.mark.margin`, `@pytest.mark.portfolio`, `@pytest.mark.gift_card`. Margin, portfolio, and gift_card tests are off by default (require `--run-margin` etc.).

### Test Fixtures (conftest.py)

- `client` — sync `Client` configured for spot testnet
- `futuresClient` — sync `Client` configured for futures demo
- `liveClient` — sync `Client` against production (use carefully)
- `clientAsync` / `futuresClientAsync` / `liveClientAsync` — async equivalents (auto-close on teardown)
- `manager` — `ThreadedWebsocketManager` for WebSocket tests

### Test File Convention

Tests come in sync/async pairs mirroring the same API surface:
- `test_client.py` ↔ `test_async_client.py` (spot)
- `test_client_futures.py` ↔ `test_async_client_futures.py`
- `test_client_margin.py` ↔ `test_async_client_margin.py` (etc.)

Async test files use `pytestmark = [pytest.mark.asyncio]` at module level. Sync tests use `requests-mock`; async tests use `aioresponses`.

## Architecture

### Client Hierarchy

`BaseClient` (base_client.py) — shared logic: API URLs, authentication, signature generation (HMAC/RSA/EDDSA), request parameter preparation.

`Client` (client.py, ~18K lines) — synchronous client using `requests.Session`. Created via `Client(api_key, api_secret)`. This is the canonical source of all API method docstrings.

`AsyncClient` (async_client.py, ~6K lines) — async client using `aiohttp.ClientSession`. Created via `await AsyncClient.create(api_key, api_secret)` (factory classmethod, not direct constructor). Methods delegate docstrings from the sync Client: `method.__doc__ = Client.method.__doc__`.

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

`ReconnectingWebsocket` (reconnecting_websocket.py) — handles auto-reconnect with exponential backoff (max 5 reconnects, max 60s wait). State machine: INITIALISING → STREAMING → RECONNECTING → EXITING.

`KeepAliveWebsocket` (keepalive_websocket.py) — extends ReconnectingWebsocket with listen key refresh for user data streams.

`WebsocketAPI` (websocket_api.py) — extends ReconnectingWebsocket for request-response style WebSocket API calls. Routes responses by request ID and supports subscription-based event routing.

`BinanceSocketManager` (streams.py) — async WebSocket manager. Manages connections in `_conns` dict, supports multiplexing multiple streams.

`ThreadedWebsocketManager` (threaded_stream.py → streams.py) — sync wrapper that runs BinanceSocketManager in a background thread with callback-based message delivery. Inherits from `ThreadedApiManager` which manages the asyncio event loop in a separate thread.

`DepthCacheManager` / `ThreadedDepthCacheManager` (depthcache.py) — maintains local order book snapshots updated via WebSocket. Variants exist for futures and options.

### Key Design Decisions

- AsyncClient uses a factory classmethod (`create()`) instead of `__init__` for proper async initialization (ping + timestamp offset calculation).
- Signatures support HMAC (default), RSA, and EDDSA via `private_key` parameter. Key can be a file path or string.
- Testnet/demo modes are toggleable via constructor flags, which swap API URLs.
- TLD parameter (`tld="us"`, `tld="jp"`) supports regional Binance domains.
- `verbose=True` enables debug logging of full request/response details.
- `code-generator.py` scrapes Binance API docs to detect missing endpoint coverage — useful when adding new endpoints.

### Ruff Configuration

Preview mode is enabled. Intentionally suppressed rules (in `pyproject.toml`): F722, F841, F821, E402, E501, E902, E713, E741, E714, E275, E721, E266, E261. Do not fix these categories as they are project-wide exceptions.
