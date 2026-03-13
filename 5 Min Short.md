You are an expert Python developer. Generate a complete production-ready Polymarket trading bot. All files fully implemented. Follow every requirement exactly.

---

## PROJECT OVERVIEW

The bot: (1) connects to Polymarket CLOB API, (2) discovers active BTC 5-minute markets, (3) runs a dynamic one-side-at-a-time rebalancing strategy, (4) places real signed orders via wallet private key, (5) tracks positions/PnL, (6) detects market closure, (7) sends activity to Telegram `#logs` and `#trades` channels, (8) provides 13 Telegram commands, (9) enforces daily loss limit + panic/resume system, (10) runs a **Paper Trading Shimmer** — full simulation using live prices, virtual balance, SQLite persistence, and strategy analytics.

---

## FILE STRUCTURE

```
polymarket_bot/
├── main.py
├── config.py
├── api/
│   ├── __init__.py
│   ├── clob_client.py       # Live CLOB REST (orders, book, wallet, cancel)
│   └── auth.py              # Wallet ECDSA + L2 HMAC auth headers
├── strategy/
│   ├── __init__.py
│   ├── position.py          # Position + OpenOrder dataclasses
│   ├── trend.py             # detect_trend / detect_up_trend / detect_down_trend
│   └── decision.py          # make_decision → TradeDecision
├── trader/
│   ├── __init__.py
│   └── executor.py          # Live order executor, panic/loss guards
├── paper_trading/
│   ├── __init__.py
│   ├── paper_clob.py        # Shim: delegates price reads, simulates order fills
│   ├── paper_store.py       # Paper StateStore: virtual balance, paper positions
│   ├── paper_executor.py    # Paper fill engine (same interface as live Executor)
│   ├── paper_analytics.py   # Win rate, PnL, Sharpe, drawdown, per-rule breakdown
│   └── paper_db.py          # SQLite: sessions, paper_markets, paper_trades tables
├── monitor/
│   ├── __init__.py
│   ├── market_finder.py
│   └── closure_checker.py   # Settles both live and paper positions on close
├── state/
│   ├── __init__.py
│   └── store.py             # Live thread-safe StateStore
├── telegram_bot/
│   ├── __init__.py
│   ├── bot.py               # Application runner (background thread)
│   ├── notifier.py          # send_log/trade/paper_trade/paper_report etc.
│   └── dashboard.py         # All 13 command handlers
├── utils/
│   ├── __init__.py
│   └── logger.py
├── requirements.txt
└── .env.example
```

---

## DEPENDENCIES

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
SQLite via Python stdlib — no extra package.

---

## .env.example

```
POLYMARKET_PRIVATE_KEY=hex_without_0x
POLYMARKET_WALLET_ADDRESS=0x...
POLYMARKET_API_KEY=...
POLYMARKET_API_SECRET=...
POLYMARKET_API_PASSPHRASE=...
TELEGRAM_BOT_TOKEN=...
TELEGRAM_LOGS_CHANNEL_ID=-100xxxxxxxxx
TELEGRAM_TRADES_CHANNEL_ID=-100yyyyyyyyy
TELEGRAM_ALLOWED_USER_ID=numeric_id
LIVE_TRADING=true
PAPER_TRADING=true
PAPER_STARTING_BALANCE=10000.0
PAPER_DB_PATH=paper_trades.db
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

Load every `.env` var above via `os.getenv`. Types: `LIVE_TRADING` and `PAPER_TRADING` are bools (`"true"` → `True`). `PAPER_STARTING_BALANCE`, `DAILY_LOSS_LIMIT_USD`, sizes are floats. Intervals are ints. Expose all as module-level constants.

---

## STRATEGY (shared by live and paper — pure functions, no side effects)

### strategy/position.py

```python
@dataclass
class OpenOrder:
    order_id: str; side: str; shares: float; price: float; placed_at: float

@dataclass
class Position:
    market_id: str; question: str = ""
    up_shares: float = 0.0;   up_total_cost: float = 0.0
    down_shares: float = 0.0; down_total_cost: float = 0.0
    trades: List[dict] = field(default_factory=list)
    open_orders: List[OpenOrder] = field(default_factory=list)

    # Properties: up_avg_price, down_avg_price, total_cost (derived from fields)
    def pnl_if_up_wins(self)   -> float: return self.up_shares   - self.total_cost
    def pnl_if_down_wins(self) -> float: return self.down_shares  - self.total_cost
    def unrealized_pnl(self, up_px: float, dn_px: float) -> float:
        return self.up_shares*up_px + self.down_shares*dn_px - self.total_cost
    # cost_per_pair_if_add_up(n, up_ask): (new_up_cost + down_cost)/min(new_up,down_shares); 0.0 if no pairs
    # cost_per_pair_if_add_down(n, dn_ask): mirror
    # apply_buy_up(n, price, order_id=""): update shares/cost, append to trades with timestamp
    # apply_buy_down(n, price, order_id=""): mirror
    # add_open_order(order), remove_open_order(order_id), get_all_order_ids()
