# MISAKA — the proof of work is an LLM inference

**PALW-RC · Post-quantum UTXO Layer 1 · Kaspa-derived BlockDAG · one canonical inference = one block ticket**

[![License: ISC](https://img.shields.io/badge/License-ISC-blue.svg)](https://github.com/MISAKA-BTC/misakas/blob/main/LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.88%2B-orange.svg)](https://github.com/MISAKA-BTC/misakas/blob/main/Cargo.toml)
![Ruleset](https://img.shields.io/badge/ruleset-PALW--RC%20(ADR--0042)-purple.svg)

MISAKA is an independent Layer 1 written in Rust and derived from
[`rusty-kaspa`](https://github.com/kaspanet/rusty-kaspa). Two things make it its own protocol:

1. **PALW — Proof of Advanced LLM Work.** The scarce resource that produces and orders blocks is
   **one deterministic LLM inference per attempt**, not a hash. A block is won by running a model,
   and the network convicts a liar by re-deriving arithmetic — never by re-running the model on
   every node.
2. **Post-quantum native authorization.** Every native transaction is signed with **ML-DSA-87**
   (FIPS 204, NIST category 5) over a 64-byte BLAKE2b-512 consensus identity. Legacy
   secp256k1/Schnorr/ECDSA and P2SH are excluded from the native lane entirely.

Together these form the mainnet-candidate ruleset specified by **ADR-0042** — *one atomic
activation bundle, one fork choice, one fingerprint*. Network rules are machine-checkable through
the ruleset and consensus fingerprints rather than prose. Testnet-11 is the currently launched RC;
the repository's mainnet parameter set is not a launched network.

> [!IMPORTANT]
> **Current status (2026-09-13): Testnet-11 Relaunch 5f is live.** It ships
> `PalwConsensusMode::ConsensusV2` at a frozen 120-second block cadence. Build the current
> [`misakas` `main`](https://github.com/MISAKA-BTC/misakas), select `testnet-11` explicitly and
> verify consensus fingerprint
> **`ae1d61628da50c7becea62f0a8f08c8654d190c60b2e104df0010b121ba4d3d8`**.
> Mainnet parameters are defined but mainnet is not launched. ADR-0123's epoch-budget release is
> implemented but remains dormant on every shipped preset.

> [!NOTE]
> The post-quantum claim covers **native transaction authorization, validator signing, and the
> 64-byte consensus identity**. It does not cover P2P transport, and the optional EVM lane
> deliberately uses Ethereum-compatible secp256k1/ECDSA in a separate signature domain.

---

## Official resources

| Resource | Link |
|---|---|
| Website | [misakachain.com](https://misakachain.com/) |
| Source code | [MISAKA-BTC/misakas](https://github.com/MISAKA-BTC/misakas) |
| Releases | [misakas releases](https://github.com/MISAKA-BTC/misakas/releases) |
| Explorer | [misakascan.com](https://misakascan.com/) |
| Protocol documents | [`docs/`](https://github.com/MISAKA-BTC/misakas/tree/main/docs) · [`docs/adr/`](https://github.com/MISAKA-BTC/misakas/tree/main/docs/adr) |
| Whitepaper / specification | [MISAKA-BTC/specification](https://github.com/MISAKA-BTC/specification) |
| Issue tracker | [GitHub Issues](https://github.com/MISAKA-BTC/misakas/issues) |
| Security policy | [`SECURITY.md`](https://github.com/MISAKA-BTC/misakas/blob/main/SECURITY.md) |
| Discord | [discord.gg/C4nDFkJE4x](https://discord.gg/C4nDFkJE4x) |
| Telegram | [t.me/misakachain](https://t.me/misakachain) |

### Start on the current public testnet

```bash
git clone https://github.com/MISAKA-BTC/misakas.git
cd misakas
git switch main
cargo build --release -p kaspad -p misaka-cli
./target/release/misaka --network testnet-11 mining setup
```

An existing PALW Bond can be checked without its key or class id:

```bash
./target/release/misaka --network testnet-11 bond status --bond <txid>:<index>
```

Use the [Testnet-11 mining runbook](https://github.com/MISAKA-BTC/misakas/blob/main/docs/testnet11-join-mining.md)
and [operator Wiki](https://github.com/MISAKA-BTC/misakas/wiki/Testnet-11-Operator-UI-JA).
Commands for retired testnet-10, testnet-21, testnet-200 or an older Testnet-11 relaunch are not
current operator instructions.

---

## PALW in one page

### The lottery: a ticket, not a hash

```text
challenge = H(network ‖ pre_pow_hash ‖ timestamp ‖ nonce ‖ class_id ‖ bond)
              │
              ▼
        ONE deterministic inference on a pinned execution class
              │
              ▼
        execution commitment  ──▶  attempt envelope  ──▶  digest < bits ?
```

One inference is one ticket: header-bound, non-transferable, progress-free. Re-rolling any chain
hash therefore costs a full inference — which is exactly the property every downstream randomness
consumer (panel draws, ticket seeds, anchors) leans on.

Hashing is retained for block and transaction identity, Merkle commitments, artifact pinning and
signature-input compression. PALW inference is the economic block-production right and work unit.
The current ruleset also carries a bounded, fee-only bondless heartbeat lane to keep the DAA clock
moving when bonded producers are silent; heartbeat hash work does not become PALW model work or a
replacement economic mining lane.

### The verification: a full node runs no model

This is the decision that makes LLM work usable as L1 work. Under ADR-0042 Decision 4 the consensus
build carries **no model dependency at all** — a node that registers no PALW runtime prices an
inference tag as *failed PoW*, it does not panic and it does not load a 1 GiB runtime to validate a
header. Validation is optimistic and sampled: a block is admitted on its commitments, a sortitioned
bonded **panel** audits it, and a dispute is settled by an **arithmetic court** that bisects the
execution trace down to a single tile and adjudicates that tile exactly.

```text
attempt ─▶ Provisional ─▶ PanelBound ─▶ ReceiptLicensed ─▶ Final
                 │                             │
                 └──────── refuted ────────────┴──▶ Voided (bond slashed)
```

| Block state | live weight | safe weight |
|---|---|---|
| Provisional / PanelBound / ReceiptLicensed | bounded immature pwu (β·pwu, β ≤ 1000‰) | 0 |
| Final | full pwu | full pwu |
| Voided | 0 | 0 |

A fresh tip is always weighable (so the chain never stalls waiting for an audit), while nothing
irreversible — reward spendability included — happens before `Final`. Both weights come from **one
fork-choice authority**: virtual selection, header processing, IBD, pruning and finality all ask
the same function, so no two subsystems can pick different tips on one DAG.

### What the court can prove

The court is **proof-carrying**. A refuter supplies the operands it claims were mis-computed, and
those operands are verified against the class's `artifact_root` — so an adjudicating node holds the
model's *root*, never its weights. Court cost is therefore **independent of model size**: a bigger
model costs one terminal adjudication more arithmetic, not every node more storage. Conviction is
what stands between a fabricated block and full weight, so a class may carry **no fork-choice
weight until its kernel catalog is complete** and every operation it can reach is adjudicable.

### Cadence and identity

PALW is frozen at **120 s per block**. Every window in the ruleset is DAA-denominated, which is why
a PALW network needs its own identity rather than a parameter edit: at a 100 ms cadence the same
numbers mean something else, and `finality_depth < w_challenge` fails on depth alone.
`assemble_palw_rc_identity_v2` refuses to mint an identity unless five gates agree — the bundle is
runnable, the catalog matches what the ruleset committed to, the cadence and fences are right, the
genesis objects actually apply (the first transition runs and its state root exists), and the
court's ladder is provisioned for the **whole step space** rather than for the classes this genesis
happens to carry.

---

## PALW work sources

The current bundle separates economic inference work from its bounded liveness heartbeat.

| Algo id | Lane | What wins the block |
|---:|---|---|
| `3` | **Heartbeat** | bounded fee-only liveness work; not PALW model work |
| `6` | **Attempt** (`PalwAttemptEnvelopeV2`) | a nonce-driven canonical inference whose digest clears `bits` |
| `7` | **Receipt** (**ADR-0044**) | a certified receipt from an inference a **user actually wanted** |

### The free-prompt lane — your own chat inference mines

```text
your app ──POST /v1/chat/completions──▶ misaka-palw-gateway ──▶ pinned worker
                                             │                      │
                                    OpenAI-style reply        ONE inference:
                                    + roots + CU in-band      answer + trace/output/schedule roots
```

You run your own model for your own work — code review, drafting, summarizing — and **that same
single inference** is what mines. The chain never assigns the prompt, there is no second
mining-only run, a receipt is usable exactly once, and block weight is never revised after
acceptance. Certification runs through the same claim lattice the attempt lane uses: commit the
trace, draw an audit panel from randomness that becomes known only *after* the commitment, certify,
then let the certified receipt win a ticket. Two grinding surfaces in the naive construction are
closed structurally — only attempt blocks carry beacon randomness, and tickets are quantized so an
executor cannot shape a free field the inference did not consume. Pricing (CU) is derived from the
executed shape, conservatively, and never from a self-declared number.

Full nodes admit a receipt block **with no model**, exactly as they admit an attempt block.

---

## `PALW-BASE-0` — the integer-only class

The model-compute floor is a **class**: a portable, integer-only execution class held permanently
Active (**ADR-0039**, **ADR-0040**). It is distinct from the narrow heartbeat lane: BASE-0 provides
the cheapest valid inference class, while heartbeat provides no model share or inference reward.

| | |
|---|---|
| Arithmetic | no IEEE-754 value, no `libm` symbol, no FP instruction on the consensus path |
| Representation | weights `int8` (per-output-channel scale), activations `int8`, accumulator `i32` |
| Requantize | an **explicit** op with `(multiplier: i32, shift: u8)` — never an implicit narrowing |
| Scales | frozen at registration; no per-inference rescaling anywhere |
| Verification | a second implementation with exact `i128` division and no shift operator, plus vendored **gemmlowp** as an authorship-independent oracle, differenced bit-for-bit |

Removing floating point removes the entire category of divergence that a float class would have to
transcribe away — glibc `expf`/`logf`/`sinf`/`cosf`, FMA contraction, and the reduction order of
every threaded sum. An integer class is why the court is reachable at all.

---

## The class economy is chain state

Registering a second model must be a **transaction, not a release** (**ADR-0045**):

```text
pwu     — derived, per candidate point:  claim == palw_pwu_v1(class_target, pwu_per_inference)
budget  — derived, per epoch boundary:   ⌊tol · E · s_c / (1000 · denom_c)⌋ blocks, frozen in state
shares  — granted, per registration:     conserved to 1000‰ by donation arithmetic, rooted
```

- **pwu has exactly one legal value.** It is the counted step-leaf count of the class's canonical
  inference, checked as equality against rooted chain state — not a self-declared number under a
  ceiling. Anything else is `PwuClaimNotDerived`.
- **Each class gets its own DAA retarget domain**, so a fast class cannot starve a slow one and a
  heavy class is not priced in a currency that structurally starves it.
- **A share is granted at registration** and conserved to 1000‰ by largest-remainder donation from
  the incumbents. A share below the minimum grantable value is refused, because a zero share is a
  zero epoch budget — "a dead class registered as if it worked".
- **Unused-budget release is not active yet.** ADR-0123 implements progressive borrowing from
  silent classes, but every shipped preset currently leaves `palw_epoch_budget_release` unset.
  The implementation is therefore not a claim about current Testnet-11 throughput until a network
  deliberately schedules that fence.
- **Post-genesis admission exists.** `verify_class_admission_v2` restates the genesis loader's
  checks against a single registration, *deriving* rather than reading — the reachable kernel set
  from the profile's own nodes, both leaf counts from the step function, and a catalog entry
  identical to the one the genesis path produces. A registrant supplies only what no function can
  invent: the artifact root, the economics, and the canonical job.

---

## Model roadmap — from the floor to 35B and 70B

The end state is explicit: **MISAKA is intended to carry large models — 35B- and 70B-class — as
registered execution classes**, so that the work securing the chain is inference people would want
anyway. That is a roadmap, and the sections below separate what is measured today from what must
still land.

### Where the ladder stands

| Stage | Geometry | Status |
|---|---|---|
| **Floor** — `PALW_RC_BASE0_GEOMETRY` | 4 layers, `d_model` 256 | ships as the RC's liveness floor. Not a performance claim: it is the class that guarantees *someone* can always produce a block |
| **A16** | Qwen2.5-1.5B, graph-v5@512 | registered on Testnet-11; producer/panel requires the class-bound `.palwart` and tokenizer binding |
| **QWEN36** | Qwen3.6-35B-A3B | registered on Testnet-11; producer/panel requires the class-bound `.palwq36` runtime |
| **larger/new families** | new canonical profiles | roadmap; each must pass admission, court coverage and certification before it receives work |

### Why large classes are architecturally reachable

Four properties were built with exactly this in mind, and none of them degrades with model size:

1. **Full nodes never run the model.** Verification is sampled and optimistic, so a 70B class does
   not put a 70B runtime on every node.
2. **The court holds a root, not the weights.** Proven operands are verified against
   `artifact_root`, so adjudication cost is independent of parameter count.
3. **Adding a class is a chain event.** Registration is admitted against derived facts, given a
   granted share and its own DAA domain — no flag day, no coordinated release.
4. **The court ladder was provisioned once, at genesis, for the whole step space.**
   `max_step_leaf_count` is inside the ruleset id, so a class deeper than the ladder would need a
   *new network*. The RC therefore refuses to mint an identity whose ladder is anything other than
   the full `PALW_STEP_MAX_LEAVES` (2²², a 22-round bisection). Four extra prosecution rounds buy
   every class that could ever be adjudicable — this is the one decision that would have expired.

### What a new class still has to prove

Testnet-11's A16 and QWEN36 rows have moved beyond the earlier prototype limitations: they are
registered classes with pinned profiles and artifacts. That does not make an arbitrary model
admissible. Every new row still needs a deterministic artifact and shape profile, complete
reachable-kernel adjudication, a worst-case job that fits the court/transport bounds, derived PWU,
tokenizer/runtime binding, independent conformance evidence, lane certification and a granted
share. Hardware-specific acceleration is introduced as an explicitly pinned class or execution
regime, never as a silent widening of an existing class.

The current operator entry point is `misaka model add`; the full process is documented in
[`palw-model-onboarding-sdk.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/palw-model-onboarding-sdk.md).

---

## The post-quantum native lane

| Area | MISAKA |
|---|---|
| Transaction signature | **ML-DSA-87** (pk 2592 B / sig 4627 B); secp256k1/Schnorr/ECDSA disabled in native consensus |
| Sighash | `calc_mldsa87_signature_hash` → 64-byte `Hash64` |
| Address | `PubKeyHashMlDsa87` only; payload = keyed BLAKE2b-512 over the verifying key, 64 B |
| Standard script | ML-DSA-87 P2PKH only; P2SH disabled |
| Consensus identity | 64-byte BLAKE2b-512 (`Hash64`) — block hash, txid, Merkle roots, UTXO commitment, parents |
| Build hygiene | `scripts/pq-ci-guard.sh` hard-gates that neither `kaspa-consensus` nor `kaspad` links secp256k1 |
| Supply | **28 B MSK cap** — 13 B genesis allocation + 15 B emission over 20 years (5 %/yr exponential decay) |

Accurate claims: native authorization uses ML-DSA-87; native secp256k1/Schnorr/ECDSA is disabled;
validator and court signing use ML-DSA-87; consensus identity is 64-byte BLAKE2b-512-based.

Claims that would be **wrong**: "all network traffic is post-quantum encrypted"; "the EVM lane is
post-quantum"; "MISAKA is a pure PoS chain"; "MISAKA uses Narwhal/Bullshark"; "LLM compute is a
reward subsidy on top of hash PoW" — under ADR-0038/0039 the inference *is* the consensus work.

### Ecosystem token note

The Solana SPL mint used by the broader MISAKA ecosystem is a separate market representation. It is
**not** a native MISAKA UTXO address, **not** an EVM contract on any MISAKA network, and **not**
block reward.

| Resource | Value |
|---|---|
| SPL mint | `4e2DhohUAJ9EbrLey3rVVgFQzLCAeeBirSbdhqrh9snX` |
| Market page | [DEX Screener](https://dexscreener.com/solana/hvcuswpugjg8omexyaexjs7wxf8izdwqgljqyxfehhqc) |

Always verify token information through the official website before interacting with a mint.

## Optional EVM lane

The `kaspad` crate's default feature set includes EVM support. EVM execution is a separate domain
from PQ-native authorization: Ethereum-compatible accounts retain secp256k1/ECDSA semantics and
operators may choose whether to expose EVM RPC/history roles.

| Property | Value |
|---|---|
| Implementation / revision | [`revm`](https://github.com/bluealloy/revm), Shanghai |
| Chain ID | `0x4D534B` (`5067595`) |
| Envelopes | Legacy, EIP-2930, EIP-1559 |
| JSON-RPC | HTTP on `--evm-rpc-listen` |
| Execution model | Selected-parent chain, mergeset-delayed acceptance |
| Native-unit bridge | `1 sompi = 10^10 wei` |

Ethereum **execution**-compatible, not Ethereum-consensus-compatible: no Beacon Chain, Engine API,
devp2p or Ethereum PoS. See
[`docs/misaka-evm-design-v0.4.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/misaka-evm-design-v0.4.md).

---

## Build from source

### Requirements

- Rust **1.88+**, Git, a C/C++ toolchain
- `protoc` (gRPC), Clang / libclang (RocksDB), OpenSSL headers + `pkg-config`

```bash
sudo apt update
sudo apt install -y git curl build-essential pkg-config libssl-dev \
  protobuf-compiler libprotobuf-dev clang libclang-dev
rustup update
```

### Node and tools

```bash
git clone https://github.com/MISAKA-BTC/misakas.git
cd misakas

cargo build --release \
  -p kaspad \
  -p kaspa-pq-validator \
  -p kaspa-pq-signer \
  -p misaka-cli --bin misaka
```

The consensus build deliberately carries **no model dependency**. The PALW runtime crates are
separate binaries a producer or auditor runs beside the node.

### PALW producer / auditor side

```bash
# the pinned worker (needs the pinned llama.cpp tree; its build.rs refuses to build blind)
MISAKA_LLAMA_SRC=/path/to/llama.cpp cargo build --release -p misaka-palw-worker

# the free-prompt gateway (ADR-0044): OpenAI-compatible front end, one inference
cargo build --release -p misaka-palw-gateway

# the integer-only class engine and its independent verification lane
cargo build --release -p misaka-palw-base0
cargo test    --release -p misaka-palw-base0-ref2
```

A build's **runtime class** is part of consensus identity, not a preference. Check yours before
producing anything:

```bash
MISAKA_PALW_GGUF=/path/to/model.gguf ./palw-worker --mode manifest
```

If `runtime_class_id` does not match the class you intend to join, you are out of class — your tags
differ from everyone else's, and that looks like a network fault while being the opposite.

## Testing

```bash
cargo test --release          # or: cargo nextest run --release
./check                       # fmt, clippy, deps, and the PQ-only CI guard
```

Consensus, cryptographic, address, activation, court and class-economy changes require
deterministic regression tests. Anything that alters accepted blocks, commitments, state roots,
receipts or activation behaviour is a consensus change and is documented as one.

---

## Repository map

| Path | Purpose |
|---|---|
| `kaspad/` | Full node daemon (runs **no** model) |
| `consensus/` | GHOSTDAG, validation, PALW state machine, court, fork choice, EVM commitments |
| `consensus/core/src/palw_attempt_v2.rs` | Attempt envelope, ticket binding, canonical hash transcript |
| `consensus/core/src/palw_state_v2.rs` | Candidate-scoped PALW chain state, deltas, state root |
| `consensus/core/src/palw_admission_v2.rs` | Bond signature, class, pwu, epoch, per-bond exposure |
| `consensus/core/src/palw_base0*.rs` | `PALW-BASE-0` consensus arithmetic |
| `consensus/core/src/palw_class_admission_v2.rs` | Post-genesis class registration gate |
| `consensus/core/src/palw_class_daa.rs` | Per-class retarget and epoch budgets |
| `consensus/core/src/palw_rc_identity_v2.rs` | The five-gate RC identity assembly |
| `misaka-palw-base0/` | The integer-only execution class: artifacts, engine, rotary table |
| `misaka-palw-base0-ref2/` | Independent re-derivation + vendored gemmlowp oracle (test-only) |
| `misaka-palw-gateway/` | Free-prompt OpenAI-compatible gateway (ADR-0044) |
| `misaka-palw-worker/`, `misaka-palw-agent/` | Pinned runtime and its supervising UDS front |
| `misaka-palw-reexecutor/`, `misaka-palw-shadow/` | Auditor capability emission, drill harness |
| `crypto/`, `wallet/` | Hash64, ML-DSA-87 addresses/scripts, wallet libraries and CLI |
| `kaspa-pq-validator/`, `kaspa-pq-signer/` | Bonded sidecar and isolated signing daemon |
| `kaspa-evm/`, `rpc/eth/` | Feature-gated EVM executor and Ethereum JSON-RPC adapter |
| `docs/` | ADRs, specifications, runbooks, audits, measurements |

## Key protocol documents

| Document | Subject |
|---|---|
| **ADR-0038** (`docs/adr/0038-palw-is-the-consensus-work.md`) | **PALW is the consensus work** — the layer inversion, sampled verification |
| **ADR-0039** (`docs/adr/0039-palw-only-block-production.md`) | A Base class instead of a hash floor; two-weight fork choice |
| **ADR-0040** (`docs/adr/0040-palw-base-0-integer-arithmetic.md`) | `PALW-BASE-0` integer-only arithmetic |
| **ADR-0042** (`docs/adr/0042-palw-mainnet-candidate-ruleset.md`) | **The PALW-RC ruleset** — one bundle, one fork choice, one fingerprint |
| **ADR-0044** (`docs/adr/0044-palw-free-prompt-receipts.md`) | Free-prompt receipts — your own inference mines |
| **ADR-0045** (`docs/adr/0045-palw-class-economy-on-chain.md`) | Derived pwu, epoch budgets, granted share table |
| **ADR-0060 / ADR-0068** | Bounded heartbeat liveness and the LLM-primary economy |
| **ADR-0122** | One operator workflow for mining, work, model and status |
| **ADR-0123** | Progressive unused class-budget release (implemented; dormant on shipped presets) |
| **ADR-0027** (`docs/adr/0027-palw-slash-unilateral-fraud-proofs.md`) / **ADR-0028** (`docs/adr/0028-palw-challenge-sampling-protocol.md`) | Unilateral fraud proofs, challenge sampling |
| **ADR-0030** (`docs/adr/0030-palw-step-function-shape-profile.md`)–**ADR-0033** (`docs/adr/0033-palw-credit-gate-wiring.md`) | Step space, transcendentals, escrow, credit gate |
| **ADR-0041** (`docs/adr/0041-palw-pruning-proof-verification.md`) | Pruning-proof verification |
| [ADR-0019](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0019-mldsa87-migration.md) | ML-DSA-87 migration (governing PQ record) |
| [`docs/kaspa-pq-spec.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/kaspa-pq-spec.md) | Consensus-level PQ specification |

ADR-0026 and later are the PALW lineage: they live in `docs/adr/` on the PALW branches and land
on `main` together with the ruleset they specify. ADR-0019 and earlier are already on `main`.

---

## Contributing

Contributions are welcome from protocol engineers, quantization and inference-runtime specialists,
wallet and RPC developers, auditors and node operators. Read
[`CONTRIBUTING.md`](https://github.com/MISAKA-BTC/misakas/blob/main/CONTRIBUTING.md), open an issue
for design-sensitive changes, and include tests and activation notes in pull requests. Work that
touches an execution class must arrive with its determinism evidence.

## Security

Do not publish suspected vulnerabilities as public issues. Follow
[`SECURITY.md`](https://github.com/MISAKA-BTC/misakas/blob/main/SECURITY.md) and report privately
with the affected component, impact and reproduction steps.

## Upstream attribution

MISAKA is derived from [`rusty-kaspa`](https://github.com/kaspanet/rusty-kaspa) and keeps many
upstream `kaspa-*` crate and binary names to preserve source history. Credit for the original Rust
Kaspa implementation belongs to the Kaspa developers; the MISAKA contributors maintain the
independent network, the post-quantum native path, PALW, and the optional EVM integration.

## License

Distributed under the **ISC License**. See
[`LICENSE`](https://github.com/MISAKA-BTC/misakas/blob/main/LICENSE).

```text
Copyright (c) 2026 MISAKA contributors
Copyright (c) 2022-2024 Kaspa developers
```

---

<sub>MISAKA — one canonical LLM inference is one block ticket, adjudicated by arithmetic, on a
post-quantum UTXO BlockDAG.</sub>
