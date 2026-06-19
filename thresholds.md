# Trading thresholds & hardcoded defaults

Reference for every rule, limit, and default in this repo. Most trading values can be overridden via **environment variables** or the **`ai_trading_settings`** row in Supabase.

---

## Quick reference (money & risk)

| What | Default | Env override | CircleCI |
|------|---------|--------------|----------|
| **Sell transaction penalty** | ₹150 per SELL | `SELL_TRANSACTION_CHARGE_INR` | `150` on `execute-ai-trades` job |
| **Buy transaction charge** | ₹0 (none) | `BUY_TRANSACTION_CHARGE_INR` | not set |
| **Max invested per stock** | ₹25,000 | `MAX_POSITION_INR` | not set |
| **Min hold before sell** | 90 calendar days | `MIN_HOLD_DAYS` | not set |
| **Thesis-break loss** | 10% down vs entry | `THESIS_BREAK_LOSS_PCT` | not set |
| **Max position % of portfolio** | 15% | DB: `ai_trading_settings.max_position_pct` | — |
| **Max open positions** | 5 | DB: `ai_trading_settings.max_positions` | — |
| **Min cash reserve** | 20% of portfolio | DB: `ai_trading_settings.min_cash_reserve_pct` | — |
| **Stale price refresh band** | ±3% | `PRICE_REFRESH_RATIO` | `0.03` |
| **Max cache age (refresh)** | 10 days | `MAX_CACHE_AGE_DAYS` | `10` |

Source files: `agent/trading_constraints.py`, `agent/execute_ai_trades.py`, `agent/portfolio_db.py`, `.circleci/config.yml`.

---

## Transaction charges

Paper model of Indian **delivery** exit costs (STT + DP + exchange + brokerage). Real markets also charge on **buy**; this repo defaults to **sell-only** for simplicity.

| Leg | Default | Env var | When applied |
|-----|---------|---------|--------------|
| SELL | **₹150** flat | `SELL_TRANSACTION_CHARGE_INR` | Deducted from wallet cash after each SELL; reduces `realized_pnl` and logged `pnl` |
| BUY | **₹0** | `BUY_TRANSACTION_CHARGE_INR` | Optional; reserved in buy sizing if &gt; 0 |

**Round trip** (if both set): `BUY_TRANSACTION_CHARGE_INR + SELL_TRANSACTION_CHARGE_INR`.

**Example:** Gross profit ₹300 on a sell with ₹150 sell penalty → **net ₹150**.

Indian delivery reality (for context): STT ~0.1% on **both** buy and sell; stamp duty on buy; DP charge on sell (~₹15–80+ per leg on small trades depending on broker and size).

---

## Min hold period (90 days)

After a live **BUY**, the executor normally **blocks SELL** for **90 calendar days** (`MIN_HOLD_DAYS`, default `90`).

Purpose: swing trading — avoid churning on noise; align with multi-week holds.

If AI recommends SELL inside 90 days → action becomes **SKIP** with reason `min_hold_period` (unless thesis-break applies).

---

## Thesis-break loss (10%)

**Thesis-break** = the original reason you bought is treated as **broken** when price falls enough that an early exit is allowed.

| Setting | Default | Env var |
|---------|---------|---------|
| Loss threshold | **10%** below average entry | `THESIS_BREAK_LOSS_PCT` |

**Logic** (in `can_sell_under_min_hold`):

1. If held **≥ 90 days** → sell allowed (if AI says SELL).
2. If held **&lt; 90 days** and loss **≤ −10%** vs avg entry → sell **allowed** (thesis break).
3. If held **&lt; 90 days** and loss is **less severe than −10%** (e.g. −5% vs entry) → sell **blocked**.

**Examples** (entry ₹1,000, thesis-break 10% → threshold ₹900):

| Day | Price | Loss | SELL allowed? |
|-----|-------|------|---------------|
| 30 | ₹880 | −12% | Yes (thesis break) |
| 30 | ₹950 | −5% | No (min hold) |
| 100 | any | any | Yes (min hold satisfied) |

Thesis-break does **not** force a sell — it only **permits** one when the AI recommendation is SELL and you are still inside the hold window.

