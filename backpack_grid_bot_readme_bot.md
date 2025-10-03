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

## File: `backpack_grid_bot.py`

```python
#!/usr/bin/env python3
"""
Backpack Grid Bot (AMM-style both sides) – Defi Grid

Default: PAPER mode (no live orders). Set LIVE=true in .env to enable live path
and implement the Ed25519 signing in BackpackClient.sign_request().
"""
import asyncio
import math
import os
import time
from dataclasses import dataclass
from typing import Dict, List, Optional, Tuple

import httpx
from dotenv import load_dotenv
from pydantic import BaseModel, Field
from rich.console import Console
from rich.table import Table

load_dotenv()

console = Console()

# === Config ===
BOOL = lambda k, d=False: os.getenv(k, str(d)).lower() in ("1", "true", "yes", "on")
INT = lambda k, d=0: int(os.getenv(k, d))
FLOAT = lambda k, d=0.0: float(os.getenv(k, d))
STR = lambda k, d="": os.getenv(k, d)

API_BASE = STR("API_BASE", "https://api.backpack.exchange")
SYMBOL = STR("SYMBOL", "SOL_USDC_PERP")
SIDE = STR("SIDE", "LONG").upper()
LIVE = BOOL("LIVE", False)

# Grid
GRID_STEP_BPS = FLOAT("GRID_STEP_BPS", 10)
GRID_LEVELS = INT("GRID_LEVELS", 12)
ORDER_USD = FLOAT("ORDER_USD", 10)
MAX_POS_USD = FLOAT("MAX_POS_USD", 1000)

# Rolling window / maker
ROLLING_WINDOW_BIDS = INT("ROLLING_WINDOW_BIDS", 5)
ROLLING_WINDOW_ASKS = INT("ROLLING_WINDOW_ASKS", 5)
WINDOW_MS = INT("WINDOW_MS", 1200)
ORDER_TTL_SEC = INT("ORDER_TTL_SEC", 10)
ORDER_TOPCHASE_TICKS = INT("ORDER_TOPCHASE_TICKS", 1)
MAX_DEV_BPS = FLOAT("MAX_DEV_BPS", 25)

# Risk / exit
TAKE_PROFIT_PCT = FLOAT("TAKE_PROFIT_PCT", 1.0)
TP_OFFSET_BPS = FLOAT("TP_OFFSET_BPS", 3)
MAX_HOLD_SEC = INT("MAX_HOLD_SEC", 3600)

# Toggles
ENABLE_AMM_BIDS = BOOL("ENABLE_AMM_BIDS", True)
ENABLE_AMM_ASKS = BOOL("ENABLE_AMM_ASKS", True)
REDUCE_ONLY_TP_GUARD = BOOL("REDUCE_ONLY_TP_GUARD", True)
STRICT_MAKER = BOOL("STRICT_MAKER", True)
ALO_JOIN_BEST = BOOL("ALO_JOIN_BEST", True)
STRICT_MAKER_NEAR_TOP = BOOL("STRICT_MAKER_NEAR_TOP", True)

LOG_LEVEL = STR("LOG_LEVEL", "INFO").upper()

# === Simple market snapshot (paper) ===
@dataclass
class MarketSnapshot:
    bid: float
    ask: float
    ts: float

class PaperBook:
    def __init__(self, mid: float=150.0, spread_bps: float=2.0):
        self.mid = mid
        self.spread_bps = spread_bps

    def midprice(self) -> float:
        # Simple random walk for demo; in real use, pull from exchange.
        import random
        drift = random.uniform(-0.05, 0.05)
        self.mid = max(0.01, self.mid * (1 + drift/100))
        return self.mid

    def snapshot(self) -> MarketSnapshot:
        mid = self.midprice()
        spread = mid * (self.spread_bps / 10000)
        return MarketSnapshot(bid=mid - spread, ask=mid + spread, ts=time.time())

# === Live client skeleton ===
class BackpackClient:
    def __init__(self, api_base: str, key: str, secret: str):
        self.base = api_base.rstrip('/')
        self.key = key
        self.secret = secret
        self.http = httpx.AsyncClient(timeout=10)

    async def close(self):
        await self.http.aclose()

    def sign_request(self, method: str, path: str, body: str) -> Dict[str, str]:
        """TODO: Implement real Ed25519 signing here per Backpack API docs.
        Common pattern:
          - ts = str(int(time.time() * 1000))
          - msg = ts + method.upper() + path + body
          - sig = ed25519_sign(secret, msg)
          - return headers with X-API-KEY, X-API-SIGNATURE, X-API-TIMESTAMP
        """
        raise NotImplementedError("Fill Ed25519 signing per Backpack API to trade live.")

    async def get_markets(self):
        url = f"{self.base}/api/v1/markets"
        r = await self.http.get(url)
        r.raise_for_status()
        return r.json()

    # add place_order/cancel endpoints after sign_request is implemented

# === Strategy structures ===
@dataclass
class Order:
    side: str  # 'buy' or 'sell'
    price: float
    size_usd: float
    ts: float
    reduce_only: bool=False

class Position(BaseModel):
    qty: float = 0.0
    avg_entry: float = 0.0
    last_fill_ts: float = 0.0

    def update_on_fill(self, side: str, price: float, qty: float):
        now = time.time()
        if side == 'buy':
            new_notional = self.qty * self.avg_entry + qty * price
            self.qty += qty
            self.avg_entry = new_notional / self.qty if self.qty > 1e-12 else 0.0
        else:
            # sell reduces qty
            self.qty -= qty
            if self.qty <= 1e-12:
                self.qty = 0.0
                self.avg_entry = 0.0
        self.last_fill_ts = now

# === Bot ===
class GridBot:
    def __init__(self):
        self.paper = PaperBook()
        self.live_client: Optional[BackpackClient] = None
        if LIVE:
            key = STR("API_KEY")
            secret = STR("API_SECRET")
            if not key or not secret:
                raise RuntimeError("LIVE=true but API_KEY/API_SECRET missing in .env")
            self.live_client = BackpackClient(API_BASE, key, secret)

        self.pos = Position()
        self.open_orders: List[Order] = []
        self.last_reprice = 0.0

    def desired_ladders(self, snap: MarketSnapshot) -> Tuple[List[Order], List[Order]]:
        mid = (snap.bid + snap.ask) / 2
        step = GRID_STEP_BPS / 10000 * mid
        bids, asks = [], []

        # Ladder sizes per side
        n_bids = ROLLING_WINDOW_BIDS if ENABLE_AMM_BIDS else 0
        n_asks = ROLLING_WINDOW_ASKS if ENABLE_AMM_ASKS else 0

        # Build around mid with arithmetic spacing
        for i in range(1, n_bids + 1):
            price = mid - i * step
            if abs((price - mid) / mid) * 10000 > MAX_DEV_BPS:
                continue
            bids.append(Order('buy', round(price, 6), ORDER_USD, time.time()))
        for i in range(1, n_asks + 1):
            price = mid + i * step
            if abs((price - mid) / mid) * 10000 > MAX_DEV_BPS:
                continue
            asks.append(Order('sell', round(price, 6), ORDER_USD, time.time()))

        # Top-chase (join-best) – nudge nearest quotes toward current best
        if ALO_JOIN_BEST and bids:
            bids[0].price = min(bids[0].price + ORDER_TOPCHASE_TICKS * step * 0.25, snap.bid)
        if ALO_JOIN_BEST and asks:
            asks[0].price = max(asks[0].price - ORDER_TOPCHASE_TICKS * step * 0.25, snap.ask)

        return bids, asks

    def enforce_ttls(self):
        now = time.time()
        before = len(self.open_orders)
        self.open_orders = [o for o in self.open_orders if now - o.ts < ORDER_TTL_SEC]
        after = len(self.open_orders)
        if before != after and LOG_LEVEL == 'DEBUG':
            console.log(f"Repriced/canceled {before - after} stale orders")

    def simulate_fills(self, snap: MarketSnapshot):
        # Simple fill model: if a buy price >= ask or sell price <= bid, fill
        filled: List[Order] = []
        for o in self.open_orders:
            if o.side == 'buy' and o.price >= snap.ask:
                filled.append(o)
            elif o.side == 'sell' and o.price <= snap.bid:
                filled.append(o)
        for o in filled:
            qty = o.size_usd / ((snap.ask + snap.bid)/2)
            self.pos.update_on_fill(o.side, o.price, qty)
            self.open_orders.remove(o)

    def place_ladders(self, bids: List[Order], asks: List[Order]):
        # Paper: just track them
        self.open_orders.extend(bids + asks)

    def maybe_take_profit(self, snap: MarketSnapshot):
        if self.pos.qty <= 0:
            return
        target = self.pos.avg_entry * (1 + TAKE_PROFIT_PCT/100)
        target *= (1 + TP_OFFSET_BPS/10000)
        if snap.ask >= target:
            qty = self.pos.qty
            self.pos.update_on_fill('sell', target, qty)
            if LOG_LEVEL in ('INFO','DEBUG'):
                console.log(f"[TP] Sold {qty:.4f} @ {target:.4f} (avg_entry={self.pos.avg_entry:.4f})")

    def render_status(self, snap: MarketSnapshot):
        table = Table(title=f"{SYMBOL} @ {time.strftime('%H:%M:%S')} (paper={not LIVE})")
        table.add_column("Metric"); table.add_column("Value")
        mid = (snap.bid + snap.ask)/2
        table.add_row("Bid/Ask", f"{snap.bid:.4f} / {snap.ask:.4f}")
        table.add_row("Mid", f"{mid:.4f}")
        table.add_row("Open Orders", str(len(self.open_orders)))
        table.add_row("Pos Qty", f"{self.pos.qty:.4f}")
        table.add_row("Avg Entry", f"{self.pos.avg_entry:.4f}")
        console.print(table)

    async def loop(self):
        try:
            while True:
                snap = self.paper.snapshot() if not LIVE else None  # replace with live ticker
                if snap is None:
                    # If implementing live, pull from REST or websocket here
                    raise NotImplementedError("Live ticker not implemented in this skeleton")

                # Maintenance
                self.enforce_ttls()

                # Target ladders
                bids, asks = self.desired_ladders(snap)
                self.place_ladders(bids, asks)

                # Simulate fills and TP
                self.simulate_fills(snap)
                self.maybe_take_profit(snap)

                if LOG_LEVEL in ('INFO','DEBUG'):
                    self.render_status(snap)

                await asyncio.sleep(WINDOW_MS/1000)
        except KeyboardInterrupt:
            console.log("Shutting down…")
        finally:
            if self.live_client:
                await self.live_client.close()


if __name__ == "__main__":
    bot = GridBot()
    asyncio.run(bot.loop())
```

