# NEXUS Swap Backend — Railway Deployment

BTC → USDT TRC-20 DEX swap via **SunSwap V2** pools on TRON.

## Architecture

```
User Browser
    │
    ├──[GET /api/price]──────────► CoinGecko API (BTC/USD)
    │                              SunSwap pool (TRX/USDT rate)
    │
    ├──[POST /api/quote]─────────► CoinGecko (BTC→USD)
    │                              CoinGecko (TRX/USD)
    │                              SunSwap Router.getAmountsOut()
    │
    ├──[POST /api/swap/execute]──► TronWeb signs tx
    │                              TronGrid broadcasts to TRON mainnet
    │                              SunSwap swapExactETHForTokens()
    │
    └──[GET /api/tx/:txid]───────► TronGrid tx status poll
```

## Swap Route

```
BTC (off-chain price) → equivalent TRX (SUN units) → USDT TRC-20
                                    ↑
                               SunSwap V2 WTRX/USDT pool
```

> **Note:** The BTC→TRX bridge is handled by the user manually or via a cross-chain bridge (BitTorrent Chain / Allbridge). This backend executes the **TRON DEX leg** (TRX → USDT via SunSwap).

## Open APIs Used

| API | Purpose | Key Required? |
|-----|---------|---------------|
| CoinGecko `/simple/price` | BTC/USD, TRX/USD prices | No (free tier) |
| SunSwap V2 Router `getAmountsOut` | DEX pool quote | No (on-chain call) |
| TronGrid `/v1/transactions` | Broadcast + status | No (optional for rate limit) |
| sun.io `/api/v3/pool/list` | Pool stats | No |

## Deploy to Railway

### 1. Push to GitHub

```bash
git init
git add .
git commit -m "nexus swap backend"
gh repo create nexus-swap --public --push
```

### 2. New Railway Project

1. Go to [railway.app](https://railway.app) → **New Project**
2. **Deploy from GitHub repo** → select `nexus-swap`
3. Railway auto-detects Node.js via `package.json`

### 3. Set Environment Variables

In Railway → Project → **Variables**:

```
TRONGRID_API_KEY=your_key_here   # optional, free at trongrid.io
```

### 4. Get your Railway domain

Railway → Project → **Settings** → **Domains** → Generate domain

Copy the URL (e.g. `https://nexus-swap-production.up.railway.app`)

### 5. Update frontend

In `swap-widget.html`, find and replace:

```js
const API_BASE = ...
  : 'https://YOUR-RAILWAY-APP.railway.app'; // ← paste your Railway URL here
```

## API Reference

### GET /api/price
Returns current BTC/USD and SunSwap pool TRX rate.

### POST /api/quote
```json
{ "btcAmount": "0.01", "slippageBps": 50 }
```
Returns full DEX quote with expected output, price impact, fees.

### POST /api/swap/execute
```json
{
  "privateKey": "hex64chars",
  "btcAmount": "0.01",
  "trxAmount": "1234.56",
  "minUsdtOut": "830.00",
  "slippageBps": 50
}
```
Signs + broadcasts the SunSwap transaction. Returns `txid`.

### GET /api/tx/:txid
Polls TronGrid for confirmation status.

### GET /api/balance/:address
Returns USDT TRC-20 and TRX balance for a TRON address.

### GET /api/pool/stats
Returns SunSwap WTRX/USDT pool liquidity & volume.

## Security Notes

- Private keys are **never logged** (`console.error` only logs `err.message`)
- Keys are cleared from memory with `privateKey = null` in `finally` block
- Use a **dedicated hot-wallet** with only the TRX needed for the swap
- Deploy with HTTPS only (Railway provides this automatically)
- Add `CORS_ORIGIN` env var to restrict to your frontend domain in production

## Contract Addresses

| Contract | Address |
|----------|---------|
| USDT TRC-20 | `TR7NHqjeKQxGTCi8q8ZY4pL8otSzgjLj6t` |
| WTRX | `TNUC9Qb1rRpN8CkFuhRRBFKBM6MLFA5cbb` |
| SunSwap V2 Router | `TKzxdSv2FZKQrEqkKVgp5DcwEXBEKMg2Ax` |
