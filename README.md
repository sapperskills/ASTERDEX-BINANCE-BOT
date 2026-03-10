# ASTERDEX-BINANCE-BOT
$ASTER  The bot that hunts the market.  Built by a real trading bot developer. Not a meme pretending to be tech — actual trading code.  ⚡ scans markets ⚡ hunts volatility ⚡ runs the books  No roadmap. Just execution.
# Binance Futures Unified v4.0 — Multi-Symbol Adaptive Trading Engine

A unified trading system that scans all markets, learns breakout patterns, and trades the best opportunities with on-the-fly parameter adaptation.

## Quick Start

```bash
# 1. Install dependency
pip install requests

# 2. Set your keys in config.json (or use --sim for paper trading)

# 3. Run multi-symbol auto mode (scans all markets, trades breakouts)
python run_unified.py --sim

# Or single-symbol mode (original behavior)
python run_unified.py --single BTCUSDT

# Or research-only mode (analyze all markets, no trading)
python run_unified.py --research
```

## Architecture

```
aster_unified/
├── run_unified.py              ← NEW: Unified entry point (all modes)
├── run.py                      ← Original single-symbol entry point
├── orchestrator.py             ← NEW: Multi-symbol trading coordinator
├── dashboard.py                ← NEW: Real-time monitoring dashboard
├── apply_patch.py              ← Merge research patches into configs
├── multi_symbol_research.py    ← Standalone research runner
├── config.json                 ← Base configuration
│
├── scanner/                    ← NEW: Market scanning + breakout detection
│   ├── market_scanner.py       ← Background scanner (all symbols)
│   └── breakout_detector.py    ← Multi-method breakout scoring
│
├── research/                   ← Multi-symbol research engine
│   ├── client.py               ← Public API client
│   ├── features.py             ← Feature engineering + regime detection
│   ├── strategies.py           ← Strategy simulations (4 types)
│   ├── optimizer.py            ← Parameter optimization + patching
│   ├── report.py               ← Report generation
│   └── engine.py               ← Research orchestration
│
├── core/                       ← State, config, types
│   ├── config.py               ← Config dataclass + JSON loader
│   ├── state.py                ← BotState persistence
│   ├── types.py                ← Snapshot, SymbolFilters, SymbolProfile
│   └── helpers.py              ← Pure utility functions
│
├── exchange/                   ← Exchange communication
│   ├── client.py               ← HTTP client, signing, time sync
│   ├── market.py               ← Price, depth, walls, OFI, spread
│   └── orders.py               ← Order placement, fills, cancellation
│
├── strategy/                   ← Signal + decision logic
│   ├── signals.py              ← MA, slope, vol, ATR computations
│   ├── decision.py             ← pick_action, confirm_action
│   └── risk.py                 ← TP/SL, trailing, sizing, daily limits
│
├── execution/                  ← Entry/exit orchestration
│   ├── entry.py                ← try_entry (maker + IOC logic)
│   └── exit_.py                ← try_exit (TP/SL/trail/emergency)
│
├── learning/                   ← ML models + adaptive tuning
│   ├── model.py                ← OnlineLogReg + RunningNorm
│   ├── learner.py              ← Labeling, training loop
│   └── adaptive.py             ← NEW: Online parameter tuning
│
└── utils/                      ← Logging, reconciliation, calibration
    ├── logging_.py             ← Central log + trade CSV
    ├── reconcile.py            ← Exchange ↔ local state sync
    └── calibration.py          ← Warmup symbol profiling
```

## How It Works

### Multi-Symbol Auto Mode (default)

1. **Scanner** runs in a background thread, continuously monitoring all USDT perpetuals:
   - Every 5 minutes: full research scan (klines, strategy simulations, regime detection)
   - Every 30 seconds: fast tick scan (price, volume, OFI breakout detection)

2. **Breakout Detection** uses 5 complementary methods:
   - **Range breakout**: price exceeds N-bar high/low by threshold
   - **Volume surge**: current volume >> rolling average
   - **Volatility expansion**: realized vol spike vs baseline
   - **OFI spike**: order flow imbalance exceeds historical norm
   - **Momentum**: consecutive same-direction moves
   - Composite score = weighted blend of all methods

