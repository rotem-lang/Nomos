# Nomos

**A C++ infrastructure for live stock trading.** You feed it a strategy plan and it trades
autonomously — fast, deadline-aware, and clean about broker connections.

## Goal

- Take a trading **strategy** (authored and backtested in a separate Python project) and
  **execute it live** in a fast, deterministic C++ engine.
- Be **fast** and **latency-aware**: know when there is enough time to complete a
  transaction and **pass (skip)** when there isn't.
- Handle broker **connections cleanly**: robust connect / reconnect / session management.
- Expose a **broker interface (`IBroker`)** with a **driver per broker** (first target:
  **Interactive Brokers**), so venues can be swapped without touching strategy logic.
- **Reuse, don't duplicate**, the existing Python trading logic (indicators, strategy
  parser, strategies) — the two sides meet through a shared, versioned contract.

## How it fits together (three planes)

| Plane | Where | Role |
| --- | --- | --- |
| **Authoring** | Python (separate, external repo) | Define indicators, parse strategies, backtest/research. |
| **Contract** | Shared, versioned | Strategy **IR** (compiled recipe) + **conformance test vectors**. |
| **Execution** | C++ (this repo) | Load the IR and trade — no Python in the live hot path. |

## Status

**Architecture & design phase — no production code yet.** See `CLAUDE.md` for current
context, locked decisions, and open questions, and `docs/Nomos-architecture.drawio` for the
system diagram.