```

### strategy/trend.py

```python
def detect_trend(history: list) -> str:
    # >= 3 points oldest-first. Rising: most_recent>oldest AND up_moves>down_moves. Falling: inverse. Flat: else.
def detect_up_trend(h): return detect_trend(h)
def detect_down_trend(h): return detect_trend(h)
```

### strategy/decision.py

```python
@dataclass
class TradeDecision:
    action: str   # "BUY_UP" | "BUY_DOWN" | "HOLD"
    shares: float; price: float; reason: str
    rule: str     # rule1|rule2_lock|rule2_expansion|rule3_lock|rule3_expansion|
                  # rule4_lock_down|rule4_lock_up|rule4_expansion|hold

def make_decision(pos, up_ask, dn_ask, up_hist, dn_hist, time_rem, base_size) -> TradeDecision:
```

**Rules (strict priority order):**

**R0 — Size:** `size = base_size`. If `time_rem < SIZE_REDUCE_AFTER_SECS`: `size = max(SIZE_MIN_SHARES, round(base_size * max(SIZE_MIN_RATIO, time_rem/SIZE_REDUCE_AFTER_SECS)))`.

**R1 — No position:** Up Rising → BUY_UP. Down Rising → BUY_DOWN. Else HOLD.

**R2 — Up only:** Lock if `up_avg + dn_ask < COST_PER_PAIR_MAX` → BUY_DOWN. Expansion if Down Rising AND `pnl_if_down < pnl_if_up` → BUY_DOWN enough shares so `pnl_if_down ≥ 0`, capped at `size×MAX_BUYS_PER_TICK`. Else HOLD.

**R3 — Down only:** Mirror of R2 (swap Up/Down).

**R4 — Both sides:** `up > down` and lock OK → BUY_DOWN. `down > up` and lock OK → BUY_UP. Expansion: Down Rising + down PnL lagging → BUY_DOWN; Up Rising + up PnL lagging → BUY_UP. Else HOLD.

**R5:** HOLD fallback.

---

## API (api/)

### auth.py — `PolyAuth(private_key, api_key, api_secret, passphrase, wallet_address)`
- `get_auth_headers(method, path, body="")` → HMAC-SHA256 of `timestamp+method+path+body` base64-encoded in `POLY_SIGNATURE`; also `POLY_ADDRESS`, `POLY_TIMESTAMP`, `POLY_NONCE="0"`, `POLY_API_KEY`, `POLY_PASSPHRASE`.
- `sign_order(order_data) → str` via `eth_account`.
- `build_order(token_id, side, size, price, expiration=0) → dict` with `makerAmount=str(int(size*price*1e6))`, `takerAmount=str(int(size*1e6))`, random salt, signed.

### clob_client.py — `CLOBClient(auth)`

| Method | Endpoint | Notes |
|---|---|---|
| `get_best_ask(token_id) → float` | GET /book?token_id=… | lowest ask |
| `get_order_book(token_id) → dict` | GET /book | full book |
| `place_order(token_id, side, size, price) → dict` | POST /order | build+sign; return response |
| `cancel_order(order_id) → bool` | DELETE /order/{id} | |
| `cancel_all_orders() → dict` | DELETE /orders | |
| `get_open_orders() → list` | GET /orders?maker=…&status=LIVE | |
| `get_wallet_balance() → dict` | GET /balance | `{"balance": float}`; fallback `0.0` |
| `get_market(market_id) → dict` | GET /markets/{id} | |
| `get_markets(params) → list` | GET /markets | |

On 429: sleep 2s retry ×2. On 5xx: sleep 1s retry ×1.

---

## PAPER TRADING SHIMMER

### paper_clob.py — `PaperCLOBClient(real_clob, paper_store)`

**Price methods** (`get_best_ask`, `get_order_book`, `get_market`, `get_markets`) → **delegate to `real_clob`** unchanged.

**Order methods — fully simulated, no HTTP:**
- `place_order(token_id, side, size, price)`: Check `paper_store.virtual_balance >= size*price` else raise `Exception("Paper: Insufficient virtual balance")`. Deduct balance. `order_id = f"PAPER-{uuid4().hex[:12].upper()}"`. Store in `_open_orders`. Return `{"orderID": id, "status": "MATCHED", "transactedAt": str(int(time.time())), "paper": True}`.
- `cancel_order(order_id)`: Remove from `_open_orders`, refund USDC. Return True.
- `cancel_all_orders()`: Cancel all, refund all, clear. Return `{"cancelled": count}`.
- `get_open_orders()`: Return `list(_open_orders.values())`.
- `get_wallet_balance()`: Return `{"balance": paper_store.get_virtual_balance()}`.

### paper_store.py — `PaperStateStore(trend_window, starting_balance)`

Mirrors `StateStore` completely but independent. Extra fields: `_virtual_balance`, `_starting_balance`, `_paper_trade_count`, `_paper_usdc_spent`, `_paper_realized_pnl`, `_closed_markets: List[dict]`.

Extra methods: `get_virtual_balance()`, `deduct_balance(amount)`, `credit_balance(amount)`, `record_paper_trade(market_id, side, shares, price, rule, reason)`, `record_closed_market(result_dict)` (append + update pnl + `credit_balance(max(0,pnl))`), `get_closed_markets()`, `get_paper_stats() → dict`, `reset(starting_balance)`. All standard StateStore methods also required. `threading.RLock` on all ops.

### paper_db.py — `PaperDB(db_path)`

SQLite WAL mode. Tables:
```sql
paper_sessions(id, started_at, ended_at, starting_balance, ending_balance, total_realized_pnl, trade_count, notes)
paper_markets(id, session_id, market_id, question, winner, up_shares, down_shares, total_cost, pnl, resolved_at, trade_count)
paper_trades(id, session_id, market_id, side, shares, price, total_cost, rule, reason, placed_at)
```
Methods: `start_session(starting_balance) → int`, `end_session(ending_balance, pnl, count)`, `save_market_result(...)`, `save_trade(...)`, `get_all_market_results()`, `get_session_market_results(session_id=None)`, `get_all_trades(session_id=None)`, `get_sessions_summary()`.

### paper_executor.py — `PaperExecutor(paper_clob, paper_store, paper_db, notifier)`

Same interface as live `Executor`: `execute(market, decision, position) → bool` and `cancel_all_open_orders() → int`. On fill: apply position + `record_paper_trade()` + `paper_db.save_trade()` + `notifier.send_paper_trade(trade_data)`. On insufficient balance: log + return False. No retries.

### paper_analytics.py — `PaperAnalytics`

`@staticmethod compute(results: List[dict]) → dict` returning:
`total_markets, winning_markets, losing_markets, breakeven_markets, win_rate_pct, total_realized_pnl, total_usdc_spent, avg_pnl_per_market, best_market_pnl, worst_market_pnl, best_market_name, worst_market_name, roi_pct, max_drawdown_pct, max_consecutive_wins, max_consecutive_losses, sharpe_ratio, avg_trades_per_market, rule_breakdown{rule→{count,total_pnl,win_rate}}, pnl_by_winner{UP/DOWN→pnl}, equity_curve[cumulative_pnl]`

Helpers: `_win_rate`, `_max_drawdown(equity)` (peak-to-trough %), `_sharpe_ratio(pnl_list)` (mean/std; 0.0 if std==0 or <3 pts), `_max_consecutive(results)`, `_rule_breakdown(results)`, `_equity_curve(results)`.

`@staticmethod format_report(analytics, virtual_balance, starting_balance) → str`: Full HTML Telegram message with sections OVERVIEW, PnL SUMMARY, RISK METRICS, RULE BREAKDOWN, WALLET using `═`/`─` dividers. Truncate to 4096 chars.

---

## LIVE INFRASTRUCTURE

### state/store.py — `StateStore(trend_window)`
`threading.RLock`. Fields: `_positions`, `_price_history`(deque/side), `_market_meta`, `_trade_count`, `_usdc_spent_today`, `_daily_realized_pnl`, `_start_time`, `_panic_mode`, `_trading_halted`. Methods: all CRUD + `add_realized_pnl(pnl)`, `get_stats() → dict`, `should_trade() → bool` (not panic and not halted).

### trader/executor.py — `Executor(clob, store, notifier)`
`execute(market, decision, position) → bool`: check `should_trade()` → signal `#logs` → `clob.place_order(...)` → on success update position + store + stats + `send_trade()`. Retry 429 ×2/2s, 5xx ×1/1s. `cancel_all_open_orders() → int`: bulk + per-order cancel, clear `open_orders`.

