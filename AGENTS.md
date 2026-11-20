# AGENTS.md — `aiopvpc-ng`

## 1. Project Identity & Mission

**`aiopvpc-ng`** is the independent, next-generation successor to [`azogue/aiopvpc`](https://github.com/azogue/aiopvpc).

- **Why this project exists:** The upstream `azogue/aiopvpc` repository has been unmaintained for over 2 years (last release `v4.3.1` on 2024-03-25), leaving critical maintenance gaps (such as hardcoded national holiday tables expiring after 2025, deprecated Python stdlib/async patterns, stale CI/packaging, and unaddressed ESIOS / Spanish market evolutions).
- **Mission:** Provide a modern, actively maintained, reliable asynchronous Python library for fetching Spanish electricity prices (**PVPC 2.0TD**, surplus/injection, MAG, OMIE, market adjustment, and composed indexed tariffs) from Red Eléctrica de España's official ESIOS API (`api.esios.ree.es`).
- **Primary consumers:** Home Assistant integrations (such as `pvpc_hourly_pricing` and custom HACS forks) as well as standalone Python energy automation and analytics tools.
- **Repository & License:** Independently hosted at `https://github.com/marcomsousa/aiopvpc-ng` under the **MIT License**, preserving full Git history and copyright attribution to the original author (Eugenio Panadero / `azogue`).

---

## 2. Next-Gen (`ng`) Goals & Modernization Priorities

When planning features, refactors, or fixes in this repository, prioritize these core objectives:

1. **Eliminate Time-Bomb & Maintenance Debt**
   - Replace the hardcoded `_NATIONAL_EXTRA_HOLIDAYS_FOR_P3_PERIOD` dictionary in `aiopvpc/pvpc_tariff.py` (which only covers 2021–2025 and raises `KeyError` from 2026 onward) with deterministic/algorithmic calculation of Spanish non-substitutable national holidays (including Good Friday) so tariff period detection (`P1`/`P2`/`P3`) never expires.
   - Clean up legacy `TODO` items (e.g., obsolete 2016–2019 browser `User-Agent` rotation in `pvpc_data.py` and geozone handling in `parser.py`).

2. **Modernize Python Runtime, Tooling & CI**
   - Remove deprecated APIs (`datetime.utcnow()` → timezone-aware `datetime.now(UTC)`; external `async_timeout` → stdlib `asyncio.timeout` where minimum Python version permits).
   - Modernize Python version support, packaging/lockfile metadata (`pyproject.toml`), pre-commit hooks, and GitHub Actions workflows (updating outdated actions and rebranding package metadata/URLs to `aiopvpc-ng` when ready for release).

3. **Keep Up With ESIOS API & Spanish Electricity Market Changes**
   - Maintain compatibility with `api.esios.ree.es` public archives and token-authenticated `/indicators/{id}` endpoints.
   - Prepare and adapt cleanly for regulatory/market evolutions (e.g., indicator changes, geozone refinements, or quarter-hourly / 15-minute pricing structures).

4. **Preserve Home Assistant Integration Ergonomics**
   - Keep the public interface (`PVPCData`, `EsiosApiData`, `BadApiTokenAuthError`, `TARIFFS`, `DEFAULT_POWER_KW`, `get_enabled_sensor_keys`) clean, typed, and predictable.
   - Follow Semantic Versioning (`SemVer`): document any intentional breaking changes clearly in `CHANGELOG.md`.

5. **High Reliability & Test Rigor**
   - Maintain **>= 95% branch test coverage** (`pyproject.toml` enforces `fail_under = 95`).
   - Keep the default test suite 100% deterministic and offline using recorded JSON fixtures in `tests/api_examples/`.

---

## 3. Repository Architecture

```text
aiopvpc-ng/
├── aiopvpc/
│   ├── __init__.py       # Public exports (PVPCData, EsiosApiData, BadApiTokenAuthError, etc.)
│   ├── const.py          # ESIOS URLs, indicator IDs (1001, 1739, 1900, 10211, 2108), tariffs, geozones, dataclasses
│   ├── pvpc_data.py      # Async HTTP client (PVPCData), request throttling, auth error handling, state processing
│   ├── parser.py         # Parsers for public archive JSON and token-authenticated ESIOS indicator JSON
│   ├── prices.py         # Today/tomorrow splitting, price ranking/stats attributes, composed sensors (INDEXED)
│   ├── pvpc_tariff.py    # 2.0TD tariff period logic (P1/P2/P3), Peninsula vs Ceuta/Melilla, P3 national holidays
│   ├── ha_helpers.py     # Home Assistant unique_id mapping and enabled-sensor resolution
│   └── utils.py          # UTC datetime normalization helpers
├── tests/
│   ├── api_examples/     # Recorded ESIOS JSON responses (including DST spring/fall days)
│   ├── conftest.py       # Pytest fixtures and mock HTTP helpers
│   ├── test_pvpc.py      # End-to-end PVPCData download & caching tests
│   ├── test_pvpc_parsing.py
│   ├── test_pvpc_zones.py
│   ├── test_tariff_attributes.py
│   ├── test_ha_helpers.py
│   └── test_real_api_calls.py  # Marked `real_api_call` (deselected by default)
├── .github/workflows/
│   ├── main.yml          # Push & PR CI test, coverage, and package build workflow
│   └── publish.yml       # Release-triggered PyPI publish workflow
├── pyproject.toml        # PEP 621 package metadata, uv dependency groups, ruff, pytest, coverage
├── uv.lock               # Reproducible uv lockfile
└── CHANGELOG.md          # Release history
```

---

## 4. Domain Invariants (Must Not Break)

Agents modifying parsing, scheduling, or tariff logic must uphold these invariants:

- **Timezone & Local-Hour Semantics:** Prices are stored in `dict[datetime, float]` keyed by tz-aware UTC datetimes, but Spanish electricity prices apply by **local wall-clock hour** (e.g., 10:00 in Canary Islands applies the 10:00 price, shifted relative to `Europe/Madrid`). Always test both `Europe/Madrid` and `Atlantic/Canary`.
- **Daylight Saving Time (DST):** Spring-forward (23-hour) and fall-back (25-hour, duplicated hour suffixed with `_d` in attributes) transitions must remain fully supported and tested.
- **Tariff Zones:** Support both `2.0TD` (`PCB`, Peninsula/Balearics/Canaries) and `2.0TD (Ceuta/Melilla)` (`CYM`, shifted `P2` hours: 8–10, 15–18, 23).
- **ESIOS Rate-Limit Respect:** `PVPCData._update_prices_series` intentionally avoids redundant API calls once today's (or today + tomorrow's after 20:00 `Europe/Madrid`) prices are cached. Do not introduce polling behavior that increases requests per day.
- **Auth Error Contract:** When using `data_source="esios"` with an API token, HTTP `401` and `403` responses must raise `BadApiTokenAuthError` so Home Assistant can trigger its re-authentication flow.
- **Units:** ESIOS API returns prices in `€/MWh`; `aiopvpc` converts and exposes all prices in `€/kWh` rounded to `PRICE_PRECISION` (5 decimal places).

