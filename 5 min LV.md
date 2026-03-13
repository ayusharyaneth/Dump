
You are an expert Python developer. Generate a complete, production-ready Python project for an automated Polymarket trading bot. Every file must be fully implemented — no stubs, no `TODO` comments, no placeholder functions. Follow every requirement exactly.

---

## PROJECT OVERVIEW

Build a Python bot that:
1. Connects to the **Polymarket CLOB API** (Central Limit Order Book)
2. Discovers and monitors active **BTC 5-minute prediction markets**
3. Executes a **dynamic one-side-at-a-time rebalancing strategy**
4. Automatically places real orders on Polymarket using the user's wallet private key — signed on-chain via ECDSA
5. Tracks positions, costs, and PnL per market in-memory
6. Detects market closure and records final PnL
7. Sends all activity to a Telegram bot with `#logs` and `#trades` channels
8. Provides a full Telegram dashboard with 10 commands (listed below)
9. Enforces a daily loss limit and full panic/resume system
10. **Runs a complete Paper Trading (Shimmer) mode** — a drop-in simulation layer that:
    - Uses **real live Polymarket prices** for all signals and decisions
    - **Simulates order fills instantly** without touching the wallet or blockchain
    - Tracks a **virtual USDC balance** and all simulated positions
    - Persists all paper trading history in **SQLite** (survives restarts)
    - Computes **full strategy performance analytics** (win rate, PnL, Sharpe, drawdown, per-rule breakdown)
    - Sends paper trade events to Telegram with a `🧪 [PAPER]` tag so they are visually distinct
    - Can run **simultaneously alongside live trading** (separate state store) OR **standalone** when `LIVE_TRADING=false`

---

## FILE STRUCTURE

Generate exactly these files:

```
polymarket_bot/
├── main.py
├── config.py
├── api/
│   ├── __init__.py
│   ├── clob_client.py              # Live Polymarket CLOB REST client
│   └── auth.py                     # Wallet ECDSA signing + L2 API auth
├── strategy/
│   ├── __init__.py
│   ├── position.py                 # Position + OpenOrder dataclasses, PnL math
│   ├── trend.py                    # Trend detection
│   └── decision.py                 # Core decision engine (shared by live + paper)
├── trader/
│   ├── __init__.py
│   └── executor.py                 # Live order executor (panic/loss guards)
├── paper_trading/
│   ├── __init__.py
│   ├── paper_clob.py               # PaperCLOBClient — shims place_order/cancel with simulation
│   ├── paper_store.py              # PaperStateStore — separate paper positions + virtual balance
│   ├── paper_executor.py           # PaperExecutor — runs decisions through paper fills
│   ├── paper_analytics.py          # Analytics engine: win rate, PnL, Sharpe, drawdown, per-rule
│   └── paper_db.py                 # SQLite persistence: paper trades, markets, sessions
├── monitor/
│   ├── __init__.py
│   ├── market_finder.py
│   └── closure_checker.py          # Also handles paper position settlement on close
├── state/
│   ├── __init__.py
│   └── store.py                    # Live StateStore (thread-safe)
├── telegram_bot/
│   ├── __init__.py
│   ├── bot.py
│   ├── notifier.py                 # send_paper_trade(), send_paper_market_closed() added
│   └── dashboard.py                # All 10 command handlers
├── utils/
│   ├── __init__.py
│   └── logger.py
├── paper_trades.db                 # SQLite DB (auto-created on first run)
├── requirements.txt
└── .env.example
```

---

## DEPENDENCIES (requirements.txt)

```
python-dotenv==1.0.0
requests==2.31.0
websocket-client==1.6.4
eth-account==0.10.0
web3==6.15.1
pydantic==2.5.3
python-dateutil==2.8.2
python-telegram-bot==20.7
```

(SQLite is part of Python stdlib — no extra package needed.)

---

## ENVIRONMENT VARIABLES (.env.example)

```
# Polymarket wallet + API (required for live trading; optional if LIVE_TRADING=false)
POLYMARKET_PRIVATE_KEY=your_wallet_private_key_hex_without_0x
POLYMARKET_WALLET_ADDRESS=your_wallet_address_0x
POLYMARKET_API_KEY=your_api_key
POLYMARKET_API_SECRET=your_api_secret
POLYMARKET_API_PASSPHRASE=your_passphrase

# Telegram
TELEGRAM_BOT_TOKEN=your_telegram_bot_token_from_BotFather
TELEGRAM_LOGS_CHANNEL_ID=-100xxxxxxxxx
TELEGRAM_TRADES_CHANNEL_ID=-100yyyyyyyyy
TELEGRAM_ALLOWED_USER_ID=your_telegram_numeric_user_id

# Mode control
LIVE_TRADING=true           # Set false to disable real order placement
PAPER_TRADING=true          # Set true to run paper trading in parallel (or standalone)

# Paper trading settings
PAPER_STARTING_BALANCE=10000.0    # Virtual USDC to start with
PAPER_DB_PATH=paper_trades.db     # SQLite file path

# Trading parameters
DAILY_LOSS_LIMIT_USD=100.0
BASE_SIZE=24
COST_PER_PAIR_MAX=1.0
MAX_BUYS_PER_TICK=2
COOLDOWN_SECS=1
SIZE_REDUCE_AFTER_SECS=240
SIZE_MIN_RATIO=0.5
SIZE_MIN_SHARES=6
TREND_WINDOW=5
MARKET_POLL_INTERVAL=15
CLOSURE_CHECK_INTERVAL=20
LOG_FILE=bot.log
```

---

## config.py

