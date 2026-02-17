# BlockEnvy

**The Solana backtesting engine. Replay any historical slot with bit-for-bit accuracy, then run thousands of Monte Carlo simulations to backtest how your transaction _would_ have performed under different conditions.**

---

## The Problem

When you submit a swap on Solana, the outcome you get — your fill price, your slippage, whether your transaction even landed — is the result of a chaotic 400-millisecond race. Your transaction competed with hundreds of others for block space. It was reordered by the validator's scheduler. It may have been front-run. Network latency decided when it arrived relative to everything else.

You see one outcome. But there were thousands of possible outcomes. Without a backtesting engine, you're flying blind:

- **Traders** can't tell if a bad fill was caused by their strategy, network conditions, or adversarial behavior
- **Market makers** can't model how different fee/timing strategies would have performed historically
- **Protocols** can't quantify how their users are affected by MEV and block dynamics
- **RPC providers and validators** can't offer their customers any insight into _why_ transactions behave the way they do

## The Solution

BlockEnvy is a **slot-fidelity backtesting engine** for Solana. It does two things:

**1. Deterministic Replay** — Download the archival data for any historical slot, reconstruct the exact bank state from snapshot archives, and re-execute every single transaction through the real Agave validator runtime. Compare the results field-by-field against the archival record to prove the replay is bit-for-bit correct.

**2. Monte Carlo Backtesting** — Run 1,000+ simulations of that same slot with randomized conditions: network latency, arrival order, MEV bot behavior, competing transactions. Each simulation produces a different outcome for your target transaction. Aggregate the results into statistical distributions that tell you what _range_ of outcomes you were exposed to, and _why_.

This is the infrastructure that turns "I got a bad fill" into "here's the probability distribution of fills I was exposed to, decomposed by cause."

---

## Why This Matters

### For Traders and Market Makers
- **Backtest execution strategies** against real historical slots, not toy simulations
- **Quantify slippage distributions** — get p5/p50/p95 outcomes from thousands of simulations, not a single data point
- **Decompose _why_ a trade went wrong** — was it network propagation (43%), lock contention (28%), MEV (22%), or compute capacity (7%)?
- **Sensitivity analysis** — re-run with specific factors disabled (no MEV, no jitter, no ghost flow) to isolate each variable

### For RPC Providers and Validators
- **Offer backtesting as a premium service** — give customers actionable intelligence about their transaction outcomes, not just raw RPC data
- **Differentiate your infrastructure** — this is a product that sits on top of archival RPC + snapshot archives, the exact data you already have
- **Plug in any data source** — BlockEnvy uses a generic archival RPC interface and standard snapshot formats. Point it at your own RPC endpoint and snapshot store
- **Batch processing** — backtest hundreds of slots in parallel with bounded concurrency

### For Protocol Developers
- **Verify execution correctness** — deterministic replay compares every field (fees, balances, errors, compute units, token balances) against archival data
- **AMM microstructure analysis** — integer-exact off-chain AMM math (starting with Raydium CPMM) validated against actual Agave execution results
- **Understand how block dynamics affect your users** — quantify the real-world impact of transaction ordering on your protocol's swap outcomes

### For Researchers
- **Fully reproducible** — every random decision is seeded: `trial_seed = SHA256(base_seed || trial_index)`. Same inputs always produce the same outputs, down to the integer token unit
- **Configurable models** — swap out network models (Agave-like vs Firedancer-like pipeline), tune MEV probability, adjust ghost flow rates, all from a TOML config file
- **Full provenance chain** — every run produces a manifest with SHA-256 hashes of every input (binaries, snapshots, blocks, configs). Run ID = `base32(sha256(manifest))`. You can prove exactly what produced a given result

---

## How It Works