### monitor/market_finder.py — `MarketFinder(clob, live_store, paper_store, notifier, live, paper)`
`find_active_btc_5m_markets()`: Query Gamma API filtered by btc+5m. Extract `market_id, question, up_token_id, down_token_id, end_date_iso`. Register in `live_store` and `paper_store` if enabled. Telegram log on new markets. `@staticmethod get_time_remaining(market) → float`.

### monitor/closure_checker.py — `ClosureChecker(clob, live_store, paper_store, paper_db, notifier, live, paper)`
`check_and_record(market_id)`: GET market, if closed → determine winner → settle live position (compute pnl, `add_realized_pnl`, check loss limit, `send_market_closed`, remove) → settle paper position if in `paper_store` (`record_closed_market`, `paper_db.save_market_result`, `send_paper_market_closed`, remove).

---

## TELEGRAM BOT

### notifier.py — `TelegramNotifier(token, logs_id, trades_id)`
Dedicated asyncio loop in daemon thread. `run_coroutine_threadsafe`. HTML parse mode.

**Live:** `send_log(msg, level)` (ℹ️/⚠️/❌/📡 prefix), `send_trade(trade_data)`, `send_market_closed(...)`, `send_panic_alert(cancelled, details)`, `send_loss_limit_alert(daily_pnl, limit)`, `send_error(context, error)`.

