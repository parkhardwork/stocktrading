# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

```bash
python run.py          # starts uvicorn on 0.0.0.0:8000 with --reload
```

`run.py` and `app/main.py` are duplicate entrypoints — both call `uvicorn.run("app.main:app", ...)`. Prefer `run.py`.

There is **no `requirements.txt` / `pyproject.toml`** in the repo. Required packages (inferred from imports): `fastapi`, `uvicorn`, `pydantic`, `pydantic-settings`, `python-dotenv`, `supabase`, `requests`, `pandas`, `numpy`, `pytz`, `schedule`, `yfinance`. `predict.py` additionally needs `tensorflow`, `scikit-learn`, `matplotlib` and is **not** invoked from the FastAPI app — treat it as a notebook-style script (it even has `!pip install supabase` at the top).

The `tests/` directory contains only an empty `__init__.py`. There is no test runner configured.

## Required environment (`.env`)

`app/core/config.py` loads these via pydantic-settings; `KIS_APPKEY` and `KIS_APPSECRET` are required (Field(...)), the rest have defaults:

- `KIS_APPKEY`, `KIS_APPSECRET`, `KIS_CANO`, `KIS_ACNT_PRDT_CD`, `KIS_USE_MOCK` — Korea Investment & Securities (KIS) credentials
- `TR_ID` — KIS transaction code (mock prefix `VTTC...` vs real `TTTC...`); used only by `get_domestic_balance`. Most other KIS calls hard-code their own TR_IDs (e.g. `VTTS3012R` for overseas balance).
- `SUPABASE_URL`, `SUPABASE_KEY`
- `ALPHA_VANTAGE_API_KEY` — for `NEWS_SENTIMENT`

`KIS_USE_MOCK=true` (default) makes `settings.kis_base_url` return `KIS_BASE_URL` (openapivts...:29443); set false to hit real (`KIS_REAL_URL`, openapi...:9443). Hard-coded TR_IDs throughout `balance_service.py` are the **mock** values — switching to real trading requires changing those too, not just the URL flag.

## Architecture

### Request entry & router wiring

`app/main.py` builds the FastAPI app with a `lifespan` context manager that:
1. runs `update_economic_data_in_background()` once at startup (synchronously awaited — startup blocks until done),
2. starts three background schedulers (`start_economic_data_scheduler`, `start_scheduler`, `start_sell_scheduler`),
3. stops them all on shutdown.

`app/api/api.py` is the single place where routers are mounted:
- `/stocks/recommendations` → `app/api/routes/stock_recommendations.py`
- `/stocks` → `app/api/routes/stocks.py`
- `/economic` → `app/api/routes/economic.py`
- `/balance` → `app/api/routes/balance.py`

### The three schedulers (`app/utils/scheduler.py`)

All three run in **daemon threads** using the `schedule` library, polling with `time.sleep(1)`. Scheduled jobs are sync; each one wraps the actual async work with `asyncio.run(...)`.

| Scheduler | Cadence (KST) | Job | Notes |
|---|---|---|---|
| Buy | 00:00 daily | `_execute_auto_buy` | runs once a day; pulls combined recs and places limit orders |
| Sell | every 1 min | `_execute_auto_sell` | guarded by US market hours check (NY 09:30–16:00 ET, weekdays) — outside hours it returns immediately |
| Economic data | 06:05 daily | `update_economic_data_in_background` | also run once at app startup |

`StockScheduler` is a singleton at module level (`stock_scheduler`). Buy and sell share one thread (`scheduler_thread`); the thread loop runs while either flag is true. Job filters use `job.job_func.__name__` to cancel only buy or sell jobs without affecting the other.

### Stock-recommendation pipeline

The investment logic lives in `app/services/stock_recommendation_service.py`. The data flow across Supabase tables:

