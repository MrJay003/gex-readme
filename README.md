# GEX Quant Engine

A Python-based **quantitative trading signal engine** built around dealer gamma exposure (GEX). It fuses GEX regime analysis with price-action structure, volume profile, inter-market correlation, and dynamic risk management into a weighted confluence scoring system.

GEX is the driving factor. No GEX confirmation = no signal.

---

## Table of Contents

- [How It Works](#how-it-works)
- [Confluence Architecture](#confluence-architecture)
- [Risk Management Stack](#risk-management-stack)
- [Backtest Results](#backtest-results)
- [Installation](#installation)
- [CLI Quick Reference](#cli-quick-reference)
- [Futures Commands](#futures-commands)
- [Options Commands](#options-commands)
- [Options Backtesting](#options-backtesting)
- [Signal Logger](#signal-logger)
- [Morning Briefing](#morning-briefing)
- [GEX Integration](#gex-integration)
- [Historical GEX Backfill](#historical-gex-backfill)
- [ES Correlation](#es-correlation)
- [TradingView Integration](#tradingview-integration)
- [CME Futures Support](#cme-futures-support)
- [Pine Script Export](#pine-script-export)
- [Interval-Aware Scaling](#interval-aware-scaling)
- [Module Descriptions](#module-descriptions)
- [Configuration](#configuration)
- [Running Tests](#running-tests)
- [Disclaimer](#disclaimer)

---

## How It Works

The engine scans bar-by-bar through this pipeline:

1. **Market structure** — swing highs/lows, structural breaks, trend detection
2. **Supply/demand zones** — institutional zones from pivot-based detection
3. **Liquidity sweeps** — stop-run reversals exceeding minimum ATR depth
4. **Key levels** — structural levels from session opens, swing points, prior day H/L
5. **Session filter** — only trade during high-volume session windows
6. **GEX hard gate** — signal rejected if GEX doesn't confirm (regime, gamma flip, or wall alignment)
7. **HTF trend filter** — signal must align with 1h structural trend or it's skipped
8. **Confluence scoring** — score each candidate against weighted factors
9. **Threshold gate** — signal must score ≥ 3.5 with ≥ 3 distinct confluences
10. **Risk management** — ATR-based SL, dynamic R:R, trailing stop, partial profits, stall exit

Signals are logged automatically to `signal_logs/signals.jsonl` for forward-test analysis.

---

## Confluence Architecture

### Scored Differentiators

These factors vary between signals and determine the confluence score:

| Confluence | Weight | What It Measures |
|---|---|---|
| **Fib Retracement** | **2.0** | Price in 62–79% Fibonacci retracement of prior day's range |
| VPOC | 1.5 | Price near yesterday's Volume Point of Control |
| Value Area | 1.5 | Position relative to VAH/VAL |
| Rejection Zone | 1.5 | HTF wick rejection zone — institutional footprint |
| ES Confirmation | 1.5 | ES futures structural trend agrees with NQ signal direction |
| GEX | 1.5 | Dealer gamma regime + wall + flip confirmation |
| Supply/Demand Zone | 1.0 | Signal originates from a valid institutional zone |
| Liquidity Sweep | 1.0 | Stop-run reversal detected before entry |
| VIX Correlation | 1.0 | Risk sentiment from VIX trend |
| NQ/ES Ratio | 1.0 | Relative strength ratio confirms direction |

### Hard Gates (weight 0 — pass/fail, no score inflation)

| Gate | Role |
|---|---|
| GEX | **Driving factor** — signal rejected when GEX data present but no confirmation |
| HTF Trend | Signal must align with 1h structural trend or skipped entirely |
| Volume Delta | Reject if ≥60% of volume opposes signal direction |
| Session Window | Must be inside a defined session window |
| Time Exclusion | NY lunch (12:00–13:30) hard-blocked |
| Market Structure | Trend alignment enforced |

### Time-of-Day Bonus/Penalty

| Window | Weight | Time (ET) |
|---|---|---|
| Late Morning | +2.5 | 10:00–11:00 |
| NY Open | +2.0 | 08:30–10:00 |
| Afternoon Session | +2.0 | 14:00–15:00 |
| Power Hour | +1.5 | 15:00–16:00 |
| NY Lunch | -2.0 | 12:00–13:30 (hard-blocked) |

---

## Risk Management Stack

| Control | Value (5m) | Purpose |
|---|---|---|
| Dynamic R:R | 1.5 / 2.0 / 2.5 / 3.0 | Scales with ATR volatility regime |
| ATR Trailing Stop | Activates 3.0× ATR, trails 2.5× ATR | Lets winners run |
| Partial Profit | 33% closed at 1R | Locks gains, trails remaining 67% |
| Breakeven Stop | Triggers at 1.0× ATR | Eliminates risk after favorable move |
| Stall Exit | 18 bars with < 0.3× ATR progress | Cuts dead trades |
| Max Bars | 72 (6 hours on 5m) | Time stop |
| Cooldown | 36 bars (3 hours on 5m) | Prevents overtrading |
| Max Signals/Day | 2 | High-conviction only |

---

## Backtest Results

### NQ Futures (Underlying)

| Test | Trades | Win Rate | Profit Factor | Return |
|---|---|---|---|---|
| TV 5m + GEX | 28 | 75% | 3.02 | 174% |
| TV 15m | 36 | 58% | 1.84 | — |
| yfinance 5m | 45 | 62% | 1.69 | — |
| CSV 5m 71d (no GEX) | 49 | 59% | 1.37 | 104% |

### Options Portfolio (SPY + QQQ + TSLA, 90 days, historical per-day GEX)

| Strategy | Trades | Win Rate | Profit Factor | Return | Max DD |
|---|---|---|---|---|---|
| Debit Spreads (auto) | 358 | 66.2% | 1.87 | +222% | 13.7% |
| Long Calls/Puts | 359 | 48.2% | 1.00 | -0.7% | 26.5% |

Spreads significantly outperform naked longs because they hold overnight and capture multi-day moves. Naked longs are forced to exit same-day with daily bar resolution.

---

## Installation

```bash
git clone <repo-url>
cd gex-quant-engine
pip install -r requirements.txt
```

### API Keys

```bash
# GEX data (required for GEX gate)
export FLASHALPHA_API_KEY=your_key_here

# Massive (required for options chains + backtesting)
export MASSIVE_API_KEY=your_key_here

# TradingView (optional — for premium data)
export TRADINGVIEW_USERNAME=your_user
export TRADINGVIEW_PASSWORD=your_pass
```

---

## CLI Quick Reference

### Base Command

```bash
python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
  --min-confluences 4.0 --gex-cache --es-corr
```

### Modes

| Mode | Flag | Description |
|---|---|---|
| **Morning Briefing** | `--briefing` | Key levels, GEX regime, dealer interpretation, active zones |
| **Live Signal** | `--signal` | Latest actionable signal as a compact card (auto-logged) |
| **Backtest** | `--backtest` | Full trade simulation with P&L report |
| **Options Backtest** | `--options-backtest` | Overlay options P&L on underlying trades via Massive |
| **Chart** | `--plot` | Interactive Plotly chart with signals |
| **Pine Script** | `--tradingview-pine` | Paste-ready TradingView overlay |

### Key Flags

| Flag | Purpose |
|---|---|
| `--gex-cache` | Disk-cache GEX data (saves API calls) |
| `--gex-refresh` | Force fresh GEX fetch (overwrites cache) |
| `--es-corr` | Enable ES/NQ inter-market correlation confluence |
| `--htf-interval 1h` | HTF timeframe for trend + rejection blocks |
| `--no-gex` | Disable GEX even if API key is set |
| `--rth-only` | Restrict to RTH (auto-enabled for CME sub-hourly) |
| `--sessions cme` | Override session profile |
| `--n-bars 10000` | Fetch more history |
| `--options-strategy` | Force `long`, `spread`, or `auto` (default) |

---

## Futures Commands

Commands for NQ/ES/MNQ/MES futures trading — the core workflow.

```bash
# Morning briefing — GEX regime, key levels, dealer interpretation
python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
  --min-confluences 4.0 --gex-cache --es-corr --gex-refresh --briefing

# Live signal card (auto-logged to signal_logs/)
python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
  --min-confluences 4.0 --gex-cache --es-corr --signal

# Full backtest with GEX + ES correlation
python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
  --min-confluences 4.0 --gex-cache --es-corr --backtest

# Backtest from CSV (no API needed)
python main.py --csv sample_data/NQ_5m_60d.csv --interval 5m --htf-interval 1h \
  --sessions cme --backtest

# Interactive chart with signals overlaid
python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
  --min-confluences 4.0 --gex-cache --es-corr --plot

# Pine Script overlay for TradingView
python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
  --min-confluences 4.0 --gex-cache --tradingview-pine --output nq_levels.pine
```

---

## Options Commands

The engine scans a curated universe of ~120 liquid tickers, runs the full GEX + confluence pipeline on each, and recommends options contracts (strike, expiry, strategy) based on signal direction and GEX regime.

### Live Scan for Signals

```bash
# Scan with options recommendations (strike, expiry, strategy)
python main.py --scan --scan-tickers SPY,QQQ,TSLA --options --gex-cache \
  --interval 5m --htf-interval 1h

# Scan the full universe — ranked by confluence score
python main.py --scan --options --gex-cache

# Scan by tier (1=mega-liquid, 2=high-liquid, 3=liquid)
python main.py --scan --scan-tiers 1,2 --options --gex-cache

# Limit number of symbols scanned
python main.py --scan --scan-max 20 --options --gex-cache
```

### Sample Scan Output

```
  Rank  Ticker   Dir   Score   #Conf  GEX        Entry        R:R   ATR      Time
  ──────────────────────────────────────────────────────────────────────────────
  1     TSLA     BUY   5.0     5      positive   $395.95      2.0   NORMAL   04-17 14:10:00
  2     SPY      SELL  4.5     5      positive   $707.06      2.0   NORMAL   04-17 13:45:00

  OPTIONS TRADE RECOMMENDATIONS
  ────────────────────────────────────────────────────────────────────────
  BUY Call Debit Spread  →  TSLA 405/410C Apr 24  ($5 wide, 5DTE)
    ↳ mean-reverting regime — put wall support at $400; IV=64%; IVR=73
  SELL Put Debit Spread  →  SPY 703/708P Apr 24  ($5 wide, 5DTE)
    ↳ mean-reverting regime — call wall resistance at $710; IV=14%; IVR=42
```

---

## Options Backtesting

Simulates options trades against historical data from Massive, using real contract prices. Requires `MASSIVE_API_KEY`.

```bash
# Portfolio backtest — debit spreads (default, best performance)
python main.py --scan --scan-tickers SPY,QQQ,TSLA --backtest --backtest-days 90 \
  --account-size 10000 --gex-cache --es-corr --interval 5m --htf-interval 1h \
  --options-backtest

# Force straight long calls/puts
python main.py --scan --scan-tickers SPY,QQQ,TSLA --backtest --backtest-days 90 \
  --account-size 10000 --gex-cache --es-corr --interval 5m --htf-interval 1h \
  --options-backtest --options-strategy long

# Force spreads only
python main.py --scan --scan-tickers SPY,QQQ,TSLA --backtest --backtest-days 90 \
  --account-size 10000 --gex-cache --es-corr --interval 5m --htf-interval 1h \
  --options-backtest --options-strategy spread
```

### Options Exit Parameters

| Parameter | Long Options | Debit Spreads |
|---|---|---|
| Profit Target | +65% of premium | +50% of max profit |
| Stop Loss | -50% of premium | -60% of max loss |
| Hold Overnight | No (same-day exit) | Yes |
| Min DTE | 3 days | 3 days |

### Historical GEX (No Look-Ahead Bias)

The scanner loads per-day GEX from `.gex_cache/` so each bar sees only that day's actual GEX — no future data leaks into signals. Days without cached GEX get no signals (GEX hard gate rejects them).

Coverage is printed during backtests: `[GEX 62/65 days]`.

---

## Signal Logger

Every signal fired in `--signal` mode is automatically appended to `signal_logs/signals.jsonl`:

```json
{
  "id": "a1b2c3d4e5f6",
  "timestamp": "2026-04-11T10:15:00-04:00",
  "direction": "BUY",
  "entry_price": 24865.50,
  "stop_loss": 24812.25,
  "take_profit": 24971.00,
  "confluence_score": 5.5,
  "confluences": ["market_structure", "gex", "fib_retracement", "vpoc", "session_window"],
  "gex": {"regime": "positive", "gamma_flip": 25270.5, "call_wall": 25072.0, "put_wall": 24865.0},
  "volatility_regime": "NORMAL",
  "rr_used": 2.0,
  "outcome": null
}
```

- **Duplicate-safe** — deterministic ID; re-runs don't double-log
- **Purpose** — build forward-test proof over 30+ days

---

## Morning Briefing

`--briefing` generates a forward-looking report: market structure, GEX levels with dealer interpretation, key levels, and active trade zones.

```
====================================================
        NQ 5m – Morning Briefing (2026-04-11)
====================================================
  Market Structure:  BEARISH (last break at 25,134.25)
  HTF Trend (1h):    BULLISH (last break at 24,422.00)
  Current Price:     25,035.50
  GEX Regime:        POSITIVE (mean-reverting)
  Gamma Flip:        25,270.50 (resistance above)
  Call Wall:         25,072.38 (resistance)
  Put Wall:          24,865.17 (support)

  Dealer Interpretation:
    Gamma: Dealers short gamma — moves amplified
    Vanna: Vol up = dealers sell delta — downside amplified
    Charm: Time decay pushing dealers to sell — pressure into close
====================================================
```

---

## GEX Integration

GEX (Gamma Exposure) is the **driving factor**. It acts as a hard gate — signals are rejected when GEX data is present but doesn't confirm.

### What GEX Does

| Check | Condition | Effect |
|---|---|---|
| Regime | Positive (mean-reverting) | Confirms all directions |
| Regime | Negative (trending) + aligned with trend | Confirms trend signals |
| Gamma Flip | Price within 1% of flip level | Structural transition zone |
| Put Wall | BUY near put wall | Support confirmation |
| Call Wall | SELL near call wall | Resistance confirmation |
| **No match** | **None of the above** | **Signal rejected** |

### ETF Proxy Scaling

FlashAlpha provides ETF-level GEX. The engine scales to futures price space:

| Futures | ETF Proxy | Formula |
|---|---|---|
| NQ, MNQ | QQQ | `futures_level = etf_level × (NQ_price / QQQ_price)` |
| ES, MES | SPY | Same scaling |

### Disk Cache

GEX data is cached to `.gex_cache/` when `--gex-cache` is used:
- `QQQ_2026-04-11_exposure.json` — full per-strike gamma data
- `QQQ_2026-04-11_levels.json` — pre-computed walls, flip, OI
- `QQQ_2026-04-11_summary.json` — dealer interpretation text

Use `--gex-refresh` to force a fresh API call and overwrite today's cache.

---

## Historical GEX Backfill

Two tools populate `.gex_cache/` with historical GEX for bias-free backtesting:

### Snapshot-Based (Current Day)

```bash
# Cache all tier-1 tickers (runs daily via LaunchAgent at 9:45 AM)
python cache_gex_daily.py

# Cache specific tickers
python cache_gex_daily.py --tickers SPY,QQQ,TSLA
```

### Historical Backfill (From Massive S3 Flat Files)

Computes GEX from historical options data — OI estimated from volume, gamma via Black-Scholes.

```bash
# Backfill 90 days for a ticker
python backfill_gex_historical.py --ticker SPY --days 90 --skip-existing --fixed-iv

# Backfill date range
python backfill_gex_historical.py --ticker QQQ --start 2026-01-15 --end 2026-04-18
```

Current coverage: SPY (64 days), QQQ (76 days), TSLA (64 days), plus 15 other tier-1 tickers.

---

## ES Correlation

`--es-corr` enables inter-market ES/NQ correlation analysis:

- **ES Confirmation** (weight 1.5) — ES structural trend agrees with NQ signal direction
- **NQ/ES Ratio** (weight 1.0) — NQ leading/lagging ES confirms relative strength

---

## TradingView Integration

```bash
pip install tradingview-datafeed
```

Credentials via env vars or anonymous access:

```bash
export TRADINGVIEW_USERNAME=your_user
export TRADINGVIEW_PASSWORD=your_pass
```

### Common Symbols

| Short | Symbol | Exchange |
|---|---|---|
| NQ | NQ1! | CME_MINI |
| ES | ES1! | CME_MINI |
| MNQ | MNQ1! | CME_MINI |
| MES | MES1! | CME_MINI |

---

## CME Futures Support

### Session Windows

| Session | Time (ET) |
|---|---|
| US Pre-market | 08:30–09:30 |
| RTH Open | 09:30–10:30 |
| Full RTH | 09:30–16:00 |
| RTH Close | 15:00–16:00 |

### Contract Specs

| Symbol | Point Value | Tick Size |
|---|---|---|
| NQ | $20.00 | 0.25 |
| MNQ | $2.00 | 0.25 |
| ES | $50.00 | 0.25 |
| MES | $5.00 | 0.25 |

---

## Pine Script Export

```bash
python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
  --gex-cache --tradingview-pine --output nq_levels.pine
```

Draws Gamma Flip (yellow), Call/Put Wall (red/green), VPOC (blue), Fib zone (aqua), key levels (silver).

---

## Interval-Aware Scaling

`interval_params.py` auto-tunes risk parameters per timeframe:

| Interval | R:R | ATR× | Cooldown | Max Bars | Trail | Stall |
|---|---|---|---|---|---|---|
| 5m | 2.0 | 1.0 | 36 bars | 72 | 3.0 / 2.5 | 18 / 0.3 |
| 15m | 1.5 | 1.0 | 8 bars | 32 | — | — |
| 1h | default | 1.5 | default | — | — | — |
| 4h | 2.5 | 1.5 | 4 bars | — | — | — |
| 1d | 3.0 | 2.0 | 3 bars | — | — | — |

---

## Module Descriptions

| Module | Description |
|---|---|
| `signal_engine.py` | Core — bar-by-bar confluence scoring with hard gates and weighted factors |
| `config.py` | `Config` dataclass — all parameters |
| `main.py` | CLI entry point |
| `gex.py` | `GEXFetcher` + `GEXData` — FlashAlpha API fetch, parse, cache, interpretation |
| `gex_proxy.py` | Futures→ETF mapping and price scaling |
| `gex_profile.py` | Multi-expiry GEX profile analysis |
| `correlation.py` | ES/NQ inter-market correlation and ratio analysis |
| `signal_logger.py` | Forward-test signal journal (JSONL) |
| `backtester.py` | Trade simulation with breakeven, trailing, partials, stall exit |
| `briefing.py` | Morning briefing report with GEX interpretation |
| `market_structure.py` | Swing detection, structural breaks, trend |
| `order_blocks.py` | Supply/demand zone detection and signal generation |
| `rejection_blocks.py` | HTF wick rejection zones — institutional footprint |
| `liquidity.py` | Liquidity levels, sweep detection, reversal signals |
| `key_levels.py` | Structural key levels (swings, breaks, PDH/PDL, session opens) |
| `sessions.py` | Session window filtering with timezone-aware logic |
| `premium_discount.py` | Equilibrium, zone classification, Fibonacci retracement |
| `volume_profile.py` | VPOC, VAH, VAL computation |
| `fib_levels.py` | Fibonacci retracement zone computation |
| `trading_day.py` | CME trading day boundaries (6PM–5PM ET) |
| `interval_params.py` | Per-interval parameter auto-tuning |
| `econ_calendar.py` | Economic calendar hard gate (FOMC, CPI, NFP) |
| `pine_export.py` | Pine Script v6 overlay generator |
| `visualizer.py` | Plotly interactive charts |
| `data_loader.py` | CSV / yfinance / TradingView data loading |
| `scanner.py` | Multi-symbol scanner — runs engine across the universe, ranks by score |
| `options_selector.py` | Strike/expiry picker — maps signals to contracts (long calls/puts, spreads) |
| `options_backtester.py` | Options-level trade simulation via Massive historical data |
| `polygon_options.py` | Live options chain fetcher — real greeks, IV, bid/ask from Massive |
| `universe.py` | Curated ~120 ticker universe across 3 liquidity tiers |
| `cache_gex_daily.py` | Batch GEX cacher — runs daily via LaunchAgent |
| `backfill_gex.py` | Snapshot-based GEX builder from live options chain |
| `backfill_gex_historical.py` | Historical GEX backfill from Massive S3 flat files |
| `backfill_greeks.py` | Historical greeks backfill (BS model on daily option bars) |
| `greeks.py` | Black-Scholes: `norm_cdf`, `bs_price`, `implied_vol`, `compute_greeks` |

---

## Configuration

Key settings in `Config` (see `config.py`):

```python
from config import Config

config = Config(
    confluence_threshold=3.5,       # Minimum weighted score
    min_confluence_count=3,         # Minimum distinct factors
    max_signals_per_day=2,          # High-conviction only
    risk_per_trade=0.01,            # 1% of account
    account_size=10_000.0,
    dynamic_rr_normal=2.0,          # R:R scales with volatility
    trail_activation_atr=3.0,       # Start trailing after 3× ATR
    trail_step_atr=2.5,             # Trail 2.5× ATR behind best
    stall_exit_bars=18,             # Exit stalled trades after 18 bars
)
```

---

## Running Tests

```bash
pip install pytest
python -m pytest tests/ -q
```

1174 tests, all synthetic data — no API calls required.

---

## Disclaimer

> **This software is for educational and research purposes only.**
> It does not constitute financial advice. Trading futures and options carries
> substantial risk of loss. Past performance (including backtests) is not
> indicative of future results. The authors accept no liability for financial
> losses incurred through the use of this software.

---

## Daily Procedures

### Futures (NQ) — Daily Workflow

**Pre-Market (before 9:30 AM ET)**

1. **Cache today's GEX** (auto-runs via LaunchAgent at 9:45 AM, or manual):
   ```bash
   python cache_gex_daily.py --tickers SPY,QQQ
   ```

2. **Morning briefing** — review GEX regime, dealer positioning, key levels:
   ```bash
   python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
     --min-confluences 4.0 --gex-cache --es-corr --gex-refresh --briefing
   ```

3. **Review briefing output** — note gamma flip, put/call walls, dealer gamma/vanna/charm interpretation. Identify buy/sell zones.

**Active Trading (9:30 AM – 4:00 PM ET)**

4. **Pull live signal card** when price approaches a briefing zone:
   ```bash
   python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
     --min-confluences 4.0 --gex-cache --es-corr --signal
   ```

5. **Execute the trade** if a signal fires — entry, SL, and TP are printed on the card. Signal is auto-logged to `signal_logs/signals.jsonl`.

6. **Repeat step 4** as needed — max 2 signals/day enforced by the engine.

**Post-Market**

7. **Review signal log** — check today's entries:
   ```bash
   tail -5 signal_logs/signals.jsonl | python -m json.tool
   ```

8. **Update outcomes** — fill in `exit_price`, `pnl_points`, `outcome` in the JSONL for forward-test tracking.

---

### Options (Multi-Symbol) — Daily Workflow

**Pre-Market (before 9:30 AM ET)**

1. **Cache GEX for all tier-1 tickers**:
   ```bash
   python cache_gex_daily.py
   ```

2. **Run a live scan** with options recommendations:
   ```bash
   python main.py --scan --scan-tickers SPY,QQQ,TSLA --options --gex-cache \
     --interval 5m --htf-interval 1h
   ```

3. **Review scan output** — ranked signals with strike, expiry, and strategy (spread vs long). Note GEX regime and IV rank for each.

**Active Trading**

4. **Execute top-ranked trades** — enter the recommended contracts. Spreads are preferred (higher WR, hold overnight).

5. **Manage exits** per the engine's parameters:
   - **Spreads**: TP at +50% of max profit, SL at -60% of max loss, hold overnight allowed
   - **Long options**: TP at +65% of premium, SL at -50% of premium, same-day exit

6. **Re-scan mid-day** if positions closed early (afternoon session signals can fire):
   ```bash
   python main.py --scan --scan-tickers SPY,QQQ,TSLA --options --gex-cache \
     --interval 5m --htf-interval 1h
   ```

**Post-Market**

7. **Log outcomes** — record actual fills, exit prices, and P&L for forward-test validation.

---

### Weekly Maintenance

1. **Verify GEX cache coverage** — confirm no gaps in `.gex_cache/`:
   ```bash
   ls .gex_cache/ | grep QQQ | wc -l
   ```

2. **Run backtest** to validate edge is holding:
   ```bash
   python main.py --tv NQ1! --exchange CME_MINI --interval 5m --htf-interval 1h \
     --min-confluences 4.0 --gex-cache --es-corr --backtest
   ```

3. **Run options backtest** with latest GEX data:
   ```bash
   python main.py --scan --scan-tickers SPY,QQQ,TSLA --backtest --backtest-days 90 \
     --account-size 10000 --gex-cache --es-corr --interval 5m --htf-interval 1h \
     --options-backtest --options-strategy spread
   ```

4. **Run test suite** after any code changes:
   ```bash
   python -m pytest tests/ -q
   ```