**Paper:** `send_paper_log(msg, level)` — same as `send_log` but prefix `🧪 [PAPER]`. `send_paper_trade(trade_data)` — to `#trades`, card format: `🧪 [PAPER] SIMULATED TRADE`, market/order ID/shares@price/virtual_balance_after/rule/PnL scenarios. `send_paper_market_closed(...)` — `🧪 [PAPER]` result card to `#trades`. `send_paper_report(report_text)` — to `#logs`, split at 4096 chars.

**Trade card format** (live and paper): HTML divider lines (`─────`), bold labels, `+$X.XX` green / `-$X.XX` red PnL, ISO timestamp.

### dashboard.py — `Dashboard(live_store, paper_store, clob, paper_clob, live_exec, paper_exec, paper_db, notifier, stop_event, allowed_user_id, paper_starting_balance)`

All handlers: async, check `_auth_check` (allowed_user_id) first.

| Command | Summary |
|---|---|
| `/status` | Health: running/halted, markets, trades, USDC spent, realized PnL, uptime, panic |
| `/positions` | Live positions: `asyncio.gather` for live prices, unrealized PnL, time remaining |
| `/pnl` | Sum pnl_if_up/down_wins across all positions + realized PnL |
| `/wallet` | CLOB balance + internal: allocated, available, daily PnL, loss room, flags |
| `/panic` | Set panic_mode=True → `cancel_all_open_orders()` → `send_panic_alert()`. Reply count. |
| `/resume` | Set panic_mode=False + trading_halted=False → log → confirm |
| `/stop` | Set `stop_event` → confirm reply |
| `/help` | List all 13 commands with descriptions |
| `/paper_status` | Virtual balance, active markets, trade count, win rate, realized PnL, ROI |
| `/paper_positions` | Paper positions with live prices via `asyncio.gather`, unrealized PnL, time remaining |
| `/paper_report` | Merge in-memory + DB results → `PaperAnalytics.compute()` → `format_report()` → `send_paper_report()` |
| `/paper_history` | Last 10 closed paper markets from DB: name, winner, PnL. Session total at bottom. |
| `/paper_reset` | `paper_db.end_session(...)` → `paper_db.start_session(PAPER_STARTING_BALANCE)` → `paper_store.reset(...)` → confirm |

### bot.py — `TelegramBotRunner(token, dashboard)`
`start()`: daemon thread with new event loop. Register all 13 `CommandHandler`s. Run `app.run_polling()`.

---

## MAIN LOOP (main.py)

**Startup validation:** `LIVE=false AND PAPER=false` → exit error. `LIVE=true` → require wallet/API creds.

**Init order:** `live_store` → `paper_store` → `TelegramNotifier` → `paper_db.start_session(PAPER_STARTING_BALANCE)` → `PolyAuth` (if live) → `real_clob` → `paper_clob` → `live_exec` → `paper_exec` → `market_finder` → `closure_checker` → `stop_event` → `dashboard` → `TelegramBotRunner.start()`.

