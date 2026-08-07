# CLAUDE.md — Nomos project memory

> Living context for Claude across sessions. **Standing rule: at the end of every working
> session, update this file** with new decisions, changes, and open threads. Keep it current
> and concise. Newest status at the bottom (see "Session log").

---

## Mission

**Nomos** is a **C++ infrastructure for live stock trading**. You feed it a strategy plan
and it trades autonomously. It must be:
- **Fast** and low-latency.
- **Deadline-aware** — it must know whether it has enough time to complete a transaction and
  **pass (skip)** when it doesn't.
- **Clean about connections** — robust connect/reconnect/session handling.

Trading logic (indicators, strategy parser, strategies, backtesting) already lives in a
**separate Python repo owned by someone else**. We integrate with it **without duplicating
its logic**. We build a **broker interface (`IBroker`) + a driver per broker**.

---

## Locked decisions (as of 2026-08-07)

- **Language / build:** C++ (target C++20/23), CMake + vcpkg.
- **First broker:** **Interactive Brokers (IBKR)** — TWS API / IB Gateway to start.
- **Execution model:** Strategies compile to a **serializable IR** ("Intermediate
  Representation" — a machine-readable recipe). **No Python runs in the live hot path.** C++
  loads the IR and runs everything in-process.
- **Latency tier:** *Mixed / grow into it* — start at low-millisecond; architect the hot
  path so we can tighten toward sub-ms later **without a rewrite**. (Note: IBKR itself caps
  at ~tens of ms; sub-ms only becomes real by swapping to a colocated/FIX venue behind the
  same `IBroker` seam.)
- **IR contract format:** a **shared schema** — **FlatBuffers recommended** (zero-copy reads;
  generates both Python and C++ types); Protobuf is the fallback.
- **Python repo status:** third-party, **cannot freely change** → treated as a black box.
- **Anti-duplication approach (recommended):** **dual indicator implementations** (Python
  for backtest, C++ for live) kept honest by a **shared golden conformance test-vector
  suite** in CI. (A single C++ core bound into Python via pybind11 was ruled out because we
  can't add a C++ dependency to the third-party Python repo.)

---

## Architecture — three planes

```
Authoring Plane (Python, external)  →  Contract (shared)  →  Execution Plane (C++, this repo)
  indicators · parser · strategies      Strategy IR            fast, deterministic trading
  backtest / research                   Conformance vectors
```

- **Authoring plane (theirs):** humans design/parse/backtest strategies. Emits strategies +
  (ideally) exports the IR and golden vectors. If it can't export IR itself, we add a thin
  **external adapter/transpiler** that reads its output — without modifying their repo.
- **Contract:** versioned IR schema + corpus of golden indicator/strategy test vectors. This
  is the only coupling point between Python and C++.
- **Execution plane (ours):** loads the IR and trades.

### Execution-plane components (live data flow: market data in → orders out)
1. **Clock / Time & Deadline budgeting** — RTT/offset estimation to broker; latency
   histograms; the "do I have enough time?" decision (`now + est_cost + margin > deadline →
   PASS`). Clock sync here is about *correct timestamps/deadlines*, not raw speed.
2. **Market Data** — feed handler + **bar builder** (must define/close bars identically to
   the Python backtest — a contract item).
3. **Indicator library** — **streaming/incremental** indicators; correctness enforced by
   conformance vectors.
4. **Strategy Engine** — loads IR, wires indicators → conditions → actions. Single-threaded,
   allocation-free in steady state.
5. **OMS + Risk** — order lifecycle state machine; pre-trade risk checks; **kill switch**;
   idempotent orders (client order IDs).
6. **`IBroker` interface + drivers** — the swappable seam; first driver = IBKR.
7. **Session / Connection mgmt** — connect→auth→subscribe→live→degraded→reconnect;
   heartbeats, backoff, gap handling, clean shutdown.
8. **Telemetry / Logging / Order Journal** — async (non-blocking) logging; append-only
   journal for crash recovery/reconciliation; latency + pass/skip + fill metrics.

### Concurrency (evolutionary)
- Phase 1: thread-per-component + SPSC lock-free ring buffers (correctness first).
- Phase 2: pin threads, busy-poll where it pays, remove steady-state allocs. Kernel bypass
  only if a sub-ms venue justifies it.

---

## Key risks to keep front-of-mind

1. **Backtest ≠ live parity** (biggest money risk) — two indicator impls will drift (float
   order, bar-close timing, timezone/session, look-ahead). Mitigate with conformance vectors
   **and** a planned **shadow mode** (run C++ on recorded data, diff decisions vs Python).
2. **Can the Python strategy language even compile to a finite IR?** If strategies are
   arbitrary Python, the "no Python in hot path" model breaks. Must see the strategy
   vocabulary before finalizing the schema.
3. **IBKR realities** — API pacing limits, market-data line caps, order-ID rules, Gateway
   latency. Don't gold-plate the hot path for a venue that can't exploit it.
4. **Crash/restart reconciliation** — broker is source of truth; reconcile open
   orders/positions from journal + broker query or risk duplicate/phantom positions.
5. **Determinism under threads** — prefer a sequenced event log driving the engine.

---

## Open questions (must answer before Phase 1 — about the Python repo)

Drive the IR schema + C++ data model:
1. How are indicators implemented (pure Python / NumPy / Numba / TA-Lib)?
2. What is a "strategy" concretely (class w/ callbacks vs declarative config vs parser AST)?
3. What does the parser parse, and does it emit a reusable AST/object tree?
4. Full strategy **vocabulary** (conditions/actions/order types/sizing/time rules) → IR opcodes.
5. Input **data model** (bars vs ticks/quotes; timeframes; exact bar-close definition).
6. Indicator **statefulness** (batch vs streaming) + warmup/lookback.
7. Required **parity tolerance** (exact float vs epsilon).
8. Can the Python repo **export IR + vectors** itself, or do we need an external adapter?

---

## Proposed repo skeleton (NOT created yet — planning only)

```
/schema        # .fbs IR schema (the shared contract)
/src
  /core        # clock, deadline, ring buffers, pools
  /marketdata  # feed handlers, bar builder
  /indicators  # streaming indicators (conformance-tested)
  /strategy    # IR loader + engine/interpreter
  /oms         # orders, positions, risk, kill switch
  /broker      # IBroker interface + /broker/ibkr driver
  /session     # connection/session state machines
  /telemetry   # logging, metrics, journal
/tests/{unit,conformance}
/tools         # IR adapter/transpiler (if Python can't emit IR directly)
/docs          # architecture diagram (exists)
```

Tech stack (proposed): FlatBuffers, spdlog (async), GoogleTest/Catch2, IBKR paper account.

---

## Artifacts in this repo

- `docs/Nomos-architecture.drawio` — editable architecture diagram (open in draw.io desktop).
- `docs/Nomos-architecture.png` — rendered image of the same.
- Full design write-up (plan): `~/.claude/plans/lets-talk-architecture-and-wild-creek.md`.

---

## Status / where we are

**Phase: architecture & design only. NO production code written yet.** We are still pinning
down the system design and the Python-repo contract.

## Session log

### 2026-08-07
- Established mission, locked the decisions above.
- Produced the three-plane architecture and the `docs/Nomos-architecture.drawio` diagram
  (reorganized into a clean vertical execution pipeline + right-rail cross-cutting services).
- Set the standing rule to update this file every session.
- **Next session (tomorrow):** user has **issues with the system design itself** — to be
  discussed before proceeding. Candidate topics flagged: (a) whether market-data belongs
  behind the same `IBroker` seam as orders or a separate feed abstraction; (b) the C++-only
  indicator split; (c) whether Clock/Deadline is over-engineered for IBKR; (d) whether a
  single vertical pipeline oversimplifies multi-strategy / multi-instrument execution.
- Still owed: answers to the 8 Python-repo open questions.