```
     Pick a historical Solana slot to backtest (e.g., slot 280,000,000)
                          |
                          v
    +-----------------------------------------+
    |           1. FETCH                      |
    |  Download snapshot archives from your   |
    |  object store + block data from your    |
    |  archival RPC endpoint                  |
    +-----------------------------------------+
                          |
                          v
    +-----------------------------------------+
    |           2. BOOT                       |
    |  Load the snapshot into an Agave worker |
    |  subprocess to reconstruct the exact    |
    |  Solana bank state at that point in     |
    |  time                                   |
    +-----------------------------------------+
                          |
                          v
    +-----------------------------------------+
    |           3. REPLAY                     |
    |  Re-execute every transaction in the    |
    |  target slot through the real Agave     |
    |  runtime. Compare results against       |
    |  archival ground truth field-by-field.  |
    |                                         |
    |  Fields verified:                       |
    |  - err (success/failure)                |
    |  - fee                                  |
    |  - preBalances / postBalances           |
    |  - preTokenBalances / postTokenBalances  |
    |  - computeUnitsConsumed                 |
    |  - logMessages, innerInstructions, ...  |
    +-----------------------------------------+
                          |
                          v
    +-----------------------------------------+
    |           4. BACKTEST                   |
    |  Run N Monte Carlo trials (default      |
    |  1,000) with stochastic models:         |
    |                                         |
    |  - Network: 2-lane latency model        |
    |    (staked vs public, log-normal)       |
    |  - Scheduler: lock-aware fee-priority   |
    |    with aging bonus                     |
    |  - Ghost flow: Poisson synthetic txs    |
    |  - MEV: front-run/back-run pairs with   |
    |    configurable probability             |
    |  - Private lane: FIFO bundle injection  |
    |                                         |
    |  Each trial produces: inclusion bool,   |
    |  focus output amount, slippage, and a   |
    |  "why" code explaining the outcome.     |
    +-----------------------------------------+
                          |
                          v
    +-----------------------------------------+
    |           5. REPORT                     |
    |  Aggregate results:                     |
    |  - Inclusion probability with CI        |
    |  - Slippage distribution (p5/p50/p95)   |
    |  - Multi-cause attribution breakdown    |
    |  - 200-bin histogram of outcomes        |
    |                                         |
    |  Output as JSON or Markdown             |
    +-----------------------------------------+
```

---

## Architecture

BlockEnvy uses a **multi-process orchestrator/worker model**. This isn't a library — it's a system of cooperating processes:

```
  ┌──────────────────────────────────────────────────────────┐
  │                  ORCHESTRATOR (blockEnvy)                 │
  │                                                          │
  │  Config, CAS cache, data acquisition from any RPC,       │
  │  Monte Carlo coordination, report generation             │
  │                                                          │
  │  Communicates with workers via gRPC over Unix sockets    │
  └──────────┬──────────────┬──────────────┬─────────────────┘
             │              │              │
    gRPC/UDS │     gRPC/UDS │     gRPC/UDS │
             │              │              │
  ┌──────────▼──┐  ┌───────▼──────┐  ┌───▼──────────────┐
  │  Worker #1  │  │  Worker #2   │  │  Worker #3       │
  │  (Agave     │  │  (Agave      │  │  (Agave          │
  │   3.1.8)    │  │   3.1.8)     │  │   3.1.8)         │
  │             │  │              │  │                   │
  │  Sandboxed: │  │  Sandboxed:  │  │  Sandboxed:      │
  │  - No net   │  │  - No net    │  │  - No net        │
  │  - 320G mem │  │  - 320G mem  │  │  - 320G mem      │
  │  - R/O CAS  │  │  - R/O CAS   │  │  - R/O CAS       │
  └─────────────┘  └──────────────┘  └──────────────────┘
```

**Why separate processes?** Solana's Agave runtime is massive and different validator versions have conflicting dependencies. By running each version as its own binary with its own `Cargo.lock`, BlockEnvy avoids dependency hell and can test against multiple runtime versions simultaneously.

**Why sandboxed?** Workers execute arbitrary transaction logic. Linux namespaces + seccomp ensure they can't access the network, can only read cached data, and have strict memory/CPU/FD limits.

### Provider-Agnostic Data Layer

BlockEnvy is designed to work with **any** Solana archival RPC provider and **any** snapshot source. The data layer is a simple interface:

| Data needed | What provides it | Pluggable? |
|---|---|---|
| Historical block data | Any Solana archival RPC (`getBlock`, `getTransaction`) | Yes — configure `data.rpc_url` |
| Snapshot archives | Any S3/GCS bucket with Agave-format snapshots | Yes — configure `data.snapshot_bucket_url` |
| Real-time telemetry | Any gRPC streaming service (optional) | Yes — or use synthetic fallback |