```python
import os
from dotenv import load_dotenv
load_dotenv()

CLOB_API_BASE: str = "https://clob.polymarket.com"
GAMMA_API_BASE: str = "https://gamma-api.polymarket.com"
POLYMARKET_PRIVATE_KEY: str = os.getenv("POLYMARKET_PRIVATE_KEY", "")
POLYMARKET_WALLET_ADDRESS: str = os.getenv("POLYMARKET_WALLET_ADDRESS", "")
POLYMARKET_API_KEY: str = os.getenv("POLYMARKET_API_KEY", "")
POLYMARKET_API_SECRET: str = os.getenv("POLYMARKET_API_SECRET", "")
POLYMARKET_API_PASSPHRASE: str = os.getenv("POLYMARKET_API_PASSPHRASE", "")

TELEGRAM_BOT_TOKEN: str = os.getenv("TELEGRAM_BOT_TOKEN", "")
TELEGRAM_LOGS_CHANNEL_ID: int = int(os.getenv("TELEGRAM_LOGS_CHANNEL_ID", 0))
TELEGRAM_TRADES_CHANNEL_ID: int = int(os.getenv("TELEGRAM_TRADES_CHANNEL_ID", 0))
TELEGRAM_ALLOWED_USER_ID: int = int(os.getenv("TELEGRAM_ALLOWED_USER_ID", 0))

LIVE_TRADING: bool = os.getenv("LIVE_TRADING", "true").lower() == "true"
PAPER_TRADING: bool = os.getenv("PAPER_TRADING", "true").lower() == "true"
PAPER_STARTING_BALANCE: float = float(os.getenv("PAPER_STARTING_BALANCE", 10000.0))
PAPER_DB_PATH: str = os.getenv("PAPER_DB_PATH", "paper_trades.db")

BASE_SIZE: float = float(os.getenv("BASE_SIZE", 24))
COST_PER_PAIR_MAX: float = float(os.getenv("COST_PER_PAIR_MAX", 1.0))
MAX_BUYS_PER_TICK: int = int(os.getenv("MAX_BUYS_PER_TICK", 2))
COOLDOWN_SECS: int = int(os.getenv("COOLDOWN_SECS", 1))
SIZE_REDUCE_AFTER_SECS: int = int(os.getenv("SIZE_REDUCE_AFTER_SECS", 240))
SIZE_MIN_RATIO: float = float(os.getenv("SIZE_MIN_RATIO", 0.5))
SIZE_MIN_SHARES: float = float(os.getenv("SIZE_MIN_SHARES", 6))
TREND_WINDOW: int = int(os.getenv("TREND_WINDOW", 5))
MARKET_POLL_INTERVAL: int = int(os.getenv("MARKET_POLL_INTERVAL", 15))
CLOSURE_CHECK_INTERVAL: int = int(os.getenv("CLOSURE_CHECK_INTERVAL", 20))
LOG_FILE: str = os.getenv("LOG_FILE", "bot.log")
DAILY_LOSS_LIMIT_USD: float = float(os.getenv("DAILY_LOSS_LIMIT_USD", 100.0))
```

---

## STRATEGY LOGIC — (identical to v3, reproduced for completeness)

### strategy/position.py

```python
from dataclasses import dataclass, field
from typing import List
import time

@dataclass
class OpenOrder:
    order_id: str
    side: str           # "UP" or "DOWN"
    shares: float
    price: float
    placed_at: float    # unix timestamp

@dataclass
class Position:
    market_id: str
    question: str = ""
    up_shares: float = 0.0
    up_total_cost: float = 0.0
    down_shares: float = 0.0
    down_total_cost: float = 0.0
    trades: List[dict] = field(default_factory=list)
    open_orders: List[OpenOrder] = field(default_factory=list)

    @property
    def up_avg_price(self) -> float:
        return self.up_total_cost / self.up_shares if self.up_shares > 0 else 0.0

    @property
    def down_avg_price(self) -> float:
        return self.down_total_cost / self.down_shares if self.down_shares > 0 else 0.0

    @property
    def total_cost(self) -> float:
        return self.up_total_cost + self.down_total_cost

    def pnl_if_up_wins(self) -> float:
        return self.up_shares * 1.0 - self.total_cost

    def pnl_if_down_wins(self) -> float:
        return self.down_shares * 1.0 - self.total_cost

    def unrealized_pnl(self, current_up_price: float, current_down_price: float) -> float:
        return (self.up_shares * current_up_price +
                self.down_shares * current_down_price - self.total_cost)

    def cost_per_pair_if_add_up(self, n_shares: float, up_ask: float) -> float:
        new_up = self.up_shares + n_shares
        new_up_cost = self.up_total_cost + n_shares * up_ask
        new_pairs = min(new_up, self.down_shares)
        if new_pairs <= 0:
            return 0.0
        return (new_up_cost + self.down_total_cost) / new_pairs

    def cost_per_pair_if_add_down(self, n_shares: float, down_ask: float) -> float:
        new_down = self.down_shares + n_shares
        new_down_cost = self.down_total_cost + n_shares * down_ask
        new_pairs = min(self.up_shares, new_down)
        if new_pairs <= 0:
            return 0.0
        return (self.up_total_cost + new_down_cost) / new_pairs

    def apply_buy_up(self, n_shares: float, price: float, order_id: str = ""):
        self.up_shares += n_shares
        self.up_total_cost += n_shares * price
        self.trades.append({"side": "UP", "shares": n_shares, "price": price,
                            "order_id": order_id, "timestamp": time.time()})

    def apply_buy_down(self, n_shares: float, price: float, order_id: str = ""):
        self.down_shares += n_shares
        self.down_total_cost += n_shares * price
        self.trades.append({"side": "DOWN", "shares": n_shares, "price": price,
                            "order_id": order_id, "timestamp": time.time()})

    def add_open_order(self, order: OpenOrder): ...
    def remove_open_order(self, order_id: str): ...
    def get_all_order_ids(self) -> List[str]: ...
```

### strategy/trend.py

```python
def detect_trend(price_history: list) -> str:
    """
    'Rising' | 'Falling' | 'Flat'.
    Requires >= 3 data points (oldest first).
    Rising: most_recent > oldest AND up_moves > down_moves.
    """

def detect_up_trend(up_history: list) -> str: ...
def detect_down_trend(down_history: list) -> str: ...
```

### strategy/decision.py

```python
@dataclass
class TradeDecision:
    action: str    # "BUY_UP" | "BUY_DOWN" | "HOLD"
    shares: float
    price: float
    reason: str
    rule: str      # "rule1" | "rule2_lock" | "rule2_expansion" | "rule3_lock" |
                   # "rule3_expansion" | "rule4_lock_down" | "rule4_lock_up" |
                   # "rule4_expansion" | "hold"

def make_decision(position, up_ask, down_ask, up_history, down_history,
                  time_remaining_secs, base_size) -> TradeDecision:
    """
    Rule 0: size reduction near end
    Rule 1: no position → follow trend
    Rule 2: up only → lock or expand
    Rule 3: down only → lock or expand
    Rule 4: both sides → rebalance
    Rule 5: HOLD fallback
    (Full logic identical to v3 spec)
    """
```

