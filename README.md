# Elsa — Polymarket BTC Momentum Trader

> A Python bot that trades Polymarket's "BTC Up or Down — 5 minute" binary prediction markets using order-book momentum signals.

---

## Overview

Elsa is a live, automated trading bot built for [Polymarket](https://polymarket.com), the on-chain prediction market running on Polygon. It targets one specific market type: **BTC Up or Down — 5 minute**.

Each market resolves a single yes/no question: did the BTC/USD price (as reported by Chainlink) go up or down from the start of a 5-minute window to the end? Elsa reads the Polymarket order book and Binance real-time price data, looks for momentum signals early in each window, and places a directional bet when conditions align.

It has been live-tested with real money on Polymarket.

---

## How It Works

### The Market Structure

Polymarket's BTC 5-minute markets are continuous — one opens roughly every five minutes, 24/7. Each market has two outcome tokens: **UP** and **DOWN**. Prices are denominated in USDC on Polygon and represent implied probabilities (e.g., 0.52 = 52% chance).

Resolution is objective and on-chain: Chainlink's BTC/USD oracle price at the close of the window determines the winner. No ambiguity, no human judges.

### The Signal

Elsa does not predict BTC price movements from fundamentals or technicals on BTC itself. It reads the **order book as a signal**. The premise: informed participants move the book before price confirms. Elsa looks for momentum building in the order book early in a window — within a specific entry window after market open — and uses BTC price velocity as a confirming input.

When a signal fires, Elsa places a single FOK (Fill-or-Kill) market order at the current best price. It then monitors the position and exits based on profit target, stop-loss, directional reversal, or timeout.

---

## Architecture

Elsa is split into two independent processes that communicate via a shared log file:

```
┌─────────────────────────────────────────────────────────────┐
│                        OBSERVER                             │
│                                                             │
│  Polymarket CLOB API ──┐                                    │
│                        ├──► Merge & Snapshot ──► JSONL log  │
│  Binance WS/REST ──────┘         (~every 2s)                │
└─────────────────────────────────────────────────────────────┘
                              │
                         shared file
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         RUNNER                              │
│                                                             │
│  Tail JSONL log ──► Signal Detection ──► Order Placement    │
│                            │                    │           │
│                     Entry Filters          py-clob-client   │
│                            │                    │           │
│                     Position Monitor       Polygon / CLOB   │
│                      (TP / SL / exit)                       │
└─────────────────────────────────────────────────────────────┘
```

**Observer** runs continuously, polling the Polymarket CLOB and Binance approximately every 2 seconds. Each poll produces a structured snapshot — order book state, BTC price, spread, market metadata — appended as a JSONL line to a rolling log file.

**Runner** tails that log in real time. On each new snapshot, it evaluates entry conditions. If a signal fires and the position manager is idle, it places an order. While a position is open, it continuously evaluates exit conditions.

The two-process design means data collection never stops even if the trading logic restarts, and the observer log doubles as a historical dataset for backtesting.

---

## Strategy (High-Level)

Elsa's strategy is momentum-based, operating on the Polymarket order book within an early entry window of each 5-minute market.

**Inputs used:**

- **Price velocity in the order book** — how aggressively one side of the book is moving
- **BTC window delta** — BTC/USD price change since the current 5-minute window opened
- **Spread** — bid/ask spread on the target outcome token
- **Price level** — current implied probability of the outcome

**Entry logic:**

Entry requires multiple conditions to align simultaneously. Book momentum must exceed a threshold, BTC directional signal must agree, spread must be within acceptable bounds, and the price must be in a range where the risk/reward makes sense. All conditions are evaluated within a time window early in the market — Elsa does not enter late.

**Exit logic:**

Once in a position, Elsa monitors four exit conditions:

| Exit type | Description |
|-----------|-------------|
| Take-profit | Price moves +10% from entry |
| Stop-loss | Price moves against position beyond threshold |
| Delta reversal | BTC price reverses direction significantly |
| Timeout | Market approaching close with no resolution |

Orders are placed as FOK market orders — either fully filled at the current best price or rejected. No partial fills, no hanging limit orders.

---

## Risk Management

- **Configurable stake per trade** — set a fixed USDC amount per order in config
- **Daily loss limit kill switch** — Runner tracks cumulative P&L for the day; if losses exceed a configured threshold, it stops placing new orders until manually reset
- **Paper trading mode** — full simulation with no real orders; signals fire and exits are tracked, but no CLOB calls are made
- **FOK orders only** — eliminates exposure from partially filled or stale orders
- **Position size discipline** — only one position open at a time; no pyramiding

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Python 3.11+ |
| Order execution | [py-clob-client](https://github.com/Polymarket/py-clob-client) |
| Wallet signing | eth-account (EIP-712) |
| Config / secrets | python-dotenv |
| HTTP | requests |
| Blockchain | Polygon (MATIC) |
| Data format | JSONL (newline-delimited JSON) |
| Process management | Bash watchdog scripts |

Credentials required: a Polymarket API key and an Ethereum private key for a proxy wallet funded with USDC on Polygon.

---

## Getting Started

### Prerequisites

- Python 3.11+
- A Polymarket account with API credentials
- A Polygon wallet (proxy wallet) funded with USDC
- Binance API access (public endpoints, no key required for price data)

### Install

```bash
git clone https://github.com/hal9000thebot/elsa-trader-public
cd elsa-trader-public
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Configure

Copy `.env.example` to `.env` and fill in your credentials:

```env
POLY_API_KEY=...
POLY_API_SECRET=...
POLY_API_PASSPHRASE=...
PRIVATE_KEY=0x...           # Ethereum private key for proxy wallet
STAKE_USDC=5.0              # USDC per trade
DAILY_LOSS_LIMIT=25.0       # Kill switch threshold
PAPER_TRADING=false         # Set true to run without placing real orders
```

### Run

Start the observer first, then the runner in a separate terminal:

```bash
# Terminal 1
python observer.py

# Terminal 2
python runner.py
```

For 24/7 operation, use the included watchdog scripts:

```bash
./watchdog_observer.sh &
./watchdog_runner.sh &
```

---

## Project Structure

```
elsa-trader/
├── observer.py          # Data collection process
├── runner.py            # Signal detection + order execution
├── strategy.py          # Entry/exit condition logic
├── position.py          # Position state management
├── clob_client.py       # Polymarket CLOB wrapper
├── binance_client.py    # Binance price feed
├── replay.py            # Backtesting harness (replays JSONL logs)
├── sweep.py             # Parameter sweep for backtesting
├── watchdog_observer.sh # Auto-restart script for observer
├── watchdog_runner.sh   # Auto-restart script for runner
├── logs/                # JSONL snapshots + trade logs
├── .env.example         # Environment variable template
└── requirements.txt
```

### Backtesting

The observer logs are the historical dataset. `replay.py` replays any log file through the signal detection logic with configurable parameters, producing a trade-by-trade summary. `sweep.py` runs a grid search over parameter ranges to find configurations that performed well on historical data.

---

## Disclaimer

This software is provided for educational and research purposes. Trading prediction markets involves real financial risk. Past performance of any configuration is not indicative of future results. Polymarket markets are permissionless and on-chain — transactions are irreversible. Use at your own risk.

This project is not affiliated with Polymarket, Chainlink, or Binance.

---

*Built for the vibecoding meetup. Yes, it trades real money. Yes, it was mostly built with Claude.*