This means BlockEnvy can be deployed on top of **any validator's infrastructure**. The RPC provider supplies the archival data and snapshots they already store. BlockEnvy turns that raw data into a backtesting product.

### Crate Layout

| Crate | What it does |
|---|---|
| `blockenv-core` | Orchestration, config, CAS store, provenance, reconciliation, reports, sandbox, shutdown, traces, CI gate, fixture types |
| `blockenv-data` | Archival RPC client, gRPC streaming client, snapshot store client |
| `blockenv-model` | Network QoS model, lock-aware scheduler, ghost flow, MEV actor, private lane, Monte Carlo runner |
| `blockenv-amm` | AMM adapter trait, Raydium CPMM off-chain math, validation gate, lite trial execution |
| `blockenv-cli` | CLI binary with all subcommands |
| `blockenv-fixture-gen` | Golden fixture generator for CI testing |
| `workers/blockenv-exec-agave-*/` | Agave worker binaries (separate workspaces, one per runtime version) |

---

## The Simulation Models

What makes BlockEnvy's backtesting realistic is five stochastic models that capture the real dynamics of Solana block production:

### Network QoS Model
Solana transactions travel through a 2-lane system:
- **Staked lane** (validators with stake): ~40ms median latency, 0.5% drop rate
- **Public lane** (everyone else): ~200ms median latency, 5% drop rate

Each lane uses a log-normal distribution. The model also supports a "Firedancer-like" mode with 40% less latency and half the jitter — useful for backtesting how the network transition affects outcomes.

### Lock-Aware Fee-Priority Scheduler
Transactions that touch the same accounts create lock conflicts. The scheduler models this realistically:
- Write locks exclude everything; multiple reads can coexist
- Priority = `fee_per_compute_unit + aging_bonus` (aging prevents starvation)
- Micro-ties (arrivals within 50 microseconds) are broken by fee, then arrival time, then signature

### Ghost Flow
Real blocks contain transactions you can't observe in advance. Ghost flow models this unseen activity as a Poisson process (default: 0.6 synthetic txs per slot) with random account locks and compute costs. This prevents backtests from being unrealistically optimistic.

### MEV Actor Model
Models the existing market dynamics of MEV extraction on Solana. Each trial has a configurable probability that the focus trade is sandwiched — a realistic scenario observed on-chain today. The model generates synthetic MEV transactions with configurable parameters (input fraction, tip amount) delivered via private lane (bundle semantics). This lets backtests measure how MEV activity affects your transaction's expected outcome distribution.

### Private Lane
Models private ordering planes (like bundle services). A configurable share of block capacity (default 15%) is reserved for FIFO bundles that bypass the public arrival queue.

---

## Setup Guide

### Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| **Rust** | 1.86.0+ | Building BlockEnvy |
| **Cargo** | (comes with Rust) | Package manager |
| **Linux** | Any modern kernel | Full sandbox support (macOS works but without sandbox) |
| **just** | Any version | Build orchestration (optional, you can use cargo directly) |

### Step 1: Clone and Build

```bash
git clone <this-repo>
cd v10-BlockEnvy

# Build the orchestrator + shared crates
cargo build --release

# Verify it works
./target/release/blockenv --help
./target/release/blockenv --version
```

This produces two binaries:
- `target/release/blockenv` — the main CLI
- `target/release/blockenv-fixture-gen` — the test fixture generator

### Step 2: Build the Worker Binaries

The workers depend on the full Agave validator crate tree. This is a separate build step and takes **15-30 minutes** the first time.

```bash
# Agave 3.1.8 worker (default)
cd workers/blockenv-exec-agave-3-1-8
cargo build --release
cd ../..

# Agave 3.0.14 worker (fallback)
cd workers/blockenv-exec-agave-3-0-14
cargo build --release
cd ../..
```

Install all binaries:

```bash
# Using just (optional)
just install /usr/local

# Or manually:
sudo install -m 755 target/release/blockenv /usr/local/bin/
sudo install -m 755 target/release/blockenv-fixture-gen /usr/local/bin/
sudo mkdir -p /usr/local/libexec/blockenv
sudo install -m 755 workers/blockenv-exec-agave-3-1-8/target/release/blockenv-exec-agave-3-1-8 /usr/local/libexec/blockenv/
sudo install -m 755 workers/blockenv-exec-agave-3-0-14/target/release/blockenv-exec-agave-3-0-14 /usr/local/libexec/blockenv/
```