Same numbers are repeated in LLM prompts (`agent/tradingagents/agents/utils/swing_policy.py`) so agents and the executor stay aligned.

---

## Position sizing & portfolio limits

### Per-stock cap (`MAX_POSITION_INR`)

| Default | Env |
|---------|-----|
| **₹25,000** total invested per ticker (including adds) | `MAX_POSITION_INR` |

Buys are skipped when `room_to_cap` &lt; one share price (`max_position_cap_reached`).

### Executor settings (`ai_trading_settings`)

Fixed row id: `00000000-0000-0000-0000-000000000002`.

If the row is **missing**, code fallbacks in `execute_ai_trades.py`:

| Field | Fallback | Meaning |
|-------|----------|---------|
| `auto_trade` | `true` | Execute paper trades |
| `dry_run` | `false` | Actually write to DB |
| `max_position_pct` | **0.15** (15%) | Max % of portfolio value per new/add position |
| `max_positions` | **5** | Max distinct open holdings |
| `min_cash_reserve_pct` | **0.20** (20%) | Keep this % of portfolio in cash |

Buy sizing: `deployable = cash_for_trade − portfolio_value × min_cash_reserve_pct`, capped by `max_position_pct`, `MAX_POSITION_INR`, and available cash.

### Prompt heuristics (not env — `swing_policy.py`)

Embedded in agent prompts for consistency with UI:

| Rule | Value in prompts |
|------|------------------|
| Min gain to favour selling | ~**3%** upside vs basis |
| Max drawdown / stop band | ~**10%** vs basis |
| Min hold | **90 days** |
| Max per stock | **₹25,000** |

Changing env vars updates **executor logic**; prompt text changes only if you edit `swing_policy.py` (or sync from swing-trader).

---

## Price refresh & stale cache

Used by `refresh_stale_recommendations.py` and the `execute-ai-trades` job (price refresh step).

| Setting | Default | Env | CircleCI |
|---------|---------|-----|----------|
| Refresh if price moved | **±3%** vs cached reference | `PRICE_REFRESH_RATIO` | `0.03` |
| Refresh if cache older than | **10 days** | `MAX_CACHE_AGE_DAYS` | `10` |

---

## LLM & agent defaults

### `agent/tradingagents/default_config.py`

| Setting | Default |
|---------|---------|
| Provider | `anthropic` |
| Models | `glm-5.2` (deep + quick) |
| Backend URL | `https://api.z.ai/api/anthropic` |
| Debate rounds | **1** |
| Risk discuss rounds | **1** |
| Recursion limit | **1000** |
| Data vendors | all **yfinance** |

### Env overrides

| Env var | Default if unset |
|---------|------------------|
| `LLM_PROVIDER` | `anthropic` |
| `DEEP_THINK_LLM` / `QUICK_THINK_LLM` | `glm-5.2` |
| `LLM_BACKEND_URL` | Z.ai URL above |
| `MAX_DEBATE_ROUNDS` | `1` |
| `MAX_RECUR_LIMIT` | `1000` |
| `LLM_HTTP_TIMEOUT` | `300` for glm (final client kwargs; glm branch sets `180` first, then overwritten) |
| `LLM_HTTP_MAX_RETRIES` | `5` for glm (same overwrite) |
| `SIGNAL_EXTRACT_MAX_ATTEMPTS` | `4` |
| `SIGNAL_EXTRACT_RETRY_DELAY_SEC` | `1.25` |
| `DATA_VENDOR_STOCKS` / `INDICATORS` / `FUNDAMENTALS` / `NEWS` | `yfinance` |
| `YFINANCE_HISTORY_PERIOD` | `10d` |

### CircleCI `batch-shard` job (hardcoded in `.circleci/config.yml`)

- `LLM_PROVIDER=anthropic`
- `DEEP_THINK_LLM=glm-5.2`
- `QUICK_THINK_LLM=glm-5.2`
- `LLM_BACKEND_URL=https://api.z.ai/api/anthropic`

API keys (`Z_API_KEY`, `GLM_API_KEY`, etc.) must be set in CircleCI Project Settings — see `env.example`.

---

## CircleCI pipeline

