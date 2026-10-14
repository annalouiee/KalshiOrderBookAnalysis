# Project Brief: Liquidity Fragility in Prediction Markets

---

## How to use this document

This is the standing context file for the project. Two ways to use it:

1. **Save it as `CLAUDE.md` in the repo root.** Claude Code reads it automatically at the start of every session, so you never have to re-explain the project.
2. **Paste the relevant phase into a fresh agent session** when starting a new chunk of work.

Work through the phases in order. Each phase has a **Done when** section — do not move to the next phase until those conditions are met. Phases 1–4 are the foundation; everything downstream is worthless if the data pipeline is wrong.

**Kickoff template for each phase:**

> Read `CLAUDE.md` for project context. We are starting Phase N. Implement only what Phase N describes — do not build ahead into later phases. Before writing code, list the files you plan to create or modify and any assumptions you are making about the Kalshi API. Flag anything in the phase description you think is wrong.

---

## 1. Research goal

Prediction markets (Kalshi, Polymarket) have thin, fragile liquidity compared to equities because few institutional market makers participate. When a scheduled information shock hits — a CPI print, an FOMC rate decision — market makers mass-cancel resting orders to avoid being picked off, and the order book evaporates seconds before the event. Traders who hit the book during that window face extreme slippage.

Kalshi's own UI only shows historical price execution and top-of-book data, so this behavior is invisible to users. This project makes it measurable.

**Research questions:**

1. How does order book depth change in the window around a scheduled macroeconomic release?
2. Can liquidity evaporation be separated from ordinary trading activity (cancellations vs. fills)?
3. Can the magnitude of pre-event liquidity withdrawal predict post-event price slippage?

**Deliverables, by ambition level:**

- **Low risk (must achieve):** A reliable WebSocket ingestion pipeline with a verified local order book reconstruction.
- **Expected:** A real-time Level 2 dashboard, plus event-window data collection across enough scheduled releases for a meaningful sample, with depth and imbalance metrics computed.
- **Stretch:** A predictive simulator estimating spread widening for a future scheduled event; analysis of unscheduled events (requires defining what counts as one).

---

## 2. Core metrics to compute

These are the research output. Everything in the stack exists to produce them accurately.

| Metric | Definition | Notes |
|---|---|---|
| **Order book imbalance** | (bid depth − ask depth) / (bid depth + ask depth) | Compute at top-of-book and at N price levels deep |
| **Cancel velocity** | Contracts cancelled per unit time, per price level | **Derived — see §3.3.** This is the headline fragility metric |
| **Depth decay** | Rate of decline in total resting size over a time window | Measure separately for each side |
| **Slippage / book-walk cost** | Cost to execute a hypothetical market order of size X against the reconstructed book | Compute pre- and post-event for the same X |
| **Spread** | Best ask − best bid, in cents | Ask side is derived; see §3.2 |

Every metric must be computable from the raw stored log, so a changed definition means recomputation, never recollection.

---

## 3. Kalshi API: what we know

**⚠️ Verify all field names, endpoint paths, and channel names against Kalshi's current API documentation before writing parsing code.** The details below are from research notes and may be stale or imprecise. Treat the *concepts* as reliable and the *exact strings* as unverified.

### 3.1 Connection and authentication

- One authenticated WebSocket carries everything. Authentication is required even for public market data.
- Auth uses an RSA key pair. Requests are signed with headers along the lines of `KALSHI-ACCESS-KEY`, `KALSHI-ACCESS-TIMESTAMP`, and `KALSHI-ACCESS-SIGNATURE`.
- Relevant channels: `orderbook_delta` (Level 2 book) and `trade` (executions).
- On subscribing to `orderbook_delta`, Kalshi sends an `orderbook_snapshot` with the full book, then streams incremental `orderbook_delta` messages. Each delta carries the price level that changed and a signed quantity change: positive means size added, negative means size removed.
- There is also a REST endpoint for the order book, used in Phase 4 for validation.

### 3.2 The book is bid-only

Because of Kalshi's reciprocal pricing, the order book only publishes bids — YES bids and NO bids. The ask side must be derived:

```
YES ask at price P  ==  NO bid at price (100 − P)
NO  ask at price P  ==  YES bid at price (100 − P)
```

Example: a NO bid at 40¢ is a YES ask at 60¢.