---

## PAPER TRADING SHIMMER LAYER

### Architecture Overview

```
Real CLOB API (live prices) ──────┐
                                  ├──► PaperCLOBClient ──► PaperExecutor ──► PaperStateStore
Strategy decision engine ─────────┘         │                                      │
(make_decision — shared)                    │                               PaperAnalytics
                                         PaperDB (SQLite)◄─────────────────────────┘
                                            │
                                     Telegram notifier
                                    (🧪 [PAPER] tagged)
```

**Key principle:** Price data always comes from the real CLOB API. Only order placement is simulated. The strategy engine (`make_decision`) is shared between live and paper modes — it receives the same prices and makes the same decisions.

---

### paper_trading/paper_clob.py — The Shimmer Client

```python
import uuid
import time
from api.clob_client import CLOBClient

class PaperCLOBClient:
    """
    Drop-in replacement for CLOBClient for order operations.
    Price reading methods (get_best_ask, get_order_book) are DELEGATED to
    the real CLOBClient so paper trading uses live market prices.
    Order writing methods (place_order, cancel_order, cancel_all_orders)
    are SIMULATED — no real API calls, no blockchain transactions.
    """

    def __init__(self, real_clob: CLOBClient, paper_store: "PaperStateStore"):
        self._real = real_clob
        self._paper_store = paper_store
        self._open_orders: dict = {}  # order_id -> order dict

    # ── Price methods — delegate to real CLOB ─────────────────────────────────
    def get_best_ask(self, token_id: str) -> float:
        """Delegate to real CLOB. Returns live ask price."""
        return self._real.get_best_ask(token_id)

    def get_order_book(self, token_id: str) -> dict:
        """Delegate to real CLOB."""
        return self._real.get_order_book(token_id)

    def get_market(self, market_id: str) -> dict:
        """Delegate to real CLOB."""
        return self._real.get_market(market_id)

    def get_markets(self, params: dict) -> list:
        """Delegate to real CLOB."""
        return self._real.get_markets(params)

    # ── Order methods — fully simulated ──────────────────────────────────────
    def place_order(self, token_id: str, side: str, size: float, price: float) -> dict:
        """
        Simulate an instant fill at the given price.
        1. Check paper_store virtual balance >= size * price. If not, raise
           Exception("Paper: Insufficient virtual balance") so the executor
           handles it as a normal failure (no crash).
        2. Deduct size * price from virtual balance.
        3. Generate a fake order ID: f"PAPER-{uuid.uuid4().hex[:12].upper()}"
        4. Store the order in self._open_orders dict.
        5. Return a dict mimicking the real CLOB response:
           {
             "orderID": fake_order_id,
             "status": "MATCHED",         # paper fills instantly
             "transactedAt": str(int(time.time())),
             "paper": True
           }
        No sleep, no HTTP call.
        """

    def cancel_order(self, order_id: str) -> bool:
        """
        Remove from self._open_orders and refund the held USDC.
        Return True always (paper cancels never fail).
        """

    def cancel_all_orders(self) -> dict:
        """
        Cancel all tracked paper orders. Refund held USDC for each.
        Clear self._open_orders.
        Return {"cancelled": count}.
        """

    def get_open_orders(self) -> list:
        """Return list of all paper open orders."""
        return list(self._open_orders.values())

    def get_wallet_balance(self) -> dict:
        """Return paper virtual balance: {"balance": paper_store.virtual_balance}"""
        return {"balance": self._paper_store.get_virtual_balance()}
```

---

### paper_trading/paper_store.py — Paper State Store

```python
import threading
import time
from collections import deque
from typing import Dict, List, Optional
from strategy.position import Position

class PaperStateStore:
    """
    Separate thread-safe state for paper trading.
    Mirrors StateStore structure but is fully independent — paper positions
    never pollute live positions and vice versa.
    """

    def __init__(self, trend_window: int, starting_balance: float):
        self._lock = threading.RLock()
        self._trend_window = trend_window

        # Paper positions (market_id → Position)
        self._positions: Dict[str, Position] = {}
        self._price_history: Dict[str, Dict[str, deque]] = {}
        self._market_meta: Dict[str, dict] = {}

        # Virtual wallet
        self._virtual_balance: float = starting_balance
        self._starting_balance: float = starting_balance

        # Paper accounting
        self._paper_trade_count: int = 0
        self._paper_usdc_spent: float = 0.0
        self._paper_realized_pnl: float = 0.0
        self._start_time: float = time.time()

        # Closed market results for analytics
        self._closed_markets: List[dict] = []  # List of result dicts

    def get_virtual_balance(self) -> float: ...
    def deduct_balance(self, amount: float): ...
    def credit_balance(self, amount: float): ...

    def get_position(self, market_id: str) -> Position: ...
    def update_position(self, market_id: str, position: Position): ...
    def remove_market(self, market_id: str): ...
    def list_active_markets(self) -> List[str]: ...

    def append_price(self, market_id: str, side: str, price: float): ...
    def get_price_history(self, market_id: str, side: str) -> List[float]: ...

    def set_market_meta(self, market_id: str, market: dict): ...
    def get_market_meta(self, market_id: str) -> Optional[dict]: ...

    def record_paper_trade(self, market_id: str, side: str, shares: float,
                           price: float, rule: str, reason: str):
        """Increment paper_trade_count and paper_usdc_spent."""

    def record_closed_market(self, result: dict):
        """
        Append to self._closed_markets.
        result dict fields:
          market_id, question, winner, up_shares, down_shares,
          total_cost, pnl, resolved_at (unix ts), trades (list)
        Also update self._paper_realized_pnl and credit_balance(max(0, pnl)).
        """

    def get_closed_markets(self) -> List[dict]:
        """Return copy of _closed_markets list."""

    def get_paper_stats(self) -> dict:
        """
        Returns:
        {
          "virtual_balance": float,
          "starting_balance": float,
          "paper_trade_count": int,
          "paper_usdc_spent": float,
          "paper_realized_pnl": float,
          "active_market_count": int,
          "closed_market_count": int,
          "start_time": float,
          "uptime_secs": float
        }
        """

    def reset(self, starting_balance: float):
        """
        Full reset of paper trading state.
        Clears all positions, history, closed_markets, resets balance and counters.
        """
```