| Item | Value |
|------|-------|
| Schedule | Mon–Fri **05:30 UTC** = **11:00 IST** (`30 5 * * 1-5`) |
| Branch | `main` only (on schedule trigger) |
| Batch shards | **6** shard jobs (`shard_total: 6`) |
| Expected tickers | **22** from `RECOMMENDATION_TICKERS` |
| Python image | `cimg/python:3.12` |
| Job timeouts | batch **50m**, execute **30m**, price refresh **60m** |

### When pipelines run (important)

| Trigger | `pipeline.trigger_source` | Workflows that run |
|---------|---------------------------|-------------------|
| **Cron schedule** (config `30 5 * * 1-5` UTC) | `schedule` or `scheduled_pipeline` | `scheduled-ai-recommendations` only |
| **Trigger Pipeline** (CircleCI UI / API) | `api` | `manual-ai-recommendations` only |
| **Git push** to `main` | `webhook` | **None** — both workflows have `when` filters that exclude push |

There is **no** default workflow without a `when` clause, so push does **not** run batch, execute, or LLM jobs.

**Verify in CircleCI UI:**

1. **Project Settings → Advanced** — avoid extra “build on push” workflows outside this config.
2. **Project Settings → Triggers** — if you also created a **schedule trigger** in the UI, you may get **duplicate** daily runs (config cron + UI cron). Use **one** scheduling method, or align both to the same time.
3. Scheduled run is **Mon–Fri UTC cron** only — no Saturday/Sunday fire (matches NSE weekdays for the cron itself; trade date still rolls Sat/Sun IST to Friday in scripts).

| Runs on git push? | **No** (empty pipeline possible, no jobs) |

---

## Tickers

**22 symbols** — not hardcoded in Python. Set via:

- CircleCI secret: `RECOMMENDATION_TICKERS`
- Local: `.env` (see `env.example`)

List in `env.example`: 20 NSE names + `BTC-USD` + `ETH-USD`.

---

## Fixed database IDs

| ID | Purpose | Defined in |
|----|---------|------------|
| `00000000-0000-0000-0000-000000000001` | Paper wallet (`ADMIN_WALLET_ID`) | `agent/portfolio_db.py` |
| `00000000-0000-0000-0000-000000000002` | AI trading settings row (`SETTINGS_ID`) | `agent/execute_ai_trades.py` |

Also created in swing-trader Supabase migrations.

---

## Display / context limits (code only)

| File | Limit |
|------|-------|
| `portfolio_db.load_recent_portfolio_trades` | **5** trades |
| `portfolio_db.load_recent_ai_recommendations` | **8** rows |
| `portfolio_db.load_backtest_strategy_summaries` | **8** rows |

These affect LLM context size only, not trading rules.

---

## Skip reasons (executor)

When a trade does not run (or is not attempted), `ai_trade_executions.skip_reason` may be:

| Reason | `action_taken` | Meaning |
|--------|----------------|---------|
| `already_executed` | SKIP | One execution per ticker/date/dry_run already logged |
| `no_recommendation` | SKIP | No row in `ai_recommendation_cache` for this ticker/date |
| `no_price` | SKIP | Could not resolve a trade price |
| `already_holding_no_overweight` | SKIP | BUY but already hold; decision not Overweight |
| `max_position_cap_reached` | SKIP | At ₹25k (or `MAX_POSITION_INR`) cap |
| `max_positions_reached` | SKIP | Already at max open positions (DB or fallback 5) |
| `insufficient_cash` | SKIP | Cash too low (includes buy charge if set) |
| `quantity_zero` | SKIP | Sized to 0 shares |
| `no_position_to_sell` | **HOLD** | SELL signal but no open position |
| `min_hold_period` | SKIP | SELL inside min hold and loss not at thesis-break level |
| `auto_trade_disabled` | SKIP | `ai_trading_settings.auto_trade = false` |

---

## Changing values

1. **Local / CircleCI env** — see tables above (`env.example` for template).
2. **Supabase** — update `ai_trading_settings` for portfolio %, max positions, cash reserve, auto_trade, dry_run.
3. **Prompts** — edit `agent/tradingagents/agents/utils/swing_policy.py` for agent-facing copy.

After changing env vars in CircleCI, the next scheduled or manual pipeline run picks them up (no push required for secret-only changes).