### Step 3: Connect Your Data Sources

BlockEnvy needs two things from the outside world — **archival RPC access** and **snapshot archives**. These can come from any provider.

#### A. Archival RPC Endpoint (Required)

Any Solana RPC provider that supports `getBlock` with full transaction details and `getTransaction` for individual lookups.

```bash
# Set your RPC provider's API key
export BLOCKENV_RPC_API_KEY="your-api-key-here"
```

This key is used for:
- **HTTP RPC** (`getBlock`, `getTransaction`) — appended as `?api-key=` query parameter
- **gRPC streaming** (real-time arrival times, if available) — sent as `x-token` header

If your provider doesn't offer gRPC telemetry streaming, BlockEnvy falls back to synthetic arrival timing automatically. No configuration change needed.

#### B. Snapshot Archive Store (Required)

BlockEnvy needs Agave-format snapshot archives to reconstruct historical bank state. You need an S3 or GCS bucket containing snapshots from a Solana validator.

**Bucket layout:**
```
your-bucket/
  snapshots/
    manifest.json                    # index of available snapshots
    full/
      snapshot-280000000-<hash>.tar.zst
      snapshot-280025000-<hash>.tar.zst
      ...
    incremental/
      incremental-snapshot-280000000-280010000-<hash>.tar.zst
      ...
```

**`manifest.json` format:**
```json
[
  {
    "type": "full",
    "slot": 280000000,
    "hash": "abc123...",
    "size_bytes": 52428800000
  },
  {
    "type": "incremental",
    "slot": 280010000,
    "base_slot": 280000000,
    "hash": "def456...",
    "size_bytes": 1073741824
  }
]
```

**Authentication (pick one):**

For **S3**:
```bash
export AWS_ACCESS_KEY_ID="AKIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"
```

For **GCS**:
```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
```

**Key constraint:** BlockEnvy needs a snapshot within **25,000 slots** (~2.7 hours) of your target slot.

#### C. Disk Space

The CAS (content-addressed storage) cache stores downloaded snapshots, blocks, and run outputs. Default limit: 200 GB.

### Step 4: Create Configuration File

Create `blockenv.toml` in your working directory:

```toml
# ── Data Sources ──
[data]
rpc_url = "https://your-rpc-provider.com"        # any Solana archival RPC
snapshot_bucket_url = "s3://your-snapshot-bucket"  # or gs:// for GCS

# ── Cache ──
[cache]
root = "/data/blockenv/cas"  # where all data is stored on disk
max_gb = 200                 # auto-evict old data beyond this

# ── Runtime ──
[runtime]
default = "3.1.8"            # which Agave version to use

# ── Monte Carlo ──
[mc]
trials = 1000                # number of backtest trials per run
ci_target = 0.02             # early stopping: stop when 95% CI half-width <= 2%

# ── Network Model ──
[network]
staked_p50_ms = 40           # staked lane median latency
public_p50_ms = 200          # public lane median latency
jitter_ms = 200              # per-lane jitter (p95 - p50)
staked_drop_rate = 0.005     # 0.5% drop probability
public_drop_rate = 0.05      # 5% drop probability

# ── Scheduler ──
[scheduler]
block_cu_cap = 48000000      # 48M CU per block
lock_aware = true

# ── Ghost Flow ──
[ghost]
lambda_per_slot = 0.6        # Poisson rate of synthetic transactions

# ── MEV ──
[mev]
front_run_probability = 0.05      # 5% per trial
front_run_input_fraction = 0.5    # 50% of focus input amount

# ── Private Lane ──
[private_lane]
enabled = true
share = 0.15                 # 15% of block CU reserved for bundles

# ── Security ──
[security]
worker_sandbox = true        # Linux namespaces + seccomp (auto-disabled on macOS)

# ── Batch ──
[batch]
max_concurrency = 4
```

All values above are the defaults — you only need to specify values you want to change.

### Step 5: Run a Backtest