**Startup log:** Mode (LIVE/PAPER/BOTH), wallet prefix, paper balance, config summary.

**Background daemon threads:**
- `market_discovery`: `market_finder.find_active_btc_5m_markets()` every `MARKET_POLL_INTERVAL`s
- `closure_check`: for each market in `live_store.list_active_markets()`: `closure_checker.check_and_record(mid)` every `CLOSURE_CHECK_INTERVAL`s

**Main tick loop:**
```
while not stop_event.is_set():
  for market_id in live_store.list_active_markets():
    try:
      up_ask = real_clob.get_best_ask(up_token_id)
      dn_ask = real_clob.get_best_ask(dn_token_id)

      # Append same live prices to BOTH stores
      live_store.append_price(market_id, "up", up_ask); live_store.append_price(..., "down", dn_ask)
      if PAPER_TRADING: paper_store.append_price(...)  # same values

      up_hist = live_store.get_price_history(market_id, "up")
      dn_hist = live_store.get_price_history(market_id, "down")
      time_rem = MarketFinder.get_time_remaining(market)
      if time_rem <= 0: continue

      # LIVE block: make_decision on live_pos → live_exec.execute() up to MAX_BUYS_PER_TICK
      # Re-evaluate after each buy; break if action changes or !should_trade()

      # PAPER block: make_decision on paper_pos → paper_exec.execute() up to MAX_BUYS_PER_TICK
      # Re-evaluate after each buy; break if action changes or balance insufficient
    except Exception as e:
      logger.error(e); notifier.send_error(market_id, str(e))

  time.sleep(max(COOLDOWN_SECS, 1))
```

**Shutdown:** `paper_db.end_session(balance, pnl, count)` → `send_log("🛑 Bot stopped. Paper session archived.")`.

---

## MODES MATRIX

| LIVE | PAPER | Behavior |
|---|---|---|
| true | false | Live only. Wallet required. |
| false | true | Paper only. No wallet. Real prices, simulated fills. |
| true | true | Both parallel. Same markets, same prices, separate state. |
| false | false | Invalid — exit on startup. |

---

## IMPLEMENTATION RULES

1. `PaperCLOBClient` **always delegates price methods to `real_clob`**. Paper always uses live prices.
2. `make_decision()` is a **pure function** — no knowledge of live vs paper mode.
3. `paper_store` and `live_store` are **fully independent objects** with separate positions.
4. Paper fills are instant at ask price (no orderbook depth modeling — slightly optimistic).
5. Virtual balance: deduct `shares×price` on fill; credit `max(0, pnl)` on market close.
6. `/paper_report` merges in-memory closed markets with `paper_db.get_session_market_results()` (dedup by market_id).
7. All Telegram messages: HTML parse mode. Paper messages always prefixed `🧪 [PAPER]`.
8. `TelegramNotifier`: `run_coroutine_threadsafe` into dedicated daemon asyncio loop.
9. `StateStore` and `PaperStateStore`: `threading.RLock` on every read/write.
10. CLOB order payloads: 6-decimal micro-units. Telegram display: human-readable dollars.
11. Every real order signed via `PolyAuth.sign_order()`. Never submit unsigned.
12. `/panic` sets flag first (executor checks atomically), then cancels. Positions stay open — binary markets can only resolve at expiry.
13. Daily loss limit: `ClosureChecker` after each live close checks `daily_realized_pnl < -DAILY_LOSS_LIMIT_USD` → `set_trading_halted(True)` + `send_loss_limit_alert()`.
14. `/paper_reset` archives old session to SQLite before clearing in-memory state.

---

## OUTPUT FORMAT

Generate every file completely in this order:
1. `requirements.txt` 2. `.env.example` 3. `config.py` 4. `utils/logger.py` 5. `state/store.py` 6. `strategy/position.py` 7. `strategy/trend.py` 8. `strategy/decision.py` 9. `api/auth.py` 10. `api/clob_client.py` 11. `paper_trading/paper_db.py` 12. `paper_trading/paper_store.py` 13. `paper_trading/paper_clob.py` 14. `paper_trading/paper_executor.py` 15. `paper_trading/paper_analytics.py` 16. `monitor/market_finder.py` 17. `monitor/closure_checker.py` 18. `trader/executor.py` 19. `telegram_bot/notifier.py` 20. `telegram_bot/dashboard.py` 21. `telegram_bot/bot.py` 22. `main.py` 23. All `__init__.py` files grouped.

Header per file: `## File: polymarket_bot/filename.py` then full Python code block. No summaries. No skipped files. No placeholder methods.
