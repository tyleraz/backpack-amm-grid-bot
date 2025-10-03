# README.md

Backpack Grid Bot (AMM-style, both sides) — Defi Grid Strategy

This is a lightweight Python grid-maker for Backpack Exchange, designed around the project’s “Defi Grid” principles:

- Arithmetic Long Grid with **rolling window** quotes on both sides (AMM-like),
- **Reduce-only** take-profit guard,
- **Join-best** chasing near top-of-book (tick chase),
- Presets: conservative / balanced / aggressive.

> ❗️Safety-first: The bot ships in **paper mode by default** (`LIVE=false`). Live-trading needs API hookup (see “Live trading” below).

---

## Quick start

1) **Python**: Use Python 3.10–3.12.

2) **Create a venv & install deps**

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install httpx python-dotenv pydantic rich
```

3) **Add one of the provided presets** to project root as `.env` (or keep multiple like `.env.long.*` and copy one to `.env` before running)

Example: `.env.long.aggressive`

```env
EXCHANGE=backpack
API_BASE=https://api.backpack.exchange
API_KEY=auto-ed25519
STRICT_MAKER=true
ALO_JOIN_BEST=true

SYMBOL=SOL_USDC_PERP
SIDE=LONG
LEVERAGE=7

GRID_STEP_BPS=6
GRID_LEVELS=22
ORDER_USD=25
MAX_POS_USD=3500

ROLLING_WINDOW_BIDS=10
ROLLING_WINDOW_ASKS=10
WINDOW_MS=900
ORDER_TTL_SEC=7
ORDER_TOPCHASE_TICKS=3
MAX_DEV_BPS=45

TAKE_PROFIT_PCT=1.6
TP_OFFSET_BPS=4
MAX_HOLD_SEC=7200

ENABLE_AMM_BIDS=true
ENABLE_AMM_ASKS=true

REDUCE_ONLY_TP_GUARD=true
STRICT_MAKER_NEAR_TOP=true
LOG_LEVEL=DEBUG

# Paper vs Live
LIVE=false              # <- keep false while testing
API_SECRET=             # required only for LIVE=true (see below)
```

You can also use the previously shared conservative/balanced presets by copying their content into `.env`.

4) **Run**

```bash
python backpack_grid_bot.py
```

The bot will run in paper mode (simulated orderbook/PNL) and print actions in the console.

5) **Stop** with Ctrl+C.

---

## How it works (core ideas)

- **Rolling Window Maker**: Maintains `ROLLING_WINDOW_BIDS` and `ROLLING_WINDOW_ASKS` ladder sizes around the mid—old quotes are canceled/repriced every `ORDER_TTL_SEC` seconds. Repricing obeys `MAX_DEV_BPS` so you don’t drift too far from mid.
- **Join-best chase**: `ORDER_TOPCHASE_TICKS` lets your top quote nudge to join the current best price while staying maker.
- **Reduce-only TP**: Exit orders created with reduce-only semantics so PnL taking won’t increase exposure accidentally.
- **AMM both sides**: `ENABLE_AMM_BIDS/ASKS` determine whether to seed both bid & ask ladders. For a pure long grid, you can still enable both to earn maker rebates and mean-revert micro-alpha while directional bias comes from the ladder skew and `SIDE` handling.

---

## Live trading (optional)

- Set `LIVE=true` and provide `API_KEY` and `API_SECRET`. The included `BackpackClient` has clear **TODO** blocks where you should fill Backpack’s exact signing rules.
- Common pattern (to implement in `sign_request()`):
  - Create a timestamp,
  - Concatenate (timestamp + method + path + body),
  - Sign with **Ed25519** using your secret,
  - Send headers like `X-API-KEY`, `X-API-SIGNATURE`, `X-API-TIMESTAMP`.

Until those TODOs are completed, the bot will raise a clear error when `LIVE=true`.

> Tip: Test symbols with smallest sizes first and set tight `MAX_POS_USD`.

---

## Key environment variables

| Key | Meaning |
|-----|---------|
| `GRID_STEP_BPS` | Distance between grid levels in bps (0.01% = 1 bps). |
| `GRID_LEVELS` | Total levels per side. |
| `ORDER_USD` | Quote size per order in quote-USD value. |
| `MAX_POS_USD` | Max gross exposure cap. |
| `ORDER_TTL_SEC` | How long a quote lives before reprice/cancel-replace. |
| `ORDER_TOPCHASE_TICKS` | How aggressively the top quote joins best. |
| `MAX_DEV_BPS` | Max allowed drift from mid for placing quotes. |
| `TAKE_PROFIT_PCT` | Percent move for TP from average entry. |
| `TP_OFFSET_BPS` | Additional spacing to avoid taker slips on TP. |
| `MAX_HOLD_SEC` | Force-close window for stale positions. |
| `ENABLE_AMM_BIDS/ASKS` | Toggle ladder sides. |
| `STRICT_MAKER` | Only place ALO/maker orders. |

---

## Troubleshooting

- **Orders not filling / only open orders**: relax `STRICT_MAKER_NEAR_TOP`, increase `ORDER_USD`, increase `ORDER_TOPCHASE_TICKS`, or widen `MAX_DEV_BPS`.
- **Quotes too slow to follow price**: reduce `ORDER_TTL_SEC`, reduce `GRID_STEP_BPS`, or increase `ROLLING_WINDOW_*`.
- **Position grows too fast**: reduce `ORDER_USD`, reduce `GRID_LEVELS`, or lower `MAX_POS_USD`.

---