```bash
# 1. Fetch data for a specific slot
blockenv fetch --slot 280000000 --runtime 3.1.8 --focus <your-tx-signature>

# 2. Verify deterministic replay (proves the engine is correct for this slot)
blockenv replay --slot 280000000 --runtime 3.1.8 \
  --full-snapshot-digest <sha256> \
  --snapshot-slot 279990000 \
  --block-digests "280000000:<sha256>"

# 3. Run Monte Carlo backtest
blockenv shadow-run --slot 280000000 --runtime 3.1.8 \
  --focus <your-tx-signature> \
  --trials 1000 \
  --full-snapshot-digest <sha256> \
  --snapshot-slot 279990000 \
  --block-digests "280000000:<sha256>"

# 4. Generate human-readable report
blockenv report --run-id <run-id> --format markdown

# 5. Verify run integrity
blockenv verify --run-id <run-id>
```

---

## Understanding the Output

### Reconciliation Report (from `replay`)

After deterministic replay, BlockEnvy compares every transaction's result against the archival record:

```
Reconciliation: 847/847 matched, focus=matched

Field comparison (per transaction):
  Required (hard error if mismatch):  err, fee, preBalances, postBalances
  Optional (compare if present):      preTokenBalances, postTokenBalances,
                                      logMessages, innerInstructions,
                                      computeUnitsConsumed, ...
```

If everything matches, you know the backtest engine is producing correct results for this slot.

### Backtest Report (from `shadow-run`)

After Monte Carlo backtesting, you get:

```
Trials completed: 1000/1000
Inclusion probability: 94.2% (95% CI: [92.1%, 96.3%])

Focus trade slippage distribution:
  p5:  -127 token units  (worst 5% of outcomes)
  p50: -42 token units   (median outcome)
  p95: +18 token units   (best 5% of outcomes)
  p99: +89 token units

Attribution breakdown:
  Propagation/QoS:  43% of variance
  Lock contention:  28% of variance
  Private lane/MEV: 22% of variance
  CU cap:           7% of variance

Why codes:
  included: 942 trials
  outbid:    31 trials
  blocked:   18 trials
  dropped:    9 trials
```

This tells you: across 1,000 simulated scenarios, the focus transaction was included 94.2% of the time with a median slippage of 42 token units. The biggest factor affecting outcomes was network propagation, not MEV.

---

## CLI Reference

| Command | Purpose |
|---|---|
| `blockenv fetch` | Download snapshot archives + block data into local cache |
| `blockenv build-bank` | Load a snapshot and reconstruct bank state in a worker |
| `blockenv replay` | Deterministic replay of target slot with reconciliation |
| `blockenv shadow-run` | Monte Carlo backtest (auto-triggers replay if needed) |
| `blockenv report` | Generate JSON or Markdown report for a completed run |
| `blockenv verify` | Verify cached data integrity, manifest hashes, reconciliation invariants |
| `blockenv run-batch` | Backtest multiple slots from an NDJSON file |
| `blockenv-fixture-gen` | Generate the Trade #18 golden test fixture |

Run any command with `--help` for full argument details.

### Batch Mode

For backtesting multiple slots at once, create an NDJSON file:

```json
{"slot": 280000000, "runtime": "3.1.8", "focus": "5vGkz...sig1"}
{"slot": 280000001, "runtime": "3.1.8", "focus": "3xPqR...sig2"}
{"slot": 280000002, "runtime": "3.1.8"}
```

```bash
blockenv run-batch --input slots.ndjson --max-concurrency 4
```

Produces `batch_summary.json` with per-slot status. Failed slots don't block the rest.

---

## The Commercial Angle

BlockEnvy is designed to be **deployed by infrastructure providers as a premium product offering**. The business model is simple:

**RPC providers and validators already have the data.** They store archival blocks. They maintain snapshot archives. They operate the infrastructure. What they _don't_ have is a way to turn that data into actionable backtesting intelligence for their customers.

BlockEnvy is the missing layer:

```
┌─────────────────────────────────────────────────────┐
│              YOUR EXISTING INFRASTRUCTURE            │
│                                                     │
│  Archival RPC ──────┐                               │
│                     │     ┌──────────────────┐      │
│  Snapshot Store ────┼────▶│    BlockEnvy     │──── Backtest Reports
│                     │     │                  │      (JSON / Markdown)
│  gRPC Streaming ────┘     │  Replay + Monte  │      │
│  (optional)               │  Carlo Engine    │      │
│                           └──────────────────┘      │
│                                                     │
│  You already pay for this    This is what you sell  │
└─────────────────────────────────────────────────────┘
```

- **White-label it** under your own brand as a "Transaction Backtesting" or "Execution Quality" product
- **Charge per-backtest** or include it in premium RPC tiers
- **Differentiate from competitors** who can only offer raw RPC data
- **The data moat works in your favor** — backtesting requires archival data and snapshots that only infrastructure operators have

The engine is provider-agnostic by design. Any validator or RPC provider can deploy it against their own data. The first one to market owns the narrative.

---

## Testing

```bash
# Run all 238 unit tests
cargo test

# Run tests for a specific crate
cargo test -p blockenv-core      # 110 tests: CAS, config, provenance, reconciliation, reports, sandbox, traces, fixtures, CI gate
cargo test -p blockenv-amm        # 38 tests: Raydium CPMM math, validation gate, lite trials
cargo test -p blockenv-model      # 61 tests: network, scheduler, ghost, MEV, private lane, Monte Carlo
cargo test -p blockenv-data       # 16 tests: archival RPC client, streaming, snapshot store
cargo test -p blockenv-cli        # 4 tests: batch parsing and serialization
cargo test -p blockenv-fixture-gen # 9 tests: keypair generation, hashing, CLI parsing

# Test the fixture generator (no external tools needed)
./target/release/blockenv-fixture-gen --dry-run
```

### The Trade #18 Golden Fixture

BlockEnvy includes a CI test fixture called "Trade #18" — a synthetic slot at slot 1001 with exactly 18 non-vote transactions designed to exercise every part of the engine:

| Transactions | Purpose |
|---|---|
| 1-6 | Account setup (create and fund token accounts) |
| 7-14 | Pool swaps (create price movement before the focus trade) |
| 15-17 | Lock contention (stress scheduler and compute limits) |
| **18** | **Focus trade: Raydium CPMM swap (wSOL/USDC)** |

The CI gate requires:
- **Deterministic parity**: every replayed field matches archival data exactly
- **Stochastic band regression**: p5/p50/p95 from 100K trials match stored expected values at integer-exact precision
- **AMM validation**: off-chain math exactly matches Agave-executed result

---

## Performance

| Metric | Target | Notes |
|---|---|---|
| Deterministic replay | <= 30 seconds | On 16-core, after bank build |
| Lite trials | >= 50/sec | Off-chain AMM math, no Agave execution |
| Full trials | >= 10/sec | Full Agave execution per trial |
| Scaling | `T = T_bank + T_replay + (trials / cores) * T_trial + IO` | Linear in trials, parallelized across cores |

---

## Project Stats

- **15,500+ lines** of Rust across **55 source files**
- **238 unit tests** across 6 crates
- **Zero warnings** on build
- Full specification: 1,083 lines

---

## Glossary

| Term | Definition |
|---|---|
| **Slot** | A ~400ms block production window on Solana |
| **Backtest** | Replaying a historical slot under varied conditions to measure outcome distributions |
| **CAS** | Content-Addressed Storage — files keyed by SHA-256 of their contents |
| **Agave** | The Solana validator client (formerly solana-labs/solana) |
| **Lite Trial** | A fast backtest trial using off-chain AMM math instead of full Agave execution |
| **Full Trial** | A backtest trial that executes all transactions through the real Agave runtime |
| **Focus Transaction** | The specific transaction you're backtesting |
| **Ghost Flow** | Synthetic transactions modeling mempool activity not visible in archival data |
| **MEV** | Maximal Extractable Value — profit extracted through transaction ordering |
| **Private Lane** | A bundle submission channel that bypasses public transaction ordering |
| **Reconciliation** | Field-by-field comparison of replayed results vs. archival ground truth |
| **Run ID** | `base32(sha256(manifest.json))` — deterministic, content-addressed identifier for a backtest run |
| **Parity** | Exact match of replayed transaction metadata against archival RPC data |

---

## License

UNLICENSED