Pick **one canonical side** (recommend: express everything in YES terms, prices in integer cents 1–99) and convert at the boundary of the book reconstruction module. Mixing conventions deeper in the code is the single most likely source of silent bugs in this project.

### 3.3 A negative delta is ambiguous — this is central

A negative delta means size left a price level. It does **not** say whether it was filled or cancelled. Since cancellations are the fragility signal we care about, they must be inferred:

```
cancelled_qty(price_level, interval) ≈ removed_qty(price_level, interval) − traded_qty(price_level, interval)
```

This requires subscribing to the `trade` channel from the very first collector, in the same process, with the same clock. **Do not defer the trade channel to a later phase** — data collected without it cannot answer the main research question, and scheduled events cannot be re-recorded.

Treat this as an estimate and document its assumptions (e.g. trade-to-delta attribution within the matching interval, handling of simultaneous fill-and-cancel at one level).

### 3.4 Sequence numbers and gaps

Every message carries a sequence number. A gap in the sequence means messages were dropped — usually from a connection interruption — and the locally reconstructed book is silently wrong from that point forward.

Required behavior:
- Check `seq` on every message.
- On a gap: log the gap with timestamps, discard the local book for that market, resubscribe, rebuild from the fresh snapshot.
- On disconnect: reconnect with exponential backoff and jitter.
- Persist a record of every gap and reconnect. Analysis must be able to exclude event windows that contain a gap — an event window with a silent hole in it is worse than no data.

### 3.5 Timestamping

Record **your local receive timestamp** (monotonic clock for intervals, UTC wall clock for alignment) on every message, in addition to any exchange timestamp. The two are different measurements and both matter. Run NTP on the collector host.

---

## 4. Stack (decided)

| Layer | Choice |
|---|---|
| Collector | Python: `asyncio`, `websockets`, `orjson`, `cryptography` (RSA signing), optionally `uvloop` |
| Raw backup | Append-only JSONL files, rotated hourly, compressed and optionally converted to Parquet daily |
| Primary store | TimescaleDB (PostgreSQL extension), written via `asyncpg` with batched `COPY` |
| Book + metrics | Separate Python process; reconstructs book, derives asks, computes metrics, writes to a metrics table |
| Heavy aggregation | In-database: SQL + TimescaleDB continuous aggregates |
| Per-event analysis | Python scripts using Polars; Jupyter notebooks only for exploration and charting of small result sets |
| API | FastAPI, streaming book state to the frontend over WebSocket |
| Frontend | React + TradingView `lightweight-charts`; canvas-based depth heatmap |
| Deployment | Docker Compose on an Oracle Cloud Always Free Ampere VM, Ashburn (US East) region |
| Monitoring | Heartbeat pings to healthchecks.io (free tier) |

### Rationale for the main choices

- **Python over a compiled language.** This project measures markets; it does not trade them. Message rates on prediction markets are orders of magnitude below equities, so Python keeps up comfortably. Network proximity to Kalshi's servers dominates language choice for latency.
- **Separate processes.** The collector must never be taken down by a dashboard bug or a slow analysis query. A crash in any other component must not cost event data.
- **Raw log as source of truth.** Metrics definitions will change during the project. Raw messages let every metric be recomputed; scheduled events cannot be re-recorded.
- **TimescaleDB over flat files alone.** Parsed columnar rows with compression are far smaller than raw JSON and let aggregation happen in the database rather than in pandas. JSONL files remain as a safety net.
- **Oracle Always Free.** The only genuinely permanent free tier with enough capacity (2 Arm Ampere cores, 12 GB RAM, 200 GB storage as of mid-2026). Caveats: the allowance has been reduced without announcement before, idle instances may be reclaimed, and the home region is permanent — choose Ashburn.

---

## 5. Repository layout (suggested)

```
.
├── CLAUDE.md                  # this file
├── docker-compose.yml
├── .env.example               # never commit real keys
├── collector/
│   ├── auth.py                # RSA signing
│   ├── client.py              # WebSocket connect, subscribe, reconnect
│   ├── sequence.py            # gap detection
│   ├── sink_jsonl.py          # raw append-only log
│   ├── sink_db.py             # batched COPY into TimescaleDB
│   └── main.py
├── book/
│   ├── reconstruct.py         # snapshot + deltas -> book state
│   ├── convert.py             # NO bids -> YES asks
│   ├── metrics.py             # imbalance, depth decay, cancel velocity, slippage
│   └── main.py
├── api/
│   └── main.py                # FastAPI
├── frontend/                  # React
├── analysis/
│   ├── events.py              # BLS/FOMC calendar handling
│   ├── extract_window.py
│   └── notebooks/
├── db/
│   └── migrations/
└── tests/
```