---

## 5. Development, Testing & Release Guidelines

### Code Quality & Tests
- **Environment & Packaging:** Managed with `uv` (`uv sync --locked`, `uv run pytest`, `uv build`). Uses PEP 621 metadata with `hatchling` as the build backend.
- **Linting & Formatting:** Configured via `ruff` in `pyproject.toml` (line length `88`, strict rule selection including `ASYNC`, `B`, `C4`, `I`, `PTH`, `RUF`, `SIM`, `TCH`). Run `uv run ruff check --no-fix` and `uv run ruff format --check` before committing.
- **Offline Test Suite (Default):** `uv run pytest` automatically runs with coverage (`fail_under = 95`) and `-m 'not real_api_call'`. All PRs and commits must pass offline tests with >= 95% branch coverage.
- **Live ESIOS API Tests:** `tests/test_real_api_calls.py` requires a personal ESIOS API token via environment variable / local `.env` and explicit `uv run pytest -m real_api_call`. **Never commit API tokens or `.env` files.**

### CI/CD & Packaging
- `.github/workflows/main.yml` runs `uv sync --locked`, tests, coverage checks, and `uv build` across Python 3.11–3.13 on pushes to `master` and pull requests.
- `.github/workflows/publish.yml` publishes `aiopvpc-ng` to PyPI via `uv publish --trusted-publishing always` only on GitHub Release publication (`release: [published]`) or manual `workflow_dispatch` using OIDC Trusted Publishing (`environment: pypi`).
- Distribution package name is `aiopvpc-ng`, while the Python import package remains `aiopvpc` (`[tool.hatch.build.targets.wheel] packages = ["aiopvpc"]`) for drop-in compatibility with Home Assistant.

### Git & Contribution Conventions
- Commit as `Marco Sousa <marcomsousa@gmail.com>` under GitHub account `marcomsousa`.
- Use clear, atomic Conventional Commits (`feat(...)`, `fix(...)`, `refactor(...)`, `chore(...)`, `docs(...)`, `test(...)`).
- Do not mix unrelated refactors with behavioral fixes; update `CHANGELOG.md` and tests alongside functional changes.
