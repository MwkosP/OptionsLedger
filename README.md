# MarketLedger 📒

A fault-tolerant pipeline that collects ephemeral market data from multiple providers on a schedule and stores it as Parquet files in Google Cloud Storage. Built to run unattended for years.

Three collectors, one pipeline, one bucket.

-----

## Collectors

### Options — `collectors/options/`

Full options chain snapshots including Greeks and IV from crypto derivatives exchanges. Hourly.

|Provider        |Currencies|Hours       |
|----------------|----------|------------|
|Deribit         |BTC, ETH  |24/7        |
|OKX             |BTC, ETH  |24/7        |
|IBIT (BlackRock)|BTC       |Market hours|
|Bybit           |BTC, ETH  |24/7        |
|Binance         |BTC, ETH  |24/7        |
|Bullish         |BTC       |24/7        |

### Liquidity — `collectors/liquidity/`

Derived order book metrics — spread, depth, imbalance, large walls. Captures liquidity structure without the cost of raw order book storage. Hourly or 5-minute.

|Provider|Pairs             |Hours|
|--------|------------------|-----|
|Binance |BTC/USDT, ETH/USDT|24/7 |
|OKX     |BTC/USDT, ETH/USDT|24/7 |
|Bybit   |BTC/USDT, ETH/USDT|24/7 |

### Prediction — `collectors/prediction/`

Hourly snapshots of prediction market odds across platforms. Historical odds datasets for these markets don’t exist freely anywhere. Powered by Unimarkets.

|Provider  |Auth   |
|----------|-------|
|Polymarket|None   |
|Kalshi    |RSA key|
|Limitless |HMAC   |
|Gemini    |API key|

-----

## Project Structure

```
MarketLedger/
│
├── collectors/
│   ├── options/
│   │   ├── providers/
│   │   │   ├── base.py
│   │   │   ├── deribit.py
│   │   │   ├── okx.py
│   │   │   ├── ibit.py
│   │   │   ├── bybit.py
│   │   │   ├── binance.py
│   │   │   └── bullish.py
│   │   └── fetcher.py        # fetchOptionsChain()
│   │
│   ├── liquidity/
│   │   ├── providers/
│   │   │   ├── base.py
│   │   │   ├── binance.py
│   │   │   ├── okx.py
│   │   │   └── bybit.py
│   │   └── fetcher.py        # fetchLiquidity()
│   │
│   └── prediction/
│       ├── providers/
│       │   ├── base.py
│       │   ├── polymarket.py
│       │   ├── kalshi.py
│       │   ├── limitless.py
│       │   └── gemini.py
│       └── fetcher.py        # fetchPredictionMarkets()
│
├── core/
│   ├── base.py               # Shared abstract provider class
│   ├── storage.py            # GCS upload + local fallback buffer
│   ├── error_handler.py      # Retry logic, alerting, structured logging
│   └── config.py             # Keys, bucket name, retry settings
│
├── models/
│   ├── options.py            # OptionContract dataclass
│   ├── liquidity.py          # LiquiditySnapshot dataclass
│   └── prediction.py         # MarketOdds dataclass
│
├── main.py                   # Entry point — runs all collectors
├── tests/
├── Dockerfile
├── pyproject.toml
├── uv.lock
└── cloudbuild.yaml           # CI/CD — tests first, deploy only if passing
```

-----

## Storage Layout

```
MarketLedger-GCS/
│
├── Options/
│   ├── Deribit/
│   │   └── 2026/04/25/
│   │       ├── deribit_BTC_2026-04-25_14-00.parquet
│   │       └── deribit_ETH_2026-04-25_14-00.parquet
│   ├── OKX/
│   ├── IBIT/
│   ├── Bybit/
│   ├── Binance/
│   └── Bullish/
│
├── Liquidity/
│   ├── Binance/
│   │   └── 2026/04/25/
│   │       ├── binance_BTCUSDT_2026-04-25_14-00.parquet
│   │       └── binance_ETHUSDT_2026-04-25_14-00.parquet
│   ├── OKX/
│   └── Bybit/
│
└── Prediction/
    ├── Polymarket/
    │   └── 2026/04/25/
    │       └── polymarket_2026-04-25_14-00.parquet
    ├── Kalshi/
    ├── Limitless/
    └── Gemini/
```

File naming: `{provider}_{pair_or_currency}_{YYYY-MM-DD}_{HH-MM}.parquet`

-----

## Usage

```python
from collectors.options.fetcher    import fetchOptionsChain
from collectors.liquidity.fetcher  import fetchLiquidity
from collectors.prediction.fetcher import fetchPredictionMarkets

# Options
fetchOptionsChain(providers=["deribit", "okx"], currencies=["BTC", "ETH"])
fetchOptionsChain(providers=["all"], currencies=["BTC"])

# Liquidity
fetchLiquidity(providers=["binance"], pairs=["BTC/USDT"])
fetchLiquidity(providers=["all"], pairs=["BTC/USDT", "ETH/USDT"])

# Prediction markets
fetchPredictionMarkets(providers=["polymarket"], query="bitcoin")
fetchPredictionMarkets(providers=["all"])
```

-----

## Infrastructure

|Component |Service                 |Cost               |
|----------|------------------------|-------------------|
|Execution |Cloud Run               |Free tier          |
|Scheduling|Cloud Scheduler (hourly)|Free tier          |
|Storage   |GCS bucket              |Free up to 5 GB    |
|Logs      |Cloud Logging           |Free up to 50 GB/mo|

Estimated storage across all collectors at hourly frequency: **~2 GB over 2 years** — within the free tier.

-----

## CI/CD

Every `git push` to `main` triggers Google Cloud Build:

1. Runs `pytest tests/` — pipeline stops here if any test fails
1. Builds Docker image on Google’s servers (no local Docker needed)
1. Deploys to Cloud Run only if tests pass

Last good version stays running if a bad push is made.

-----

## Error Handling

- Each collector and provider runs in full isolation — one failure never affects others
- Automatic retry with exponential backoff (up to 3 attempts)
- If GCS is unreachable, data is saved locally and re-uploaded next run
- Alerts fire if any provider fails 3 consecutive runs

-----

## Adding a Collector

1. Create `collectors/{name}/providers/base.py` inheriting from `core/base.py`
1. Implement each provider
1. Create `collectors/{name}/fetcher.py` with unified fetch function
1. Add a model to `models/`
1. Register in `main.py`

## Adding a Provider to an Existing Collector

1. Create `collectors/{collector}/providers/{name}.py`
1. Inherit the collector’s base class
1. Register in the collector’s `fetcher.py`

Done — no other changes needed.