---

## 6. Phases

### Phase 0 — Scaffold

**Goal:** A repo an agent can work in without guessing.

- Repo structure as above, `pyproject.toml` or `requirements.txt`, linting (`ruff`), formatting, `pytest`.
- Config via environment variables; `.env.example` committed, `.env` gitignored.
- A market-list config file naming the tickers to track.
- `docker-compose.yml` with a TimescaleDB service (app services added in later phases).

**Done when:** `docker compose up db` works, tests run, and no secrets are tracked by git.

---

### Phase 1 — Connect and log raw data

**Goal:** Not losing messages. Nothing else.

- Implement RSA request signing.
- Connect to the WebSocket; subscribe to `orderbook_delta` **and** `trade` for 3–5 markets tied to upcoming CPI or FOMC releases.
- Write every message, with local receive timestamp, to rotating JSONL files. No parsing beyond what logging requires.
- Instrument message counts and bytes per second per market, and log a summary every minute.

**Done when:** The collector runs for 24 hours without crashing, and you can state the observed message rate and daily data volume in bytes. **Report those numbers — they size every later decision.**

**Do not yet:** reconstruct the book, write to the database, or build any UI.

---

### Phase 2 — Resilience

**Goal:** A collector that survives the real world, so the first captured release is not wasted.

- Sequence-gap detection per market; on a gap, log it and resubscribe for a fresh snapshot.
- Reconnect with exponential backoff and jitter; resubscribe to all markets on reconnect.
- A persisted `connection_events` log: connects, disconnects, gaps, resubscribes, with timestamps.
- Heartbeat ping to healthchecks.io so a dead collector sends an email alert.
- Graceful shutdown that flushes buffers.

**Done when:** Killing the network for 60 seconds produces a logged gap, an automatic reconnect, a fresh snapshot, and no crash — verified by test.

---

### Phase 3 — Database

**Goal:** Queryable storage that keeps up under load.

Schema sketch (refine as needed):

```sql
CREATE TABLE book_deltas (
  recv_ts  TIMESTAMPTZ NOT NULL,
  ticker   TEXT        NOT NULL,
  seq      BIGINT,
  side     TEXT,        -- raw Kalshi side, as received
  price    SMALLINT,    -- cents
  delta    INTEGER      -- signed
);
SELECT create_hypertable('book_deltas', 'recv_ts');
```

Plus `trades`, `book_snapshots`, and `connection_events`.

- Writes batched roughly every second via `asyncpg`'s `copy_records_to_table`. **Never insert one row per message.**
- Keep JSONL writing in parallel as the safety net.
- Compression policy on chunks older than ~1 day.
- A loader that replays JSONL into the database, so a database outage is recoverable.

**Done when:** The collector sustains the Phase 1 observed peak rate with the write buffer staying bounded, and a JSONL file can be replayed into the database to produce identical rows.

---

### Phase 4 — Book reconstruction and validation

**Goal:** Proof the data is trustworthy. **This is the gate for the whole project.**

- Rebuild the book from snapshot + deltas, in memory, per market.
- Convert NO bids into YES asks (§3.2), with unit tests on the conversion specifically.
- Periodically (every few minutes) fetch the REST order book and compare to the local reconstruction; log any mismatch with full context.
- Invalidate the local book on a sequence gap until a fresh snapshot arrives.

**Done when:** The reconstruction matches the REST order book across several hours of continuous running, with mismatches only explained by logged gaps. **This satisfies the "low-risk result" in the proposal — report it to Professor Fodor when it lands.**

---

### Phase 5 — Metrics engine

**Goal:** Turn a correct book into research numbers.

- Compute per snapshot interval (start with 1 second): imbalance (top-of-book and N-deep), total depth per side, spread, and book-walk cost for a few fixed order sizes.
- Compute cancel velocity per §3.3 by reconciling removed quantity against trades at each price level. Document the attribution rule chosen.
- Write results to a `metrics` table consumed by both dashboard and analysis — one source of numbers, never two.
- Add TimescaleDB continuous aggregates for per-minute rollups.
- Mark any interval overlapping a sequence gap as invalid, so analysis can exclude it.