1. **`economic_and_stock_data`** — daily snapshots populated by `stock.collect_economic_data` (FRED API + yfinance). Updated by `economic_service.update_economic_data_in_background`.
2. **`stock_recommendations`** — `generate_technical_recommendations()` reads the last 180 days from #1, computes SMA20/SMA50 / RSI(14) / MACD(12,26,9) per ticker, **DELETEs the entire table** then re-inserts the latest row per stock.
3. **`stock_analysis_results`** — populated externally (e.g. by `predict.py`'s ML model). `get_stock_recommendations()` filters to `Accuracy ≥ 80%` and `Rise Probability ≥ 3%`.
4. **`ticker_sentiment_analysis`** — `fetch_and_store_sentiment_for_recommendations()` calls Alpha Vantage `NEWS_SENTIMENT` for each candidate ticker (5 s sleep between calls), again **deletes the whole table** before inserting.
5. **`get_combined_recommendations_with_technical_and_sentiment()`** is what the buy scheduler consumes. It joins #2 + #3 + #4 with this rule:
   - if `sentiment_score ≥ 0.15`: require ≥ 2 of {golden_cross, RSI<50, MACD buy} to be true
   - else: require all 3
   - composite score = `0.3·rise_probability + 0.4·weighted_tech_count + 0.3·sentiment_score`, sorted desc

The sell side (`get_stocks_to_sell`) reads live KIS balances + `stock_recommendations` + `ticker_sentiment_analysis` and triggers on:
- ≥ +5% gain (take profit) or ≤ −7% loss (stop loss), OR
- 3+ technical sell signals (dead cross, RSI > 70, no MACD buy), OR
- sentiment < −0.15 AND 2+ technical sell signals

### Korean-name ↔ ticker bridge

`STOCK_TO_TICKER` in `stock_recommendation_service.py` is the **only** mapping between Korean stock names (used as DB column names in `economic_and_stock_data` and as `종목` values in `stock_recommendations`) and US tickers (used by KIS/Alpha Vantage). When adding a new symbol you must update this dict; otherwise it gets silently dropped from results.

### Exchange-code translation gotcha

The KIS API uses two different exchange-code conventions and code must convert between them every time:
- **Quote / current-price API (`get_current_price`)** wants `NAS` / `NYS` (3-letter)
- **Order API (`order_overseas_stock`)** and balance responses use `NASD` / `NYSE` (4-letter)

See the `api_exchange_code` translations in `_execute_auto_buy` and `_execute_auto_sell`. Pass the wrong one and the call silently fails with `rt_cd != "0"`.

### KIS access token

`balance_service.get_access_token()` is the single token entry point used by every KIS call. It implements three layers:
1. in-memory cache (`_token_cache`) — checked first
2. `access_tokens` Supabase table — checked second, expiration parsed via `auth_service.parse_expiration_date` (tolerates the 5-digit-microsecond format Supabase returns)
3. `refresh_token_with_retry` — falls back to issuing a new token, persists it to Supabase

KIS enforces a 1-token-per-minute limit (error code `EGW00133`). The code self-throttles via `_last_refresh_time` + 60 s sleep and a `Lock` to serialize refreshes.

## Things to know before editing

- **Schemas live in two places.** Pydantic request/response models are in `app/schemas/stock.py`; `BaseModel` request bodies are also defined inline in route files (e.g. `OrderResvRequest` in `routes/balance.py`).
- **Korean comments and identifiers throughout.** Column names in DB and DataFrames are Korean (`종목`, `날짜`, `골든_크로스`, `MACD_매수_신호`). Don't rename them — they're the live schema.
- **`stock.py` is imported as a top-level module** by `app/services/economic_service.py` (`from stock import collect_economic_data`). It hasn't been moved into the `app/` package; keep it at repo root.
- `predict.py` is an out-of-band ML training/prediction script (Keras transformer). It is **not** wired into the FastAPI app and contains a `!pip install` line — assume it's run manually in a notebook and don't try to import from it.
- Logging: `stock_scheduler.log` (rolling text file) at repo root + stdout. The file is `.gitignore`d? — no, it's currently committed; avoid letting it grow further in PRs.
- CORS is wide-open (`allow_origins=["*"]`).
- The `.env` in this repo currently contains real-looking API keys / Supabase keys committed to the working tree. Treat them as secrets in any external-facing change.