---

### paper_trading/paper_db.py — SQLite Persistence

```python
import sqlite3
import json
import time
from typing import List, Optional

class PaperDB:
    """
    Persists paper trading history across bot restarts.
    All writes are atomic. Thread-safe via a single connection with WAL mode.

    Schema:
    ┌─────────────────────────────────────────────────────────────────────┐
    │ TABLE paper_sessions                                                 │
    │   id INTEGER PRIMARY KEY AUTOINCREMENT                              │
    │   started_at REAL, ended_at REAL, starting_balance REAL            │
    │   ending_balance REAL, total_realized_pnl REAL                     │
    │   trade_count INTEGER, notes TEXT                                   │
    ├─────────────────────────────────────────────────────────────────────┤
    │ TABLE paper_markets                                                  │
    │   id INTEGER PRIMARY KEY AUTOINCREMENT                              │
    │   session_id INTEGER (FK paper_sessions.id)                         │
    │   market_id TEXT, question TEXT, winner TEXT                        │
    │   up_shares REAL, down_shares REAL, total_cost REAL, pnl REAL      │
    │   resolved_at REAL, trade_count INTEGER                             │
    ├─────────────────────────────────────────────────────────────────────┤
    │ TABLE paper_trades                                                   │
    │   id INTEGER PRIMARY KEY AUTOINCREMENT                              │
    │   session_id INTEGER, market_id TEXT, side TEXT                     │
    │   shares REAL, price REAL, total_cost REAL                         │
    │   rule TEXT, reason TEXT, placed_at REAL                           │
    └─────────────────────────────────────────────────────────────────────┘
    """

    def __init__(self, db_path: str):
        self._db_path = db_path
        self._conn: Optional[sqlite3.Connection] = None
        self._session_id: Optional[int] = None
        self._connect()
        self._create_tables()

    def _connect(self):
        """Open connection, enable WAL mode for thread safety."""

    def _create_tables(self):
        """CREATE TABLE IF NOT EXISTS for all three tables."""

    def start_session(self, starting_balance: float) -> int:
        """
        Insert a new row into paper_sessions with started_at=now, starting_balance.
        Store and return the new session_id.
        """

    def end_session(self, ending_balance: float, total_pnl: float, trade_count: int):
        """
        UPDATE paper_sessions SET ended_at=now, ending_balance, total_realized_pnl,
        trade_count WHERE id = self._session_id.
        """

    def save_market_result(self, market_id: str, question: str, winner: str,
                           up_shares: float, down_shares: float, total_cost: float,
                           pnl: float, resolved_at: float, trade_count: int):
        """INSERT into paper_markets."""

    def save_trade(self, market_id: str, side: str, shares: float, price: float,
                   rule: str, reason: str):
        """INSERT into paper_trades with placed_at=now."""

    def get_all_market_results(self) -> List[dict]:
        """
        SELECT all rows from paper_markets (all sessions combined).
        Return as list of dicts.
        """

    def get_session_market_results(self, session_id: int = None) -> List[dict]:
        """
        If session_id is None, use self._session_id (current session).
        SELECT from paper_markets WHERE session_id = ?
        """

    def get_all_trades(self, session_id: int = None) -> List[dict]:
        """SELECT from paper_trades for a session (or current session)."""

    def get_sessions_summary(self) -> List[dict]:
        """SELECT all rows from paper_sessions ordered by started_at DESC."""
```

---

### paper_trading/paper_executor.py — Paper Fill Engine

```python
class PaperExecutor:
    """
    Executes trading decisions in paper mode.
    Mirrors the real Executor interface exactly so main.py can use either
    interchangeably depending on PAPER_TRADING flag.
    """

    def __init__(self, paper_clob: PaperCLOBClient, paper_store: PaperStateStore,
                 paper_db: PaperDB, notifier: TelegramNotifier):
        ...

    def execute(self, market: dict, decision: TradeDecision, position: Position) -> bool:
        """
        1. Log signal to logger (tagged [PAPER]).
        2. Send signal to Telegram #logs via notifier.send_paper_log().
        3. Call paper_clob.place_order(token_id, "BUY", decision.shares, decision.price).
        4. On success:
             a. Apply position update.
             b. paper_store.record_paper_trade(...).
             c. paper_db.save_trade(...).
             d. notifier.send_paper_trade(trade_data) → sends to #trades with 🧪 tag.
             e. Return True.
        5. On failure (e.g. insufficient virtual balance):
             a. Log "Paper: insufficient virtual balance, skipping".
             b. Return False.
        No retries needed — paper orders never fail from network issues.
        """

    def cancel_all_open_orders(self) -> int:
        """
        Call paper_clob.cancel_all_orders().
        Clear open_orders on all paper positions.
        Return count cancelled.
        """
```

---

### paper_trading/paper_analytics.py — Performance Engine