**Done when:** Metrics exist for a full trading day, and a hand-computed check on a few intervals matches the pipeline output.

---

### Phase 6 — Event calendar and windows

**Goal:** Align market data to information shocks.

- Build an `events` table: event type, release time in UTC, source.
- Release times come from the **BLS calendar** (CPI, jobs reports — typically 08:30 ET) and the **FOMC calendar** (rate decisions — typically 14:00 ET). FRED's release-dates endpoint gives dates but not exact times; use it to cross-check dates only.
- A window-extraction function pulling metrics for an arbitrary window around an event (default ±10 minutes, configurable).
- **Record target markets continuously, not just around events.** Market makers often withdraw quotes minutes ahead of a release, the window may need widening later, and non-event baselines are needed for comparison.
- Generate matched non-event control windows (same market, same time of day, no scheduled release).

**Done when:** A single command produces a tidy metrics table for a named event across all tracked markets, plus its control windows.

---

### Phase 7 — Dashboard

**Goal:** Make the book visible — for the demo, and for spotting data problems by eye.

- FastAPI streaming current book state and recent metrics over WebSocket.
- React frontend: depth ladder, depth heatmap over time, price series via `lightweight-charts`, live imbalance and cancel-velocity readouts.
- A countdown and visual marker for the next scheduled event.
- A replay mode that plays a stored event window back through the same UI — the most valuable feature for the thesis defense.

**Done when:** The dashboard renders live data without affecting collector stability, and replay of a stored event window works.

**Note:** The dashboard does not need to be hosted. It can run locally against the remote database. Only the collector must run 24/7.

---

### Phase 8 — Deployment

**Goal:** Never miss a scheduled release.

- Dockerize collector, book/metrics process, and API; `restart: unless-stopped` on each.
- Deploy to the Oracle Always Free Ampere VM in Ashburn. Note that Ampere is ARM — use ARM-compatible base images.
- Nightly job copying compressed raw logs off the VM (Oracle Object Storage or Cloudflare R2 free tier). **Raw data is the one thing that cannot be recreated; it must not live in only one place.**
- Set a billing alert on day one. A card on file means an expired free allowance can quietly become a paid one.
- Caddy in front of FastAPI for automatic HTTPS if the dashboard is ever shared.
- Verify the VM's clock sync and measure round-trip latency to Kalshi from the VM.

**Done when:** The stack survives a VM reboot unattended and captures a real scheduled release end to end.

---

### Phase 9 — Analysis

**Goal:** Answer the research questions.

- Per-event analysis scripts (Polars) producing one results row per event per market: pre-event depth decay rate, peak cancel velocity, spread widening, realized post-event slippage.
- Compare event windows against matched control windows.
- Jupyter notebooks only for exploring and charting these small result sets.
- Characterize the typical shape of liquidity withdrawal around a release: when it starts, how deep it goes, how fast it recovers.

**Done when:** A results table covers every captured event, with figures suitable for the thesis.

---

### Phase 10 — Stretch

Only after Phase 9 is solid.

- **Predictive model:** given pre-event book state and event type, predict spread widening and slippage for a given order size. Start with a simple regression baseline; a model that cannot beat "the last three events looked like this" is not worth more complexity. Honor chronological train/test splits — no future data leaking into training.
- **Unscheduled events:** requires an operational definition (e.g. a volatility or volume threshold crossing with no scheduled release within N minutes). Document the definition carefully; it is itself a research contribution.
- **Simulator in the dashboard:** visualize predicted book state ahead of an upcoming event.

---

## 7. Standing rules for the agent

1. **The collector is sacred.** Never let another component's failure stop collection. Never deploy an untested change to the collector within an hour of a scheduled release.
2. **Raw data is append-only.** Never mutate or delete the raw log; derived tables can always be rebuilt from it.
3. **Verify API details against live documentation** rather than trusting the field names in this document.
4. **One canonical price convention** throughout (YES terms, integer cents), converted once at the book-reconstruction boundary.
5. **Never commit API keys.** Environment variables only.
6. **Flag estimates as estimates.** Cancel velocity is inferred, not observed — label it as such in code comments, schema comments, and any figure.
7. **Write tests for the conversion and reconstruction logic.** Those are where silent, data-corrupting bugs live.
8. **Prefer boring and verifiable** over clever. This is a research instrument; its credibility rests on being auditable.