3. **Orchestrator** spawns per-symbol trading threads when breakout score > threshold:
   - Each symbol gets its own Config, BotState, ML model
   - Research recommendation auto-applied as initial parameters
   - Warmup is model-aware and profile-aware (pretrained+profiled symbols start faster)
   - Portfolio-level risk limits enforced (max concurrent, max daily loss)

4. **Adaptive Learner** tunes parameters on the fly based on trade outcomes:
   - TP/SL multipliers adjusted based on win rate and profit factor
   - Trailing start fraction tuned based on TP vs trail exit ratio
   - Confirmation ticks adjusted based on false entry rate
   - OFI threshold tuned based on signal quality
   - Bayesian shrinkage prevents overfitting (capped deviation from priors)
   - Poor performers auto-paused (WR < 30%, 5+ consecutive losses)

### Modes

| Mode | Command | Description |
|------|---------|-------------|
| Multi-symbol | `python run_unified.py` | Scan all markets, trade breakouts |
| Single symbol | `python run_unified.py --single BTCUSDT` | Original single-symbol bot |
| Research only | `python run_unified.py --research` | Analyze markets, output reports |
| Simulation | `python run_unified.py --sim` | Force simulation (no real orders) |
| Dashboard | `python dashboard.py` | Real-time terminal monitoring |

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MAX_SYMBOLS` | 3 | Max concurrent symbols trading |
| `MAX_PORTFOLIO_LOSS` | 100 | Daily portfolio loss limit (USD) |
| `SCAN_INTERVAL` | 30 | Seconds between market scans |
| `BREAKOUT_THRESHOLD` | 0.45 | Min breakout score to trade (0-1) |

## Adaptive Learning Details

The adaptive learner maintains per-symbol performance statistics and adjusts parameters every 60 seconds (if enough trades have occurred). Adjustments include:

- **TP_ATR_MULT**: Widened if win rate is high but average win is small; tightened if win rate is low
- **SL_ATR_MULT**: Widened if stop-loss hit rate > 50%; tightened if barely hitting SL and profit factor is good
- **TRAIL_START_FRAC_TP**: Reduced if TP hits dominate (to capture more from runners); increased if trail exits dominate
- **ENTRY_CONFIRM_TICKS**: Increased on low win rate; decreased on very high win rate
- **HEUR_OFI_MIN**: Stricter on low win rate; looser on high win rate

All adjustments are capped at 1.5x deviation from the research prior to prevent runaway drift. State persists across restarts in `adaptive_state/`.

## Auto-Pause Conditions

A symbol is automatically paused when:
- Recent win rate (last 20 trades) drops below 30%
- 5 or more consecutive losses
- Max drawdown exceeds 40% of peak PnL for that symbol

## Upgrading Components

| Want to change... | Edit this file | Won't break... |
|---|---|---|
| Breakout detection methods | `scanner/breakout_detector.py` | Trading, learning, research |
| Scanner behavior | `scanner/market_scanner.py` | Trading, learning |
| Adaptive tuning logic | `learning/adaptive.py` | Scanner, trading |
| Multi-symbol coordination | `orchestrator.py` | Scanner, individual trading |
| Signal logic / indicators | `strategy/signals.py` | Orders, state, exchange |
| Entry/exit decision rules | `strategy/decision.py` | Execution, learning |
| TP/SL/trailing/sizing | `strategy/risk.py` | Signals, exchange, ML |
| Order types / maker logic | `exchange/orders.py` | Strategy, learning |
| ML model architecture | `learning/model.py` | Strategy, execution |
| Research strategies | `research/strategies.py` | Scanner, trading |

## Config Tips

- Every parameter in `config.json` overrides the defaults in `core/config.py`
- Unknown keys are silently ignored, so old configs work with new code
- In multi-symbol mode, the research engine generates per-symbol config patches automatically
- The adaptive learner further refines these patches online
- Use `SIMULATION_MODE: true` for risk-free testing


Scanner now consumes pretrain breakout profiles to make breakout thresholds symbol-aware (dynamic move / volume / volatility / OFI thresholds, plus slight score-bias and cooldown tuning from historical validation quality).


## Homeostatic population manager

This build adds a system-level controller that watches live/sim population health and dynamically adjusts scanner aggressiveness and promotion strictness. The engine rotates between EXPAND, NORMAL, and DEFENSIVE modes and persists its state to `population_state.json`.