```python
from typing import List, Dict
import math

class PaperAnalytics:
    """
    Computes full strategy performance metrics from a list of closed market results.
    All methods are pure functions (no state) — pass in the results list.
    """

    @staticmethod
    def compute(results: List[dict]) -> dict:
        """
        Master method. Calls all sub-methods and returns a single analytics dict.

        Input: results = list of dicts, each with:
          market_id, question, winner, up_shares, down_shares,
          total_cost, pnl, resolved_at, trade_count, trades (list)

        Returns:
        {
          "total_markets":       int,    # markets fully resolved with positions
          "winning_markets":     int,    # markets where pnl > 0
          "losing_markets":      int,    # markets where pnl < 0
          "breakeven_markets":   int,    # markets where pnl == 0
          "win_rate_pct":        float,  # winning_markets / total_markets * 100
          "total_realized_pnl":  float,  # sum of all pnl values
          "total_usdc_spent":    float,  # sum of all total_cost values
          "avg_pnl_per_market":  float,
          "best_market_pnl":     float,
          "worst_market_pnl":    float,
          "best_market_name":    str,
          "worst_market_name":   str,
          "roi_pct":             float,  # total_realized_pnl / total_usdc_spent * 100
          "max_drawdown_pct":    float,  # max peak-to-trough equity drop in %
          "max_consecutive_wins":   int,
          "max_consecutive_losses": int,
          "sharpe_ratio":        float,  # approximation using per-market PnL as returns
          "avg_trades_per_market": float,
          "rule_breakdown":      dict,   # {rule_name: {count, total_pnl, win_rate}} per rule
          "pnl_by_winner":       dict,   # {"UP": total_pnl, "DOWN": total_pnl}
          "equity_curve":        list,   # list of cumulative PnL after each market, oldest first
        }
        """

    @staticmethod
    def _win_rate(results: list) -> tuple:
        """Return (wins, losses, breakeven, win_rate_pct)."""

    @staticmethod
    def _max_drawdown(equity_curve: list) -> float:
        """
        Given a list of cumulative PnL values (the equity curve),
        compute max drawdown as a percentage of the peak value.
        Formula: max((peak - trough) / peak) * 100 over all peaks.
        Return 0.0 if fewer than 2 data points.
        """

    @staticmethod
    def _sharpe_ratio(pnl_values: list, risk_free_rate: float = 0.0) -> float:
        """
        Approximate Sharpe ratio using per-market PnL as discrete returns.
        sharpe = (mean(pnl) - risk_free_rate) / std(pnl)
        Return 0.0 if std == 0 or fewer than 3 data points.
        Use population std (ddof=0).
        """

    @staticmethod
    def _max_consecutive(results: list) -> tuple:
        """Return (max_consecutive_wins, max_consecutive_losses)."""

    @staticmethod
    def _rule_breakdown(results: list) -> dict:
        """
        For each trade in each market's trade list, group by rule.
        For each rule, compute:
          count: total trades using that rule
          total_pnl: sum of the parent market's pnl for markets where this rule was used
                     (attribute the market's full pnl to each rule it contained)
          win_rate: % of markets using this rule that were profitable
        Return: {"rule1": {"count": int, "total_pnl": float, "win_rate": float}, ...}
        """

    @staticmethod
    def _equity_curve(results: list) -> list:
        """
        Sort results by resolved_at (ascending).
        Compute cumulative sum of pnl values.
        Return list of cumulative PnL after each market.
        """

    @staticmethod
    def format_report(analytics: dict, virtual_balance: float,
                      starting_balance: float) -> str:
        """
        Format the analytics dict into a multi-line HTML Telegram message.

        Format:
        ═══════════════════════════════
        🧪 <b>PAPER TRADING REPORT</b>
        ═══════════════════════════════
        📊 <b>OVERVIEW</b>
        Markets played:   {total_markets}
        ✅ Wins:          {winning_markets} ({win_rate_pct:.1f}%)
        ❌ Losses:        {losing_markets}
        ➖ Breakeven:     {breakeven_markets}
        ───────────────────────────────
        💰 <b>PnL SUMMARY</b>
        Total realized:   {+$X.XX or -$X.XX} USDC
        Total spent:      ${total_usdc_spent:.2f} USDC
        ROI:              {roi_pct:+.2f}%
        Avg per market:   {avg_pnl_per_market:+.2f} USDC
        ───────────────────────────────
        🏆 Best market:   +${best_market_pnl:.2f} ({best_market_name})
        💀 Worst market:  {worst_market_pnl:+.2f} ({worst_market_name})
        ───────────────────────────────
        📈 <b>RISK METRICS</b>
        Sharpe ratio:     {sharpe_ratio:.3f}
        Max drawdown:     {max_drawdown_pct:.2f}%
        Max consec. wins: {max_consecutive_wins}
        Max consec. loss: {max_consecutive_losses}
        ───────────────────────────────
        🎯 <b>RULE BREAKDOWN</b>
        {for each rule: "rule_name: N trades | PnL: $X.XX | WR: X.X%"}
        ───────────────────────────────
        💼 <b>WALLET</b>
        Starting balance: ${starting_balance:.2f}
        Current balance:  ${virtual_balance:.2f}
        Net change:       {virtual_balance - starting_balance:+.2f} USDC
        ═══════════════════════════════
        (Truncate to 4096 chars if needed)
        """
```

---

## LIVE API CLIENT (api/auth.py + api/clob_client.py)

### api/auth.py

```python
class PolyAuth:
    def __init__(self, private_key: str, api_key: str, api_secret: str,
                 passphrase: str, wallet_address: str): ...

    def get_auth_headers(self, method: str, path: str, body: str = "") -> dict:
        """HMAC-SHA256 auth headers: POLY_ADDRESS, POLY_SIGNATURE, POLY_TIMESTAMP,
        POLY_NONCE, POLY_API_KEY, POLY_PASSPHRASE"""

    def sign_order(self, order_data: dict) -> str:
        """Sign with wallet private key via eth_account. Return hex signature."""

    def build_order(self, token_id: str, side: str, size: float,
                    price: float, expiration: int = 0) -> dict:
        """Build full signed order dict with makerAmount/takerAmount in micro-units."""
```

### api/clob_client.py

```python
class CLOBClient:
    def __init__(self, auth: PolyAuth): ...
    def get_best_ask(self, token_id: str) -> float: ...
    def get_order_book(self, token_id: str) -> dict: ...
    def place_order(self, token_id: str, side: str, size: float, price: float) -> dict: ...
    def cancel_order(self, order_id: str) -> bool: ...
    def cancel_all_orders(self) -> dict: ...
    def get_open_orders(self) -> list: ...
    def get_wallet_balance(self) -> dict: ...
    def get_market(self, market_id: str) -> dict: ...
    def get_markets(self, params: dict) -> list: ...
```

(Full implementations required for all methods — same as v3 spec.)

---

## LIVE STATE STORE (state/store.py)

Same as v3 spec — full implementation required with:
- `_positions`, `_price_history`, `_market_meta`
- `_panic_mode`, `_trading_halted`
- `_trade_count`, `_usdc_spent_today`, `_daily_realized_pnl`
- All methods: `get_position`, `update_position`, `remove_market`, `list_active_markets`,
  `append_price`, `get_price_history`, `set_market_meta`, `get_market_meta`,
  `increment_trade_count`, `add_usdc_spent`, `add_realized_pnl`, `get_stats`,
  `set_panic_mode`, `is_panic_mode`, `set_trading_halted`, `is_trading_halted`, `should_trade`

