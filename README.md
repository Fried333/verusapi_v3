# Verus Ticker API v3

> **⚠️ Deprecated** — superseded by [verusapi_v4](https://github.com/Fried333/verusapi_v4), which produces byte-identical output with ~7× less RPC load. This repo is kept for reference only; new deployments should run v4. The longer-term plan is for the explorer's market-worker (in [scan-verus-cx](https://github.com/Fried333/scan-verus-cx)) to absorb both.

Python/FastAPI service that aggregates Verus DEX trading-pair data from a local `verusd` and serves it in CoinGecko / CoinMarketCap / Coinpaprika formats for volume-aggregator ingestion.

## Table of contents

- [What you get](#what-you-get)
- [What you provide](#what-you-provide)
- [Quick start](#quick-start)
- [API endpoints](#api-endpoints)
- [Response formats](#response-formats)
- [Configuration](#configuration)
- [Operating in production](#operating-in-production)
- [Migrating to v4](#migrating-to-v4)
- [License](#license)
- [Disclaimer](#disclaimer)

## What you get

- Trading-pair ticker data for the Verus DEX (VRSC + PBaaS chains) in three aggregator formats
- 60-second cached endpoints (~50 ms response) + optional live debug endpoints
- Volume-weighted price aggregation across multiple converters trading the same pair
- ERC20 symbol mapping for cross-chain compatibility

## What you provide

- Python 3.8+
- A running local `verusd` (with `txindex`/`addressindex` enabled for the chains you index)
- RPC credentials for that daemon (read from your `.env`)

## Quick start

```bash
git clone https://github.com/Fried333/verusapi_v3.git
cd verusapi_v3
pip install fastapi uvicorn requests python-dotenv
cp env_format .env
$EDITOR .env          # fill RPC creds
python3 main.py
# → http://localhost:8765
```

Sanity check:
```bash
curl http://localhost:8765/health
curl http://localhost:8765/coingecko | head -c 200
```

## API endpoints

### Health & status

| Endpoint | Description |
|---|---|
| `GET /health` | RPC connection, cache status, block heights |
| `GET /verussupply` | VRSC supply info |
| `GET /stats` | CoinGecko format rendered as HTML with USD volume |

### Cached endpoints (60 s TTL)

| Endpoint | Format |
|---|---|
| `GET /coingecko` | CoinGecko |
| `GET /coinmarketcap` | CoinMarketCap |
| `GET /coinpaprika` | Coinpaprika |
| `GET /coinmarketcap_iaddress` | CMC format with Verus i-address identifiers |

### Live debug endpoints (disabled by default)

Set `ENABLE_LIVE_ENDPOINTS=true` in `.env` to enable. Each makes fresh RPC calls — slower (~2-5 s) but reflects current chain state.

| Endpoint |
|---|
| `GET /coingecko_live` |
| `GET /coinmarketcap_live` |
| `GET /coinpaprika_live` |
| `GET /coinmarketcap_iaddress_live` |

## Response formats

### CoinGecko

```json
[
  {
    "ticker_id": "VRSC_DAI.vETH",
    "pool_id": "iH37kRsdfoHtHK5TottP1Yfq8hBSHz9btw",
    "base_currency": "VRSC",
    "target_currency": "DAI.vETH",
    "last_price": "2.15310337",
    "base_volume": "1234.56789",
    "target_volume": "2658.91234"
  }
]
```

### CoinMarketCap

```json
{
  "0": {
    "base_name": "VRSC",
    "base_symbol": "VRSC",
    "quote_name": "DAI.vETH",
    "quote_symbol": "DAI.vETH",
    "last_price": "2.15310337",
    "base_volume": "1234.56789",
    "quote_volume": "2658.91234"
  }
}
```

### Coinpaprika

```json
{
  "code": "200000",
  "data": {
    "time": 1756195613998,
    "ticker": [
      { "symbol": "VRSC-DAI.vETH", "last": "2.15310337", "volume": "1234.56789", "high": "2.20000000", "low": "2.10000000", "open": "2.18000000" }
    ]
  }
}
```

## Configuration

```env
# VRSC RPC connection
VERUS_RPC_HOST=127.0.0.1
VERUS_RPC_PORT=27486
VERUS_RPC_USER=your_rpc_user
VERUS_RPC_PASSWORD=your_rpc_password

# Live endpoints off by default (CPU-heavy — only enable for dev)
ENABLE_LIVE_ENDPOINTS=false
```

Add per-chain RPC blocks for PBaaS (vDEX, vARRR, CHIPS) using the same naming pattern — see `env_format` in the repo.

### Volume-weighted price aggregation

Multiple converters trading the same pair contribute to one price via quote-volume weighting:

```
weighted_price = (price1 × quote_vol1 + price2 × quote_vol2) / (quote_vol1 + quote_vol2)
```

Stablecoins (DAI, vUSDC, vUSDT) are hardcoded at $1.00.

## Operating in production

### Logs

stdout/stderr from `python3 main.py`. Hostile under load — v3 emits ~9,600 log lines per 60-s refresh cycle. **Recommend v4** if you care about log volume.

### Common errors

| Symptom | Likely cause |
|---|---|
| `/health` shows RPC disconnected | wrong `VERUS_RPC_*` creds, daemon not running, or wrong port |
| Cached endpoints return empty / 500 | converter discovery failed — check daemon is fully synced and has DEX baskets |
| Stale data past 60 s TTL | background refresh hung on a slow RPC; restart resolves |

### Upgrading to v4

v4 is a drop-in replacement with the same port + endpoints + output. `git clone github.com/Fried333/verusapi_v4`, copy your `.env`, switch the systemd unit. See [Migrating to v4](#migrating-to-v4).

## Migrating to v4

v4's behaviour is intentionally byte-identical to v3 (validated 2026-03-11). The only difference is efficiency:

| Metric | v3 | v4 |
|---|---|---|
| Cached response time | ~280 ms | ~21 ms |
| RPC calls / 60-s cycle | ~286 | ~40 |
| Log lines / cycle | ~9,600 | ~360 |

Switching:
1. Clone v4 alongside v3
2. Copy `.env` over (same shape)
3. Stop v3's systemd unit, point `ExecStart` at v4's `main.py`
4. Start; `curl /health` to confirm

## License

MIT — see [LICENSE](./LICENSE) if present, or the standard MIT terms.

## Disclaimer

This software is provided **"AS IS"**, without warranty of any kind, express or implied. In no event shall the authors or copyright holders be liable for any claim, damages, or other liability arising from the use of this software. Run your own review before relying on it for production volume-aggregator reporting.
