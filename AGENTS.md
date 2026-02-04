# Labs Module - Agent Notes

This document captures technical decisions, problems encountered, and solutions implemented during Labs module development. It serves as institutional knowledge for future development.

## Tooling: uv (Required)

**Use `uv` for all dependency and script execution in Labs.** Do not use `pip`, `pip install`, or `python3` / `.venv/bin/python` for Labs-specific work.

| Task | Command | Working directory |
|------|---------|-------------------|
| Add dependency | `uv add <pkg>` or `uv add --optional data <pkg>` | `labs/` |
| Install/sync deps from lockfile | `uv sync` | `labs/` |
| Run a script | `uv run python cli/download_yahoo_data.py` | `labs/` |
| Run tests | `uv run pytest tests/ -v` | `labs/` |
| Run with optional group (data) | `uv sync --extra data` then `uv run python cli/...` | `labs/` |

**From repo root** (e.g. Makefile or one-off):
- Sync labs deps: `uv sync --project labs` or `cd labs && uv sync`
- Run labs script: `uv run --project labs python src/labs/cli/download_yahoo_data.py` (path relative to `labs/`)
- Run labs tests: `uv run --project labs pytest tests/ -v`

**Always ensure you are in the correct folder:**
- Changes to `labs/pyproject.toml` or running labs cli/tests from inside the module: **`cd labs`** first, then `uv add` / `uv sync` / `uv run ...`.
- Root Makefile targets (e.g. `make labs-data-pull`) run from **repo root** and use `cd labs &&` before commands.

## Certificate Handling for Local Development

### Problem Encountered (2026-02-03)

When implementing Yahoo Finance data download (`src/labs/cli/download_yahoo_data.py`), we encountered SSL certificate verification errors:

```
Failed to perform, curl: (60) SSL certificate problem: unable to get local issuer certificate
```

This affected:
- Yahoo Finance API (`yfinance` library via `curl-cffi`)
- FRED API (`pandas-datareader`)
- Any HTTPS requests from Labs cli

**Impact**: Data download completely failed; no CSV files could be generated.

### Root Cause