---

## LIVE EXECUTOR (trader/executor.py)

Same as v3 spec — full implementation. Checks `store.should_trade()` on every call.

---

## MONITOR (monitor/)

### market_finder.py
Same as v3. When PAPER_TRADING=True, also register new markets in paper_store via `paper_store.set_market_meta()`.

### closure_checker.py
Extend v3 spec with paper settlement:

```python
class ClosureChecker:
    def __init__(self, clob: CLOBClient, live_store: StateStore,
                 paper_store: Optional[PaperStateStore],
                 paper_db: Optional[PaperDB],
                 notifier: TelegramNotifier,
                 live_trading: bool, paper_trading: bool): ...

    def check_and_record(self, market_id: str):
        """
        GET /markets/{market_id}.
        If closed:
          1. Determine winner.
          2. If live_trading:
               Live settlement: compute live pnl, store.add_realized_pnl(pnl),
               check daily loss limit, notifier.send_market_closed().
               live_store.remove_market(market_id).
          3. If paper_trading and market in paper_store:
               Paper settlement:
                 a. Get paper position = paper_store.get_position(market_id).
                 b. If no position, skip.
                 c. Compute paper pnl (same formula).
                 d. paper_store.record_closed_market({...}).
                 e. paper_db.save_market_result(...).
                 f. notifier.send_paper_market_closed(market_id, question, winner,
                                                      paper_pnl, ...).
                 g. paper_store.remove_market(market_id).
          4. Log both results.
        """
```

---

## TELEGRAM BOT (telegram_bot/)

### telegram_bot/notifier.py

Add these new methods to the TelegramNotifier class (all v3 methods still required):

```python
def send_paper_log(self, message: str, level: str = "info"):
    """
    Send to #logs channel, prefixed with 🧪 [PAPER].
    Otherwise identical to send_log().
    """

def send_paper_trade(self, trade_data: dict):
    """
    Send to #trades channel with 🧪 [PAPER] tag.
    trade_data: same fields as send_trade() with paper=True.

    Format:
    ─────────────────────────
    🧪 <b>[PAPER] SIMULATED TRADE</b>
    ─────────────────────────
    📌 <b>Market:</b> {question}
    🆔 <b>Order ID:</b> {order_id}  ← (PAPER-xxxx fake ID)
    📊 <b>Shares:</b> {shares} @ ${price}
    💵 <b>Virtual spent:</b> ${total_spent} USDC
    💼 <b>Virtual balance:</b> ${virtual_balance_after} USDC
    📐 <b>Rule:</b> {rule} — {reason}
    ─────────────────────────
    📈 PnL if Up wins:   {pnl_if_up_wins:+.2f}
    📉 PnL if Down wins: {pnl_if_down_wins:+.2f}
    🕐 {timestamp}
    ─────────────────────────
    """

def send_paper_market_closed(self, market_id: str, question: str, winner: str,
                              pnl: float, up_shares: float, down_shares: float,
                              total_cost: float, virtual_balance: float):
    """
    Send to #trades. Same format as send_market_closed() but with 🧪 [PAPER] header.
    Also show virtual_balance after settlement.
    """

def send_paper_report(self, report_text: str):
    """
    Send the full analytics report (from PaperAnalytics.format_report())
    to the #logs channel. If over 4096 chars, split into multiple messages.
    """
```

### telegram_bot/dashboard.py

Add these commands (all v3 commands still required):

```python
# ── /paper_status ─────────────────────────────────────────────────────────
async def cmd_paper_status(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    /paper_status — Quick paper trading summary.
    Requires paper_trading to be enabled.

    stats = paper_store.get_paper_stats()
    analytics = PaperAnalytics.compute(paper_store.get_closed_markets())

    Format (HTML):
    ─────────────────────────
    🧪 <b>PAPER TRADING STATUS</b>
    ─────────────────────────
    💼 Virtual balance:   ${virtual_balance:.2f} USDC
    📦 Active markets:    {active_market_count}
    🔄 Total trades:      {paper_trade_count}
    💵 Total spent:       ${paper_usdc_spent:.2f}
    ─────────────────────────
    ✅ Markets won:       {winning_markets} ({win_rate_pct:.1f}%)
    ❌ Markets lost:      {losing_markets}
    💰 Realized PnL:      {total_realized_pnl:+.2f} USDC
    📈 ROI:               {roi_pct:+.2f}%
    ─────────────────────────
    If no closed markets: show "No markets resolved yet."
    """

# ── /paper_report ──────────────────────────────────────────────────────────
async def cmd_paper_report(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    /paper_report — Full strategy performance analytics report.

    1. Get all closed markets from paper_store (in-memory) + paper_db (historical).
    2. Merge them (deduplicate by market_id).
    3. Compute analytics via PaperAnalytics.compute(results).
    4. Format via PaperAnalytics.format_report(analytics, virtual_balance, starting_balance).
    5. Send the formatted report via notifier.send_paper_report(report_text).
    6. Reply: "📊 Full report sent to #logs channel."

    If no markets resolved yet: reply "No closed markets to report on yet."
    """

# ── /paper_positions ───────────────────────────────────────────────────────
async def cmd_paper_positions(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    /paper_positions — Show all active paper positions with live prices and PnL.

    For each market in paper_store.list_active_markets():
      1. Get paper position.
      2. Fetch live up_ask and down_ask from real CLOB.
      3. Compute unrealized_pnl = position.unrealized_pnl(up_ask, down_ask).
      4. Get time_remaining.

    Format per market (HTML):
    ─────────────────────────
    🧪 <b>[PAPER] {question}</b>
    🆔 {market_id[:12]}...
    ─────────────────────────
    🟢 <b>UP:</b>   {up_shares:.1f} @ avg ${up_avg:.3f} | cost ${up_total_cost:.2f}
    🔴 <b>DOWN:</b> {down_shares:.1f} @ avg ${down_avg:.3f} | cost ${down_total_cost:.2f}
    ─────────────────────────
    💵 Total cost: ${total_cost:.2f}
    📈 Live UP ask:  ${up_ask:.3f}
    📉 Live DOWN ask: ${down_ask:.3f}
    💹 Unrealized PnL: {unrealized_pnl:+.2f} USDC
    ⏳ Time remaining: {Xm Ys}
    ─────────────────────────

    If no paper positions: reply "No active paper positions."
    """

# ── /paper_reset ───────────────────────────────────────────────────────────
async def cmd_paper_reset(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    /paper_reset — Reset paper trading state.

    1. Call paper_db.end_session(current_balance, total_pnl, trade_count)
       to archive the current session.
    2. Call paper_db.start_session(PAPER_STARTING_BALANCE) for a new session.
    3. Call paper_store.reset(PAPER_STARTING_BALANCE) to clear all in-memory state.
    4. Send to #logs: "🧪 Paper trading reset. New virtual balance: ${PAPER_STARTING_BALANCE:.2f}"
    5. Reply:
       "🔄 <b>Paper trading reset.</b>
        Previous session archived to SQLite.
        New virtual balance: ${PAPER_STARTING_BALANCE:.2f} USDC
        All paper positions and history cleared."
    """

# ── /paper_history ─────────────────────────────────────────────────────────
async def cmd_paper_history(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """
    /paper_history — Show last 10 closed paper markets from SQLite.

    results = paper_db.get_session_market_results()  ← current session
    Show last 10 sorted by resolved_at descending.

    For each result:
    {i}. {question[:40]} | {winner} won | PnL: {pnl:+.2f} USDC

    Format (HTML):
    🧪 <b>PAPER HISTORY (last 10)</b>
    ─────────────────────────
    1. BTC 5m Up/Down | UP won | <b>+$3.12</b>
    2. BTC 5m Up/Down | DOWN won | <b>-$1.44</b>
    ...
    ─────────────────────────
    Session total: {sum_pnl:+.2f} USDC

    If empty: reply "No closed paper markets in this session."
    """

# Also update /help to include all paper commands:
# /paper_status   — Quick paper trading summary
# /paper_report   — Full strategy performance report
# /paper_positions — Active paper positions with live PnL
# /paper_history  — Last 10 closed paper markets
# /paper_reset    — Reset paper state and archive session
```

