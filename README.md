# OptionsLedger 📒

A fault-tolerant pipeline that fetches crypto options data from multiple providers every hour and stores it as Parquet files in Google Cloud Storage. Built to run unattended for years.

-----

## Providers

|Provider        |Currencies|Hours       |
|----------------|----------|------------|
|Deribit         |BTC, ETH  |24/7        |
|OKX             |BTC, ETH  |24/7        |
|IBIT (BlackRock)|BTC       |Market hours|
|Bybit           |BTC, ETH  |24/7        |
|Binance         |BTC, ETH  |24/7        |
|Bullish         |BTC       |24/7        |

-----

## Project Structure

```
OptionsLedger/
│
├── providers/
│   ├── base.py           # Abstract base — all providers inherit this
│   ├── deribit.py
│   ├── okx.py
│   ├── ibit.py
│   ├── bybit.py
│   ├── binance.py
│   └── bullish.py
│
├── models/
│   └── options.py        # Unified OptionContract dataclass
│
├── fetcher.py            # fetchOptionsChain() — unified entry point
├── storage.py            # GCS upload + local fallback buffer
├── error_handler.py      # Retry logic, alerting, structured logging
├── config.py             # Keys, bucket name, retry settings
├── main.py               # Cloud Run entry point
│
├── tests/
├── Dockerfile
└── requirements.txt
```

-----

## Storage Layout

```
OptionsHistorical/
├── Deribit/
│   └── 2026/
│       └── 04/
│           └── 25/
│               ├── deribit_BTC_2026-04-25_00-00.parquet
│               └── deribit_ETH_2026-04-25_00-00.parquet
├── OKX/
├── IBIT/
├── Bybit/
├── Binance/
└── Bullish/
```

File naming: `{provider}_{currency}_{YYYY-MM-DD}_{HH-MM}.parquet`

-----

## Usage

```python
from fetcher import fetchOptionsChain

# Single provider
fetchOptionsChain(providers=["deribit"], currencies=["BTC"])

# Multiple
fetchOptionsChain(providers=["deribit", "okx"], currencies=["BTC", "ETH"])

# Everything
fetchOptionsChain(providers=["all"], currencies=["BTC", "ETH"])
```

-----

## Infrastructure

|Component |Service                 |Cost               |
|----------|------------------------|-------------------|
|Execution |Cloud Run               |Free tier          |
|Scheduling|Cloud Scheduler (hourly)|Free tier          |
|Storage   |GCS bucket              |Free up to 5 GB    |
|Logs      |Cloud Logging           |Free up to 50 GB/mo|

Estimated storage at hourly frequency across all providers: **~1.5 GB over 2 years** — well within the free tier.

-----

## Error Handling

- Each provider runs in isolation — one failure never affects others
- Automatic retry with exponential backoff (up to 3 attempts)
- If GCS is unreachable, data is saved locally and re-uploaded next run
- Alerts fire if any provider fails 3 consecutive runs

-----

## Adding a Provider

1. Create `providers/{name}.py` inheriting `BaseProvider`
1. Implement `fetch(currency) -> list[OptionContract]`
1. Register it in `fetcher.py`

Done — no other changes needed.