The `yfinance` library uses `curl-cffi` (not Python's `requests`), which relies on system curl's CA bundle. On macOS development machines, especially behind corporate proxies (Zscaler), curl needs explicit CA certificate configuration.

### Repo's Certificate Pattern (Already Solved)

**Investigation showed the repo already handles this:**

1. **Makefile.local** (`create-test-user` target):
   ```bash
   export SSL_CERT_FILE="/etc/ssl/cert.pem"
   export REQUESTS_CA_BUNDLE="/etc/ssl/cert.pem"
   export CURL_CA_BUNDLE="/etc/ssl/cert.pem"
   ```

2. **README.md** section "Certificate errors (corporate proxy)":
   ```bash
   make local-podman-install-cert
   ```
   This installs corporate CA (e.g., Zscaler) into Podman machine at `/etc/pki/ca-trust/source/anchors/`.

3. **Consistent approach**: All local dev tools use the same CA bundle paths.

### Solution Implemented

Applied the repo's existing certificate pattern to Labs:

#### 1. Script-Level: `src/labs/cli/download_yahoo_data.py`

Added `_ensure_ssl_env()` function at import time:

```python
def _ensure_ssl_env() -> None:
    """Set CA bundle env vars for Python SSL, requests, and curl (yfinance/curl-cffi)."""
    if os.environ.get("SSL_CERT_FILE") and os.environ.get("CURL_CA_BUNDLE"):
        return
    try:
        import certifi
        path = certifi.where()
        if not os.environ.get("SSL_CERT_FILE"):
            os.environ["SSL_CERT_FILE"] = path
        if not os.environ.get("REQUESTS_CA_BUNDLE"):
            os.environ["REQUESTS_CA_BUNDLE"] = path
        if not os.environ.get("CURL_CA_BUNDLE"):
            os.environ["CURL_CA_BUNDLE"] = path
    except ImportError:
        # Fallback: macOS / Homebrew
        for candidate in ("/etc/ssl/cert.pem", "/etc/pki/tls/certs/ca-bundle.crt"):
            if os.path.isfile(candidate):
                os.environ.setdefault("SSL_CERT_FILE", candidate)
                os.environ.setdefault("REQUESTS_CA_BUNDLE", candidate)
                os.environ.setdefault("CURL_CA_BUNDLE", candidate)
                break
```

**Rationale**: Script works standalone (using certifi) OR via Makefile (explicit env vars).

#### 2. Makefile-Level: Root `Makefile`

Added Labs-specific targets mirroring the repo pattern:

```makefile
# Use same certificate env as Makefile.local create-test-user
# Prefer uv (run from repo root): uv run --project labs python cli/download_yahoo_data.py
LABS_SSL_CERT ?= $(shell test -f /etc/ssl/cert.pem && echo /etc/ssl/cert.pem || (cd labs && uv run python -c "import certifi; print(certifi.where())") 2>/dev/null || true)

labs-download-data: ## Download Yahoo Finance data for Labs (uses repo cert env)
    @if [ -z "$(LABS_SSL_CERT)" ] || [ ! -f "$(LABS_SSL_CERT)" ]; then \
        export SSL_CERT_FILE=$$(cd labs && uv run python -c "import certifi; print(certifi.where())"); \
        export REQUESTS_CA_BUNDLE=$$SSL_CERT_FILE; export CURL_CA_BUNDLE=$$SSL_CERT_FILE; \
        cd labs && uv run python cli/download_yahoo_data.py; \
    else \
        export SSL_CERT_FILE="$(LABS_SSL_CERT)"; export REQUESTS_CA_BUNDLE="$(LABS_SSL_CERT)"; export CURL_CA_BUNDLE="$(LABS_SSL_CERT)"; \
        cd labs && uv run python cli/download_yahoo_data.py; \
    fi
```

**Rationale**: Consistent with rest of repo (`make local`, `make create-test-user`).

#### 3. Dependencies: `labs/pyproject.toml`

Added `certifi>=2024.0.0` to `[project.optional-dependencies]` `data` group.

**Rationale**: Provides CA bundle when system cert isn't available.

#### 4. Documentation

Updated:
- `labs/README.md`: Certificates section pointing to repo convention
- `labs/docs/DATA_DOWNLOAD_STATUS.md`: Workarounds using repo pattern
- Help text in `Makefile`: Labs section

### Verification

**Before fix:**
```
0/63 price data files downloaded (all failed with SSL error)
```

**After fix (from repo root):**
```bash
$ make labs-data-pull
# Or from labs/: uv sync --extra data && uv run python src/labs/cli/download_yahoo_data.py
Using CA bundle: /etc/ssl/cert.pem
... Saved 1255 rows to labs/data/price/eqt/AAPL.csv
Downloaded 63/63 price data files
```

**Result**: All Yahoo Finance and FRED downloads succeed.

### Prevention for Future Development

**When adding any Labs script that makes HTTPS requests:**

1. **Import pattern**: Add `_ensure_ssl_env()` at top of script (copy from `download_yahoo_data.py`).

2. **Makefile pattern**: If creating a new Make target, export the three env vars:
   ```makefile
   export SSL_CERT_FILE="$(LABS_SSL_CERT)"
   export REQUESTS_CA_BUNDLE="$(LABS_SSL_CERT)"
   export CURL_CA_BUNDLE="$(LABS_SSL_CERT)"
   ```

3. **Testing**: Always test with `make labs-<command>` (not just direct script) to verify Makefile integration.

4. **Dependencies**: If script uses libraries that need SSL, add with **`uv add`** from **`labs/`**: `cd labs && uv add certifi` (or `uv add --optional data certifi`). Do not use `pip install`.

5. **Corporate proxy**: Document in README if script requires `make local-podman-install-cert` for corporate environments.

### Key Takeaway

**Don't reinvent certificate handling.** The repo already has a working pattern (`Makefile.local` + `/etc/ssl/cert.pem` + certifi fallback). When Labs encountered the same issue, applying the existing pattern solved it immediately. Future Labs development should follow this pattern for all HTTPS operations.

---

## Do Not

**curl-cffi / yfinance:**
- Do NOT import yfinance inside functions - import at module level after SSL setup
- Do NOT assume setting CURL_CA_BUNDLE at runtime works - must be set before curl loads
- Do NOT use logging.info() in `_ensure_ssl_env()` - use print() (logging not configured yet)

**SSL troubleshooting:**
- Do NOT skip testing with SSL-requiring endpoints (ticker.info, recommendations)
- Do NOT assume price data working means SSL is configured (price uses different endpoint)

---

## Data Sources and API Limits

### Yahoo Finance (via `yfinance`)

**What's available:**
- Historical OHLCV (Open, High, Low, Close, Volume): 5+ years daily
- Fundamentals (income, balance, cashflow): quarterly/annual
- No official API or rate limits documentation (unofficial web backend)

**Labs usage:**
- 63 symbols for price data
- 15 symbols for fundamentals
- Total ~100 requests per full download

**Rate limiting strategy:**
- `REQUEST_DELAY_SECONDS = 0.5` (configurable)
- Sequential downloads with `time.sleep()` between requests
- No parallelization to avoid appearing as bot traffic
- Skip already-downloaded files unless `--force`

**Practical limits (observed):**
- No published rate limits
- Conservative approach: 0.5–2 seconds between requests
- Full download ~50+ seconds minimum
- If encountering 429/403, increase delay and retry

**Optimization strategy:**
- Download in tiers (Tier 1 → Tier 2 → ... → Tier 6)
- Use `--symbols` flag for subset testing
- Make delay configurable via CLI override
- Cache results to avoid re-downloading

### FRED API (via `pandas-datareader`)

**What's available:**
- US macroeconomic data (CPI, GDP, etc.)
- Free tier: sufficient for Labs (~3 requests)

**Labs usage:**
- US macro only (CPI, GDP)
- Other countries: synthetic data generation

---

## Testing Strategy

### Labs Tests Use Fixtures, Not Real Data

**Important**: Labs unit tests (`labs/tests/`) use pytest fixtures with sample data, **not** downloaded CSV files.

- `tests/conftest.py` provides `sample_price_data`, `sample_mapping_yaml`, `temp_data_dir`, etc.
- Tests are fast and don't depend on external APIs
- Real data download is **optional** for development

**When to download real data:**
- Manual testing with Marimo notebooks
- Validating data quality (`make labs-data-pull` + `src/labs/cli/validate_data.py`)
- Integration testing (future)

**When NOT needed:**
- Running unit tests: from **repo root** `make labs-verify`, or from **`labs/`** `uv run pytest tests/ -v`
- TDD development cycle
- CI/CD pipelines

---

## Future Improvements

### Data Download

1. **Parallel downloads**: Implement limited concurrency (2-3 workers) with rate limiting
2. **Resume capability**: Track last-downloaded symbol and resume on failure
3. **Incremental updates**: Only fetch new data since last download (e.g., append last week)
4. **Retry logic**: Exponential backoff on 429/5xx errors
5. **CLI enhancements** (run from `labs/`):
   ```bash
   cd labs && uv run python src/labs/cli/download_yahoo_data.py --delay 1.0 --tier 1 --parallel 2
   ```

### Data Catalog

Consider generating a "data catalog" JSON/MD that inventories:
- Available symbols per asset class
- Date ranges covered
- Items available per namespace
- Last update timestamp

This becomes the "spec of free data" referenced by users.

### Monitoring

- Log download success/failure rates
- Track API response times
- Alert on persistent failures or throttling

---

## References

- **Repo cert handling**: `Makefile.local` (lines 463+, `create-test-user` target)
- **README cert docs**: Root `README.md` "Certificate errors (corporate proxy)"
- **Labs cert implementation**: `src/labs/services/downloader.py` (SSL setup)
- **CLI entry point**: `src/labs/cli/main.py` (typer app)
- **Data download service**: `src/labs/services/downloader.py` (all 212 files)
- **Database generator**: `src/labs/services/db_generator.py` (sim_prod.db)
- **Catalog generator**: `src/labs/services/catalog.py` (100% completeness scoring)
- **ADR**: `docs/architecture/ADRs/ADR-011-labs-module.md`

**Labs commands (always from correct folder):**
- In `labs/`: `uv add <pkg>`, `uv sync --extra data`, `uv run labs data --help`, `uv run labs data download`, `uv run labs data db`, `uv run labs data catalog`
- From repo root: `make labs-run`, `make labs-stop`, `make labs-status`, `make labs-cleanup`

**Makefile targets:**
- `labs-run`: Installs, downloads, generates DB, launches quickstart notebook
- `labs-stop`: Kills marimo process
- `labs-status`: Shows complete environment status (all 212 files, dependencies, notebook, tests)
- `labs-cleanup`: Removes all artifacts (data, venv, caches)

---

## curl-cffi / yfinance SSL Import Timing (CRITICAL)

### Problem
`src/labs/cli/yf_data_catalog_generator.py` encountered persistent SSL certificate errors despite having `_ensure_ssl_env()`:

```
Failed to perform, curl: (60) SSL certificate problem: unable to get local issuer certificate
```

Modules affected: `info`, `calendar`, `recommendations`, `major_holders`, `institutional_holders`.

### Root Cause: Import Timing

**curl-cffi reads `CURL_CA_BUNDLE` when the curl library is first loaded, not when requests are made.**

❌ **This DOES NOT work:**
```python
def _ensure_ssl_env():
    os.environ["CURL_CA_BUNDLE"] = certifi.where()

_ensure_ssl_env()  # Sets env vars
import pandas as pd

def probe_module():
    import yfinance as yf  # TOO LATE - curl already loaded
    ticker = yf.Ticker("AAPL")
    return ticker.info  # SSL ERROR
```

✅ **This WORKS:**
```python
def _ensure_ssl_env():
    os.environ.setdefault("CURL_CA_BUNDLE", certifi.where())
    os.environ.setdefault("SSL_CERT_FILE", certifi.where())
    os.environ.setdefault("REQUESTS_CA_BUNDLE", certifi.where())
    print(f"[SSL] Using CA bundle: {certifi.where()}")

_ensure_ssl_env()  # Sets env vars FIRST
import yfinance as yf  # Import at module level AFTER SSL setup

def probe_module():
    ticker = yf.Ticker("AAPL")  # Uses module-level import
    return ticker.info  # SSL OK
```

### Fix Pattern for yfinance Scripts

**Template:**
```python
#!/usr/bin/env python3
import os
from pathlib import Path

# 1. Define SSL setup FIRST (before any imports)
def _ensure_ssl_env() -> None:
    if os.environ.get("SSL_CERT_FILE") and os.environ.get("CURL_CA_BUNDLE"):
        return
    try:
        import certifi
        path = certifi.where()
        os.environ.setdefault("SSL_CERT_FILE", path)
        os.environ.setdefault("REQUESTS_CA_BUNDLE", path)
        os.environ.setdefault("CURL_CA_BUNDLE", path)
        print(f"[SSL] Using CA bundle: {path}")  # Use print, logging not ready yet
    except ImportError:
        for candidate in ("/etc/ssl/cert.pem", "/etc/pki/tls/certs/ca-bundle.crt"):
            if os.path.isfile(candidate):
                os.environ.setdefault("SSL_CERT_FILE", candidate)
                os.environ.setdefault("REQUESTS_CA_BUNDLE", candidate)
                os.environ.setdefault("CURL_CA_BUNDLE", candidate)
                print(f"[SSL] Using CA bundle: {candidate}")
                break

# 2. Call SSL setup BEFORE any imports that use curl
_ensure_ssl_env()

# 3. Import yfinance at MODULE LEVEL (not inside functions)
import yfinance as yf
import pandas as pd
import logging

# 4. Now configure logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s - %(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

# 5. Use yf in functions (it's already imported at module level)
def fetch_data(symbol: str):
    ticker = yf.Ticker(symbol)  # Uses module-level import
    return ticker.info  # SSL works
```

### Prevention

**For any new Labs script using yfinance:**

1. **Copy SSL setup from `yf_data_catalog_generator.py`** (reference implementation)

2. **NEVER import yfinance inside functions** - always at module level after `_ensure_ssl_env()`

3. **Use print() for SSL logging** - logging isn't configured yet when SSL setup runs

4. **Test with SSL-requiring endpoints**:
   ```bash
   cd labs && uv run python src/labs/cli/your_script.py
   # Should show: [SSL] Using CA bundle: /path/to/cacert.pem
   # Then fetch ticker.info successfully
   ```

### Verification

```bash
cd labs && uv run python src/labs/cli/yf_data_catalog_generator.py --symbols "AAPL" --modules "info"
```

Expected output:
```
[SSL] Using CA bundle: /opt/homebrew/.../certifi/cacert.pem
... (successful data fetch, no SSL errors)
```

---

## Do

- Use `uv` for all Labs dependency management (not pip)
- Call `_ensure_ssl_env()` before any yfinance/HTTPS imports
- Use `print()` for SSL logging (logger not configured yet at import time)
- Convert DataFrame column names and dict keys to strings before JSON serialization
- When debugging JSON serialization errors, check for non-string keys (Timestamp, int, custom objects)
- Use `str(key)` and `str(value)` defensively when building JSON-bound dicts from DataFrames
- Update tests when mapping files transition from "to be created" to "committed in repo"
- Run tests after creating new files to verify tests still pass (file existence may change test behavior)

## Do Not

- Do not use `pip install` in Labs (use `uv add`)
- Do not assume logging works at module import time
- Do not skip testing SSL-requiring endpoints (info, recommendations)
- Do not use pandas DataFrame column names directly as JSON keys without converting to strings first
- Do not assume all dict keys are JSON-serializable - validate or convert to str before json.dumps/json.dump
- Do not write overly strict test assertions for error messages - use substring matching with "in" operator
- Do not leave tests expecting FileNotFoundError when mapping files are now committed to repo
- Do not skip mapping file validation tests once files exist (remove `@pytest.mark.skipif`)

## Lessons Learned

- curl-cffi requires CURL_CA_BUNDLE set BEFORE library loads
- Use `setdefault()` not conditional checks for env vars
- Always log which CA bundle is being used for debugging
- Cursor skills that modify files need explicit constraints on which files
- Use session context to identify relevant files, don't search entire repo
- When generating JSON from pandas DataFrames, explicitly convert all column names and values to strings to avoid TypeError with Timestamp objects
- Timestamp objects as dict keys will cause `json.dumps` to fail - convert to str before serialization
- Test assertions should use substring matching for error messages, not exact string equality
- When testing catalog generation modes, ensure test parameters match expected mode (e.g., modules_filter=None for "full" mode)
- When mapping files are committed to repo, update tests that expected FileNotFoundError to expect success
- Tests for mapping file validation should run always (not skipped) once files exist in repo
- System Python on macOS is "externally managed" - always use uv, never pip directly

---

## YF Data Catalog Generation (2026-02-03)

### Overview

The Labs module includes a comprehensive Yahoo Finance data catalog generator (`src/labs/cli/yf_data_catalog_generator.py`) that documents API availability, schemas, and field mappings for simulation data.

### Catalog Purpose

The catalog serves as metadata about what data is available from Yahoo Finance:

1. **API Coverage**: Which yfinance modules work for which asset classes
2. **Schema Documentation**: Field mappings from Yahoo → Production → DSL
3. **Fundamental Items**: Available financial items per module
4. **Availability Matrix**: Cross-reference of modules × asset types

### SIMULATION_DATA_SPEC Compliance

The catalog addresses **Tiers 1-5** from `labs/docs/SIMULATION_DATA_SPEC.md`:

- **Tier 1**: US Equities (43 symbols)
- **Tier 2**: International Equities (10 symbols: UK, DE, JP, HK)
- **Tier 3**: ETFs & Indices (5 symbols)
- **Tier 4**: Crypto, FX, Fixed Income (8 symbols)
- **Tier 5**: Fundamentals (15 stocks)
- **Tier 6**: Macro data (FRED API, not yfinance) - documented but not probed

### Probe Panel (14 representative symbols)

```python
PROBE_SYMBOLS = {
    "equity_us": ["AAPL", "MSFT", "JPM"],      # US majors
    "equity_uk": ["BP.L"],                      # UK
    "equity_de": ["SAP.DE"],                    # Germany
    "equity_jp": ["6758.T"],                    # Japan (Sony)
    "equity_hk": ["0700.HK"],                   # Hong Kong (Tencent)
    "etf": ["SPY", "QQQ"],                      # ETFs
    "index": ["^GSPC", "^DJI"],                 # US indices
    "crypto": ["BTC-USD", "ETH-USD"],           # Crypto
    "fx": ["EURUSD=X", "GBPUSD=X"],             # FX pairs
    "fix": ["^TNX", "^TYX"],                    # Treasury yields (NEW)
}
```

Key: **FIX asset class** (treasury yields) added to cover SIMULATION_DATA_SPEC Tier 4 requirements.

### Discovery Mode (New in v2.1)

The generator can programmatically discover fresh symbols using the `--discover` flag:
1. **Sectors**: Technology, Financial Services, Healthcare, Energy, etc.
2. **Screeners**: Day Gainers, Most Actives, Undervalued Growth, etc.
3. **Integration**: Discovered symbols are added to the probe list, ensuring the catalog reflects currently active market participants.

### Usage

**From repo root** (recommended - uses SSL cert env):

```bash
# Download data (simplified - catalog/db generation removed from targets)
make labs-data-pull
```

**Direct script usage** (from `labs/`):

```bash
cd labs

# Download data
uv run python src/labs/cli/download_yahoo_data.py

# Generate catalog
uv run python src/labs/cli/yf_data_catalog_generator.py

# Quick mode (4 modules, 6 symbols, ~24 requests, ~12s)
uv run python src/labs/cli/yf_data_catalog_generator.py --quick

# Validate existing
uv run python src/labs/cli/yf_data_catalog_generator.py --validate

# Ensure exists (generate if missing)
uv run python src/labs/cli/yf_data_catalog_generator.py --ensure

# Custom symbols/modules
uv run python src/labs/cli/yf_data_catalog_generator.py --symbols "AAPL,MSFT" --modules "history,info"

# Discovery mode (finds new symbols via Sector/Screener APIs)
uv run python src/labs/cli/yf_data_catalog_generator.py --quick --discover
```

### Catalog Structure (v2.1)

```json
{
  "version": "2.1",
  "generated_at": "2026-02-03T...",
  "generation_mode": "discovery",
  "duration_seconds": 120.5,
  "completeness_score": 0.87,
  
  "spec_compliance": {
    "simulation_data_spec": "labs/docs/SIMULATION_DATA_SPEC.md",
    "tiers_covered": [1, 2, 3, 4, 5],
    "tier_6_note": "Macro data from FRED API, not yfinance"
  },
  
  "schema_mapping": {
    "history": {
      "Date": {"production": "PRICE_DATE", "dsl_item": null},
      "Open": {"production": "PRICE_OPEN", "dsl_item": "OpenPrice"},
      "Close": {"production": "PRICE", "dsl_item": "ClosePrice"},
      "Volume": {"production": "VOLUME", "dsl_item": "Volume"}
    }
  },
  
  "fundamental_items": {
    "income_stmt": ["Revenue", "NetIncome", "EPS", "OperatingIncome"],
    "balance_sheet": ["TotalAssets", "ShareholdersEquity", "TotalDebt"],
    "cash_flow": ["FreeCashFlow", "OperatingCashFlow", "CapitalExpenditures"]
  },
  
  "availability_matrix": {
    "history": {"equity_us": true, "equity_uk": true, "fix": true, ...},
    "info": {"equity_us": true, "fix": false, ...},
    "dividends": {"equity_us": true, "crypto": false, ...}
  },
  
  "modules": [
    {
      "module": "history",
      "description": "Historical OHLCV price data",
      "symbols_probed": ["AAPL", "MSFT", "BP.L", ...],
      "availability": {"equity_us": true, "equity_uk": true, ...},
      "schema": {"Open": {"type": "float64", "nullable": false}, ...},
      "sample_row": {...}
    }
  ]
}
```

### Outputs

1. **`labs/data/catalog/yf_catalog.json`** - Structured catalog (metadata)
2. **`labs/data/catalog/yf_catalog_samples.jsonl`** - Raw sample data (JSONL)

### When to Regenerate

Regenerate the catalog when:

1. **Adding new asset classes** to `yahoo_to_dsl.json`
2. **yfinance API changes** (e.g., new modules, deprecated endpoints)
3. **Catalog validation fails** (`make labs-catalog-validate`)
4. **Testing data availability** before downloading full datasets

### Integration with Data Download

The catalog is **metadata** about what's available. To actually download data:

```bash
make labs-data-pull  # Downloads 63 symbols, 5 years history
```

The catalog answers "what can I download?" The download script gets the actual data.

### Testing

Unit tests cover:

- Schema inference
- Validation logic
- Ensure catalog generation
- Expanded probe panel coverage
- Catalog structure v2.0

Run tests:

```bash
make labs-test
# or from labs/: uv run pytest tests/test_yf_data_catalog_generator.py -v
```

### SSL Certificate Handling

The catalog generator uses the same SSL pattern as other Labs cli. The `make labs-data-pull` target automatically configures SSL certificates (uses certifi or `/etc/ssl/cert.pem`).

If running directly, SSL env vars are set at script startup via `_ensure_ssl_env()`.

### Performance

| Mode | Modules | Symbols | Requests | Duration |
|------|---------|---------|----------|----------|
| Full | 15 | 14 | ~210 | ~2 min |
| Quick | 4 | 6 | ~24 | ~12s |
| Validate | - | - | 0 | <1s |
| Ensure (exists) | - | - | 0 | <1s |
| Ensure (missing) | 4 | 6 | ~24 | ~12s |

Rate limiting: 0.5s delay between requests (configurable via `REQUEST_DELAY_SECONDS`).

---

## Do

- Use `uv run` from `labs/` directory for all commands
- Use `labs/.venv` (not root `.venv`) for Labs development
- Open `labs/` folder directly in VS Code/Cursor for correct interpreter
- Gracefully handle optional dependency failures (e.g., pandas-datareader)
- Test CsvAdapter programmatically before launching Marimo

## Do Not

- Do NOT run `marimo edit` from root venv - causes bokeh build errors
- Do NOT pin old bokeh versions (2.x incompatible with Python 3.12)
- Do NOT assume pandas-datareader works - it has compatibility issues with newer pandas
- Do NOT add `data/` prefix in mapping file base_path - CsvAdapter already includes it

## Module Structure (2026-02-04)

Labs uses **src-layout** for better packaging:

```
labs/
├── src/labs/          # Python package
│   ├── adapters/
│   └── cli/
├── config/            # Config/schemas (versioned)
│   ├── dsl-mapping.yaml
│   └── yahoo-symbols.json
├── data/              # Downloaded data (gitignored)
│   ├── price/
│   ├── fundamentals/
│   └── macro/
├── notebooks/         # Marimo notebooks (outside package)
├── tests/
└── pyproject.toml
```

**Key changes from flat layout:**
- Module code in `src/labs/` (not `labs/labs/`)
- Config files in `config/` (not `data/mapping/`)
- Notebooks at project root `notebooks/` (not inside package)
- Data at project root `data/` (not inside package)

## Gotchas

- **VIRTUAL_ENV warning**: Harmless. If root venv is active, `uv run` warns but uses correct labs venv anyway
- **bokeh==2.0.0**: Uses `SafeConfigParser` removed in Python 3.12 - Marimo may try to auto-install it
- **pandas-datareader**: `deprecate_kwarg()` TypeError with newer pandas - macro data download fails silently
- **Mapping paths**: `base_path: "price/eqt"` NOT `"data/price/eqt"` (adapter prepends `labs/data/`)
- **Src-layout structure**: Package is `src/labs/` not `labs/labs/` - cli at `src/labs/cli/`

## Lessons Learned (2026-02-03)

- Fixed duplicate `data/data/` path in mapping file - base_path should be relative to adapter's base_path
- Made macro data download optional - script now warns and continues if pandas-datareader fails
- Added `labs/.vscode/settings.json` to auto-select labs/.venv interpreter
- Created `run.sh` wrapper to suppress VIRTUAL_ENV warning (optional, uv run works fine)
- Marimo notebook uses Altair (not Bokeh) - no need for bokeh dependency

## Complete Data Download & Verification (2026-02-04)

### Full Dataset Coverage

**Verified all 212 data files created after complete download:**

| Asset Class | Files | Status |
|-------------|-------|--------|
| EQT (Equities) | 49 | ✓ |
| ETF | 2 | ✓ |
| IDX (Indices) | 3 | ✓ |
| CRY (Crypto) | 2 | ✓ |
| CUR (Currency) | 4 | ✓ |
| FIX (Fixed Income) | 2 | ✓ |
| Quarterly | 48 | ✓ |
| Annual | 49 | ✓ |
| Info | 49 | ✓ |
| Macro | 4 | ✓ (synthetic) |
| Catalog | 2 | ✓ |
| Database | 1 (12MB) | ✓ |

### Key Fixes Applied

1. **Makefile database check**: Fixed `yf-prod.db` → `sim_prod.db` (actual filename from db_generator)
2. **Macro data fallback**: Synthetic generation when `pandas-datareader` fails (SSL or compatibility)
3. **Database generation**: Successfully creates `sim_prod.db` with 78K price rows + 497 fundamentals rows
4. **Catalog generation**: 100% completeness score (10/10 modules probed successfully)

### Test Coverage Added

- **`TestFullDatasetVerification`** (15 tests): Verify all 212+ files by asset class
- **`TestComprehensiveDataVerification`** (9 tests): Sample data verification (AAPL/MSFT/GOOGL)
- **`TestQuickstartNotebook`** (4 tests): Notebook execution and data loading

All tests pass with full dataset.

### Download Workflow Verified

```bash
# Full workflow (from repo root):
make labs-run  # OR manual steps:

# 1. Sync dependencies
cd labs && uv sync --extra data --extra dev

# 2. Download all data (~2 min, 62 symbols)
CURL_CA_BUNDLE=/etc/ssl/cert.pem \
SSL_CERT_FILE=/etc/ssl/cert.pem \
REQUESTS_CA_BUNDLE=/etc/ssl/cert.pem \
uv run labs data download

# 3. Generate database (~5s)
uv run labs data db

# 4. Generate catalog (~100s, 100% completeness)
uv run labs data catalog

# 5. Verify all files
make labs-status
```

### Gotchas in Full Download

- **Macro data**: pandas-datareader has compatibility issues, script gracefully falls back to synthetic
- **Database filename**: Code uses `sim_prod.db` not `yf-prod.db` (fixed in Makefile)
- **Download time**: ~2 min for 62 symbols + fundamentals + macros (0.5s delay between requests)
- **SSL**: Must be set BEFORE download command (Makefile handles this)

### Do

- Run `make labs-status` to verify complete environment
- Test full download workflow before deploying changes to data services
- Verify all 212 files created (not just sample 3)
- Run comprehensive test suite when data available: `cd labs && uv run pytest tests/test_makefile_commands.py::TestFullDatasetVerification -v`

### Do Not

- Assume catalog generation without 100% completeness score
- Expect pandas-datareader to work without fallback
- Run partial downloads as smoke tests (they skip fundamentals and catalog)
- Trust `make labs-status` to show accurate counts without running download first