### telegram_bot/bot.py

Register ALL command handlers including paper commands:

```python
handlers = [
    # Live commands
    ("positions",      dashboard.cmd_positions),
    ("wallet",         dashboard.cmd_wallet),
    ("panic",          dashboard.cmd_panic),
    ("resume",         dashboard.cmd_resume),
    ("status",         dashboard.cmd_status),
    ("pnl",            dashboard.cmd_pnl),
    ("stop",           dashboard.cmd_stop),
    ("help",           dashboard.cmd_help),
    # Paper trading commands
    ("paper_status",   dashboard.cmd_paper_status),
    ("paper_report",   dashboard.cmd_paper_report),
    ("paper_positions",dashboard.cmd_paper_positions),
    ("paper_history",  dashboard.cmd_paper_history),
    ("paper_reset",    dashboard.cmd_paper_reset),
]
```

---

## MAIN LOOP (main.py)

```python
"""
main.py — Entry point

Startup:
1. Load and validate all env vars. Rules:
   - If LIVE_TRADING=true: require POLYMARKET_PRIVATE_KEY, WALLET_ADDRESS, API creds.
   - If LIVE_TRADING=false: skip wallet validation (paper-only mode).
   - Always require TELEGRAM_* vars.

2. Initialize:
   a. StateStore(TREND_WINDOW)                           ← live state (always)
   b. PaperStateStore(TREND_WINDOW, PAPER_STARTING_BALANCE)  ← paper state
   c. TelegramNotifier
   d. paper_db = PaperDB(PAPER_DB_PATH)
      paper_db.start_session(PAPER_STARTING_BALANCE)
   e. PolyAuth (if LIVE_TRADING)
   f. real_clob = CLOBClient(auth)  (if LIVE_TRADING, else use a read-only CLOBClient
                                     with no auth — only get_best_ask, get_market work)
   g. paper_clob = PaperCLOBClient(real_clob, paper_store)  (if PAPER_TRADING)
   h. live_executor = Executor(real_clob, live_store, notifier)  (if LIVE_TRADING)
   i. paper_executor = PaperExecutor(paper_clob, paper_store, paper_db, notifier)  (if PAPER_TRADING)
   j. market_finder = MarketFinder(real_clob, live_store, paper_store, notifier,
                                   LIVE_TRADING, PAPER_TRADING)
   k. closure_checker = ClosureChecker(real_clob, live_store, paper_store, paper_db,
                                       notifier, LIVE_TRADING, PAPER_TRADING)
   l. stop_event = threading.Event()
   m. dashboard = Dashboard(live_store, paper_store, real_clob, paper_clob,
                             live_executor, paper_executor, paper_db,
                             notifier, stop_event, TELEGRAM_ALLOWED_USER_ID,
                             PAPER_STARTING_BALANCE)
   n. TelegramBotRunner(TELEGRAM_BOT_TOKEN, dashboard).start()

3. Startup Telegram message:
   "🚀 Bot started
    Mode: {'LIVE + PAPER' if both else 'LIVE ONLY' if live else 'PAPER ONLY'}
    {'Wallet: ' + addr[:10] + '...' if LIVE_TRADING else 'No wallet (paper only)'}
    Paper balance: ${PAPER_STARTING_BALANCE:.2f} USDC
    Size: {BASE_SIZE} | CostMax: {COST_PER_PAIR_MAX}"

4. Background threads:
   a. market_discovery_thread: runs market_finder every MARKET_POLL_INTERVAL secs.
      Registers new markets in both live_store and paper_store if enabled.
   b. closure_check_thread: calls closure_checker.check_and_record(mid) for each
      market in live_store (markets are shared — same real markets for both modes).
   c. telegram_bot_thread: TelegramBotRunner.

5. Main trading loop:
   while not stop_event.is_set():
     market_ids = live_store.list_active_markets()  # same markets for both modes

     for market_id in market_ids:
       try:
         market = live_store.get_market_meta(market_id)
         if market is None:
             continue

         up_ask   = real_clob.get_best_ask(market["up_token_id"])
         down_ask = real_clob.get_best_ask(market["down_token_id"])

         # Update price history in BOTH stores (both use same live prices)
         live_store.append_price(market_id, "up", up_ask)
         live_store.append_price(market_id, "down", down_ask)
         if PAPER_TRADING:
             paper_store.append_price(market_id, "up", up_ask)
             paper_store.append_price(market_id, "down", down_ask)

         up_history   = live_store.get_price_history(market_id, "up")
         down_history = live_store.get_price_history(market_id, "down")

         time_remaining = MarketFinder.get_time_remaining(market)
         if time_remaining <= 0:
             continue

         # ── LIVE TRADING ──────────────────────────────────
         if LIVE_TRADING and live_store.should_trade():
             live_pos = live_store.get_position(market_id)
             decision = make_decision(live_pos, up_ask, down_ask, up_history,
                                      down_history, time_remaining, BASE_SIZE)
             if decision.action != "HOLD":
                 for _ in range(MAX_BUYS_PER_TICK):
                     if not live_store.should_trade():
                         break
                     ok = live_executor.execute(market, decision, live_pos)
                     if not ok:
                         break
                     live_pos = live_store.get_position(market_id)
                     new_dec = make_decision(live_pos, up_ask, down_ask, up_history,
                                             down_history, time_remaining, BASE_SIZE)
                     if new_dec.action != decision.action:
                         break
                     decision = new_dec

         # ── PAPER TRADING ─────────────────────────────────
         if PAPER_TRADING:
             paper_pos = paper_store.get_position(market_id)
             paper_decision = make_decision(paper_pos, up_ask, down_ask, up_history,
                                            down_history, time_remaining, BASE_SIZE)
             if paper_decision.action != "HOLD":
                 for _ in range(MAX_BUYS_PER_TICK):
                     ok = paper_executor.execute(market, paper_decision, paper_pos)
                     if not ok:
                         break
                     paper_pos = paper_store.get_position(market_id)
                     new_pdec = make_decision(paper_pos, up_ask, down_ask, up_history,
                                              down_history, time_remaining, BASE_SIZE)
                     if new_pdec.action != paper_decision.action:
                         break
                     paper_decision = new_pdec

       except Exception as e:
           logger.error(f"[{market_id}] Tick error: {e}")
           notifier.send_error(f"Market {market_id}", str(e))

     time.sleep(max(COOLDOWN_SECS, 1))

6. On shutdown:
   paper_db.end_session(
       paper_store.get_virtual_balance(),
       paper_store.get_paper_stats()["paper_realized_pnl"],
       paper_store.get_paper_stats()["paper_trade_count"]
   )
   notifier.send_log("🛑 Bot stopped. Paper session archived.", level="warn")
"""
```

---

## PAPER TRADING MODES MATRIX

| Config | Behavior |
|---|---|
| `LIVE_TRADING=true, PAPER_TRADING=false` | Live only. No paper state. |
| `LIVE_TRADING=false, PAPER_TRADING=true` | Paper only. No wallet needed. Reads live prices, simulates fills. |
| `LIVE_TRADING=true, PAPER_TRADING=true` | Both run in parallel. Same markets, same decisions, separate state. Use to compare paper vs live results. |
| `LIVE_TRADING=false, PAPER_TRADING=false` | Invalid. Exit with error on startup. |

---

## IMPORTANT IMPLEMENTATION NOTES

### Paper Trading Layer
1. **PaperCLOBClient always delegates `get_best_ask` and `get_order_book` to the real CLOBClient.** Paper trading uses real live prices — this is the entire point of the shimmer architecture.
2. **`place_order` in PaperCLOBClient assumes instant fill at the ask price.** This is a deliberate simplification — real orders may not fill instantly if the orderbook is thin. The paper results will be slightly optimistic as a result.
3. **Paper positions are fully separate from live positions.** `paper_store` and `live_store` are distinct objects. The same `market_id` can have a live position AND a paper position simultaneously.
4. **SQLite persistence:** `PaperDB` saves every paper trade and every closed market. When the bot restarts, `paper_db.get_session_market_results()` lets `PaperAnalytics.compute()` include historical data from the current session even after restart.
5. **Virtual balance starts at `PAPER_STARTING_BALANCE`.** After each simulated fill, `paper_clob.place_order()` deducts `shares * price` from the virtual balance. When a market closes and the paper position wins, `paper_store.record_closed_market()` credits `pnl` back to the virtual balance (if positive).
6. **`/paper_report` merges in-memory + SQLite** so the analytics are complete even after a restart.
7. **Paper Telegram messages are always prefixed with 🧪 [PAPER]** so they are visually distinct from live trade messages even in the same channels.

### Shared Strategy Engine
8. **`make_decision()` is shared between live and paper modes.** It is a pure function — it takes prices and position as input, returns a decision. It has no side effects and no knowledge of whether it is running in live or paper mode. This guarantees the paper mode tests exactly the same strategy as live mode.

### Mode Control
9. **Paper-only mode (`LIVE_TRADING=false`):** The bot still creates a `CLOBClient` for price reading, but skips `PolyAuth` initialization and never calls `place_order` on the real client.
10. **`/paper_reset` archives the old session to SQLite and starts a fresh one.** Historical sessions are preserved and can be queried.

### General
11. **All Telegram messages use HTML parse mode.**
12. **StateStore and PaperStateStore use `threading.RLock`.**
13. **TelegramNotifier uses a dedicated asyncio event loop** in a background thread for all `send_*` calls.

---

## OUTPUT FORMAT

Generate each file completely. Present in this order:

1. `requirements.txt`
2. `.env.example`
3. `config.py`
4. `utils/logger.py`
5. `state/store.py`
6. `strategy/position.py`
7. `strategy/trend.py`
8. `strategy/decision.py`
9. `api/auth.py`
10. `api/clob_client.py`
11. `paper_trading/paper_db.py`
12. `paper_trading/paper_store.py`
13. `paper_trading/paper_clob.py`
14. `paper_trading/paper_executor.py`
15. `paper_trading/paper_analytics.py`
16. `monitor/market_finder.py`
17. `monitor/closure_checker.py`
18. `trader/executor.py`
19. `telegram_bot/notifier.py`
20. `telegram_bot/dashboard.py`
21. `telegram_bot/bot.py`
22. `main.py`
23. All `__init__.py` files (group together)

For each file:
```
## File: polymarket_bot/filename.py
```
followed by a complete fenced Python code block.

Do not summarize. Do not skip any file. Do not write placeholder functions. Every method body must be fully implemented.

