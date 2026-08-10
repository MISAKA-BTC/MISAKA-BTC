# Hello from MISAKA ⚡️

**Post-quantum-native UTXO Layer 1 · Kaspa-derived PoW BlockDAG · Verified-LLM-Token-weighted BFT finality · Optional EVM execution lane**

[![License: ISC](https://img.shields.io/badge/License-ISC-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.88%2B-orange.svg)](Cargo.toml)
[![Network](https://img.shields.io/badge/network-testnet--10-yellow.svg)](https://misakascan.com)

**misakas** is an independent Layer 1 network written in Rust and derived from
[`rusty-kaspa`](https://github.com/kaspanet/rusty-kaspa). Its native UTXO transaction path is
post-quantum-only: transaction authorization uses **ML-DSA-87** (FIPS 204, NIST security category
5), native addresses and scripts accept only the ML-DSA-87 P2PKH form, and legacy
secp256k1/Schnorr/ECDSA and P2SH paths are excluded from native consensus, mempool, and wallet
operation.

On top of that PoW base, MISAKA is building a finality overlay whose voting power comes from
**independently verified LLM inference** rather than from capital: *Verified LLM Token-weighted BFT*
(ADR-0024). Block production stays PoW/GHOSTDAG — the overlay decides finality, never ordering.

misakas has its own genesis, address namespace, consensus identity, and chain state. It is **not
compatible with Kaspa**, earlier `kaspa-pq` networks, or the older Narwhal/Bullshark-based MISAKA
experiments.

> [!IMPORTANT]
> The public network currently operated by this repository is the experimental **`testnet-10`**
> network. The `mainnet` parameter set exists in the codebase but **no supported misakas mainnet is
> launched**. Do not treat testnet balances, RPC stability, or activation parameters as production
> guarantees.

> [!NOTE]
> The post-quantum claim applies to the **native UTXO authorization path**, validator attestations,
> and 64-byte consensus identity. P2P transport is not currently part of that claim. The optional
> EVM lane intentionally uses Ethereum-compatible secp256k1/ECDSA in a separate signature domain.

---

## Official resources

| Resource | Link |
|---|---|
| Website | [misakachain.com](https://misakachain.com/) |
| Source code | [MISAKA-BTC/misakas](https://github.com/MISAKA-BTC/misakas) |
| Releases | [misakas releases](https://github.com/MISAKA-BTC/misakas/releases) |
| Testnet explorer | [misakascan.com](https://misakascan.com/) |
| Protocol documents | [`docs/`](https://github.com/MISAKA-BTC/misakas/tree/main/docs) |
| Whitepaper / specification | [MISAKA-BTC/specification](https://github.com/MISAKA-BTC/specification) |
| Issue tracker | [GitHub Issues](https://github.com/MISAKA-BTC/misakas/issues) |
| Security policy | [`SECURITY.md`](https://github.com/MISAKA-BTC/misakas/blob/main/SECURITY.md) |
| Discord | [discord.gg/C4nDFkJE4x](https://discord.gg/C4nDFkJE4x) |
| Telegram | [t.me/misakachain](https://t.me/misakachain) |

## Current network status

| Item | Current state |
|---|---|
| Public network | `testnet-10` |
| Base consensus | Kaspa-derived PoW BlockDAG with GHOSTDAG ordering |
| Block production target | 10 DAG blocks per second (100 ms target interval) |
| Native ledger | Transparent UTXO model |
| Native transaction signature | ML-DSA-87 / FIPS 204 |
| Finality extension | Stake-bonded DNS finality overlay on top of PoW (active from genesis) |
| Finality weighting | Bonded stake today; **Verified-LLM-Token (VLT) weighting implemented and shipped dormant** behind two DAA fences |
| Compute-backed token (TOK) | Design draft + Phase A implementation; **not shipped on any network** |
| EVM | Feature-gated, Shanghai-compatible execution lane on testnet |
| Mainnet | Parameters defined; not launched or endorsed for production |

Operators must run matching node and miner builds from the same release whenever a release changes
consensus-committed fields.

## Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│              PoW BlockDAG + GHOSTDAG ordering                   │
│             10 BPS target · Kaspa-derived core                  │
├─────────────────────────────────────────────────────────────────┤
│  DNS finality overlay: bonded ML-DSA-87 validators              │
│  round 1 prevote (attestation shard) → round 2 lock+precommit   │
│  voting weight: bonded stake  →  verified LLM compute (VLT)     │
│  PoW still produces blocks and drives GHOSTDAG selection        │
├───────────────────────────────┬─────────────────────────────────┤
│ Native UTXO lane              │ Optional EVM lane               │
│                               │                                 │
│ ML-DSA-87 only                │ revm / Shanghai                 │
│ Transparent UTXO accounting   │ EIP-155 / EIP-1559              │
│ 64-byte BLAKE2b-512 identity  │ secp256k1/ECDSA domain          │
│ misaka / misakatest addresses │ Selected-parent acceptance      │
├───────────────────────────────┴─────────────────────────────────┤
│ gRPC · wRPC Borsh/JSON · Ethereum HTTP JSON-RPC                 │
└─────────────────────────────────────────────────────────────────┘
```

### Base consensus and native UTXO lane

- **PoW BlockDAG:** block production and tip selection remain PoW/GHOSTDAG rather than PoS or a
  Narwhal/Bullshark BFT committee.
- **ML-DSA-87 authorization:** native transaction and validator signing use ML-DSA-87 with explicit
  domain separation.
- **PQ-only native scripts:** the standard spend path is ML-DSA-87 P2PKH. Legacy native
  secp256k1/Schnorr/ECDSA signatures, legacy addresses, and P2SH are disabled.
- **64-byte consensus identity:** block hashes, transaction IDs, Merkle commitments, parent IDs, and
  UTXO commitments use the misakas `Hash64` design based on BLAKE2b-512.
- **Independent network:** native address prefixes are `misaka:`, `misakatest:`, `misakasim:`, and
  `misakadev:` according to network.
- **Layer-0 PoW:** the bundled PQ miner grinds the BLAKE2b-512 Layer-0 proof-of-work path.

### DNS finality overlay

misakas adds a stake-bonded validator overlay without replacing PoW block production. Validators
lock native UTXOs (production floor: **20,000,000 MSK**), sign epoch attestations with ML-DSA-87,
and contribute to stake-confirmed canonical anchors. On the `testnet`/`mainnet` parameter sets
confirmation is **two-dimensional**: an anchor needs both accumulated blue work depth and attested
stake depth. The overlay adds reorg protection, validator rewards, slashing, and anti-equivocation
state while GHOSTDAG and blue work remain the underlying ordering mechanism.

Finality is a **two-round accountable commit**, not a single tally:

1. **Prevote** — the existing attestation shard (unchanged on the wire).
2. **Lock + precommit** — signed only for an epoch whose prevote quorum this chain already shows,
   and carrying the `(locked_epoch, locked_hash)` the signer held, inside the signed digest.

An anchor is DNS-confirmed only when **both** rounds clear quorum over the same pinned weight table.
Two precommits naming one `locked_epoch` with different anchors are self-contained evidence that the
signer held two locks at one height — and burn the bond. Locks are read from the chain, never from
local memory, so a restarted or restored node restates the lock everyone else already holds a
signature for.

Validator operation is documented in
[`docs/validator-runbook.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/validator-runbook.md).
A separate [`kaspa-pq-signer`](https://github.com/MISAKA-BTC/misakas/tree/main/kaspa-pq-signer)
daemon can keep the validator key outside the validator process, enforce a signing policy, and
guard against equivocation through a protected Unix-domain socket.

---

## Verified LLM Token-weighted BFT (VLT)

> Status: **implemented, shipped dormant.** Both activation fences are `u64::MAX` on every shipped
> preset, so every live network — including `testnet-10` — still weighs finality votes by bonded
> stake and is byte-identical to its pre-VLT behaviour. Governing record:
> [ADR-0024](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0024-verified-llm-token-weighted-bft.md).

Ordinary proof-of-stake maps *money → power*. VLT replaces the **source** of that power with
**verified useful compute**: a validator's voting weight is the amount of independently re-executed
LLM inference it recently supplied. The bond stops *being* the power and starts *collateralizing*
it — it remains the participation floor and the cap on how much compute weight one identity may
carry.

### How weight is computed

```text
x_j     = ρ(S_j)·(a·t_j^in + b·t_j^out)   if Verify(S_j, R_j, C_j) = 1, else 0
X_i(e)  = Σ_j x_j                          validator i's certified jobs in epoch e
C_i(E)  = Σ_{τ=1..K} d_τ · X_i(E − τ)      decayed credit window, 1 = d_1 ≥ … ≥ d_K > 0
W_i(E)  = min{ C_i(E), λ·B_i(E) }          compute, capped by bonded collateral
W(E)    = Σ_i W_i(E)                       Q(E) = ⌊2·W(E)/3⌋ + 1
```

- `t_in` / `t_out` are prefill and decode tokens; `a`, `b` weight them (decode is bandwidth-bound
  and costs more). `ρ(S_j)` is the model cost factor — a **consensus parameter**, never an executor
  input, so nobody can invent a fictitious expensive model. An unregistered model mints zero.
- Credit **decays**: stopped hardware loses voting power within a window (shipped calibration:
  K = 96 epochs, 0.97/epoch, half-life ≈ 23 epochs).
- Epoch credit becomes **binary on the exact BFT threshold** `Q(E) = ⌊2W(E)/3⌋ + 1`, restoring the
  quorum-intersection safety argument that a graded stake score could not support.
- Buying stake buys no votes: `C_i = 0 ⇒ W_i = 0` regardless of bond size.

### How compute is verified

The job lifecycle runs entirely as overlay transactions on subnetwork ids `0x14`–`0x1a`:

```text
LlmJobSpec ─▶ Commitment(0x17) ─▶ sortitioned verifier committee ─▶ Certificate(0x14)
                                        Verdicts(0x18) / Challenge(0x15)
                                        └─ survives the challenge window ─▶ credit
```

- **`CanonicalFullReplay`** is the only consensus-eligible verification relation in v0.1: the
  JobSpec pins model weights, runtime, quantization, input, sampling seed, and token limit, so an
  honest verifier must reproduce the executor's receipt byte-for-byte. Committee: 3 drawn,
  2 confirmations.
- Acceptance is **refutation-dominant** — one `Refuted` verdict fails the job even if the
  confirmation count is met; the challenge path then decides who is slashed.
- Verifiers are bonded, independently sortitioned, and must not be the executor. Auditing is paid;
  executing is not — executing is already self-interested, since it buys voting weight.
- Consensus only ever checks commitments, never tensors. Running the model as executor or verifier
  is node-side software outside the consensus surface.
- A bond may only be slashed at acceptance by an offence **provable from the transaction itself**
  (e.g. contradictory verification, equivocation, a broken precommit lock). Unprovable claims deny
  a certificate its credit but never burn a bond.

### Why the denominator is pinned

`Q(E) = ⌊2W(E)/3⌋ + 1` is a two-thirds threshold only if every branch arguing about epoch `E`
divides by the same `W(E)`. So weights come from a **`VltEpochSnapshot`** — a credit table plus the
block it was taken at. The reorg gate builds exactly one, pinned at the selected-chain common
ancestor of the two branches, and hands the same one to both; a bond or a certificate that exists
only above the pin weighs zero on both sides. Votes sign a commitment to the frozen snapshot, so a
vote counted under one denominator cannot be replayed under another.

### Two-fence activation

Turning the overlay on and handing it the vote are different risks, so they are different hard forks:

| Fence | At and above it | Finality |
|---|---|---|
| `vlt_shadow_activation_daa_score` | certificates credited, committees drawn, verdicts counted and paid, settled challenges slashing, credit accumulator filling | unchanged — bonded stake |
| `vlt_activation_daa_score` | `W_i(E) = min{C_i(E), λ·B_i(E)}` becomes voting weight; credit rule becomes `Q(E)` | replaced |

The interval between them is not slack, it is the **soak**: `C_i(E)` sums a credit window, so
flipping both at once would hand voting power to an empty table and stall finality. A pre-flight
check (`vlt_params_consistent()`) refuses a preset whose fences are closer together than the window
it takes for compute credit to mean anything, and a network that fails it stays in Bootstrap with
the reorg gate dormant rather than arming a gate over a denominator that has not filled.

Rollout is four evidenced steps: five-validator private devnet → shadow mode → testnet shadow fork
→ testnet weight fork; mainnet repeats the last two with its own heights.

### Verified so far

On a five-validator private devnet, with quota plan 8/5/3/2/2:

- Frozen voting snapshots carried the intended weights exactly (400/250/150/100/100 M µRTE,
  `W = 1e9`, `Q = 666,666,667`), root-identical on all five nodes and stable across restarts.
- Quorum behaves as specified: 600 and 650 weight units of signers never met quorum; 800 and 750
  certified, with strictly ascending certificate epochs recorded in a durable
  `DnsFinalityCertificate`.
- Equivocation on prevote, precommit and lock each burned the offender's bond, while the filing
  epoch's snapshot and certificate held — the frozen denominator keeps its epoch, and slashes reach
  the denominator forward, never retroactively.

---

## Compute-backed native token — Token (TOK)

> Status: **design draft v0.1 + Phase A implementation on a branch. Not active on any network.**

VLT weight is deliberately non-transferable and decaying — it is voting power, not money. The Token
Program adds a separate, protocol-native asset ledger whose flagship asset **Token (TOK)** is minted
in proportion to the *same* verified compute measure:

```text
reward_i(E) = R(E) · X_i(E) / X(E)
```

In one line: **mine verifiable LLM compute instead of hashes.** `R(E)` is a fixed emission schedule
with halving steps; total issuance is decided by the schedule, while measured compute `X(E)` decides
only the split — so difficulty is emergent, as the ratio `X(E)/R(E)`. The harder the network works,
the more compute one TOK costs, exactly as PoW difficulty retargets.

Hard boundaries the design commits to:

- **A TOK balance grants zero voting weight.** Finality power comes from `C_i(E)` and the `λ·B_i`
  cap alone; the asset cannot buy it.
- **Emission is fork-invariant**: it settles from the pinned VLT snapshot after the challenge window
  fully closes, so compute that exists only on a losing branch is never monetized, and no clawback
  is needed.
- **One protocol implementation** of balances, transfers and burns (SPL-style), not a contract
  standard with per-deployment differences.
- Usefulness is *not* judged by consensus — only the objective quantities `ρ`, `a`, `b` enter.

Emission constants (`R0`, halving interval, settlement offset) are deliberately unfrozen until
testnet shadow measurement.

---

## Post-quantum security scope

The following are valid descriptions of the present implementation:

- Native transaction authorization uses **ML-DSA-87**.
- Native secp256k1/Schnorr/ECDSA authorization is disabled in PQ consensus mode.
- Validator attestations, precommits, and the remote-signer path use ML-DSA-87.
- Consensus identity is 64-byte BLAKE2b-512-based `Hash64`.
- The default `kaspad` build does not link the optional EVM/secp256k1 execution stack.

The following claims would be inaccurate:

- “All network traffic is post-quantum encrypted.”
- “The EVM lane is post-quantum.”
- “misakas uses Narwhal/Bullshark consensus.”
- “misakas is a pure PoS chain.”
- “LLM compute replaces proof-of-work.” — VLT weights *finality votes*; PoW still produces and
  orders blocks.
- “Verified-LLM-Token weighting is live on testnet.” — it is implemented and dormant behind two
  fences.
- “ML-DSA-65 is the current native signature scheme.”

Transport-layer post-quantum confidentiality remains outside the current public PQ claim; an
ML-KEM-based transport design would require a separate implemented and audited protocol change.

## Native supply

The native network parameters define a maximum supply of **28 billion MSK**:

- **13 billion MSK** in genesis allocations; and
- **15 billion MSK** in network emission over 20 years, following the configured 5% annual
  exponential-decay schedule.

TOK, if and when activated, is a **separate asset** on its own ledger and does not draw from the MSK
supply schedule.

### Ecosystem token note

The Solana SPL mint used by the broader MISAKA ecosystem is a separate market representation; it is
**not** a native misakas UTXO address, **not** an EVM contract deployed on this testnet, and **not**
the compute-minted TOK described above.

| Resource | Value |
|---|---|
| SPL mint | `4e2DhohUAJ9EbrLey3rVVgFQzLCAeeBirSbdhqrh9snX` |
| Market page | [DEX Screener](https://dexscreener.com/solana/hvcuswpugjg8omexyaexjs7wxf8izdwqgljqyxfehhqc) |

Always verify token information through the official website before interacting with a contract or
mint.

## Optional EVM lane

The EVM implementation is an **opt-in node feature**, not part of the default secp-free build.
It provides an Ethereum-compatible execution environment while preserving a separate native PQ UTXO
lane.

| Property | Value |
|---|---|
| EVM implementation | [`revm`](https://github.com/bluealloy/revm) |
| EVM revision | Shanghai |
| Chain ID | `0x4D534B` (`5067595`) |
| Supported envelopes | Legacy, EIP-2930, EIP-1559 |
| JSON-RPC | HTTP on `--evm-rpc-listen` (normally port `8545`) |
| Execution model | Selected-parent chain with mergeset-delayed acceptance |
| Fees | EIP-1559-style base fee and gas accounting |
| Native-unit bridge | `1 sompi = 10^10 wei` |

The lane supports normal EVM transfers and contracts, an in-consensus **UTXO → EVM deposit** flow,
and an **EVM → UTXO withdrawal** path through the MISAKA withdrawal precompile. It is Ethereum
execution-compatible rather than Ethereum-consensus-compatible: there is no Beacon Chain, Engine
API, Ethereum devp2p, or Ethereum PoS consensus. See
[`docs/misaka-evm-design-v0.4.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/misaka-evm-design-v0.4.md),
[`docs/connecting-ethereum-tooling.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/connecting-ethereum-tooling.md),
and
[`docs/evm-differences-from-ethereum.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/evm-differences-from-ethereum.md).

---

## Build from source

### Requirements

- Rust **1.88 or newer**
- Git and a C/C++ build toolchain
- Protocol Buffers compiler (`protoc`)
- Clang / libclang for RocksDB bindings
- OpenSSL development headers and `pkg-config`

Ubuntu/Debian example:

```bash
sudo apt update
sudo apt install -y \
  git curl build-essential pkg-config libssl-dev \
  protobuf-compiler libprotobuf-dev clang libclang-dev

rustup update
```

### Clone and build the native node and tools

```bash
git clone https://github.com/MISAKA-BTC/misakas.git
cd misakas

cargo build --release \
  -p kaspad \
  -p kaspa-pq-miner \
  -p kaspa-pq-validator \
  -p kaspa-pq-signer \
  -p misaka-cli --bin misaka
```

The unified operator CLI is the `misaka` binary from the `misaka-cli` package — name both
explicitly so Cargo never depends on workspace defaults. Prebuilt Linux binaries and `SHA256SUMS`
are published on the
[Releases](https://github.com/MISAKA-BTC/misakas/releases) page.

### Build an EVM-enabled node

```bash
cargo build --release -p kaspad --features evm
```

Use a miner built from the **same source tag or commit** as the EVM-enabled node.

## Run a `testnet-10` node

```bash
./target/release/kaspad --testnet --utxoindex --rpclisten-borsh=default --rpclisten-json=default
```

The node discovers public testnet peers through the MISAKA DNS seeders
(`seeder1.misakascan.com` / `seeder2.misakascan.com`). `--utxoindex` is required for wallet and
validator funding lookups. Use `--enable-unsynced-mining` only for a deliberately isolated network:
mining before you have synced to the public testnet forks you off from genesis.

### Default `testnet-10` ports

| Interface | Port | Default state | Used by |
|---|---:|---|---|
| P2P | `26211` | Enabled | Node-to-node BlockDAG traffic |
| gRPC | `26210` | Loopback, enabled | Miner and protobuf clients |
| wRPC Borsh | `27210` | Disabled until configured | CLI wallet and validator sidecar |
| wRPC JSON | `28210` | Disabled until configured | JSON WebSocket clients / explorer backends |
| Ethereum JSON-RPC | `8545` | EVM build only; disabled until configured | ethers, viem, Hardhat, Foundry |

Mainnet uses `26110/27110/28110` (P2P `26111`), devnet `26610/27610/28610` (P2P `26611`).
Do not point a WebSocket wallet at the gRPC port.

### Start the bundled miner

Mine only after the node is synchronized with the public testnet, and only to a native 64-byte
ML-DSA-87 `misakatest:` address:

```bash
./target/release/kaspa-pq-miner --node-grpc 127.0.0.1:26210 --network-id testnet-10 --blocks 0 --min-block-interval-ms 250 --pay-address <misakatest:...>
```

## Run a validator (testnet)

The `kaspa-pq-validator` sidecar connects to a local node over wRPC and attests while its ML-DSA-87
stake bond is active. Testnet enforces the production minimum of **20,000,000 MSK**
(`2e15` sompi).

```bash
kaspa-pq-validator keygen --out val.seed --network testnet
```

```bash
kaspa-pq-validator bond --node-rpc 127.0.0.1:27210 --validator-key val.seed --amount 2000000000000000 --network testnet-10
```

```bash
kaspa-pq-validator run --node-rpc 127.0.0.1:27210 --validator-key val.seed --stake-bond <txid:index> --signed-epoch-db val.state --network testnet-10 --attest-poll-secs 3
```

Use a **fresh** `--signed-epoch-db` per network — reusing one across networks trips the
anti-equivocation guard on overlapping epoch numbers. Once enough weight has attested,
`getDnsConfirmation` reports `dnsConfirmed: true` plus a `lastDnsConfirmedAnchor`; treat that anchor
as DNS-final. Full procedure:
[`docs/validator-runbook.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/validator-runbook.md).

## Repository map

| Path | Purpose |
|---|---|
| `kaspad/` | Full node daemon |
| `consensus/` | GHOSTDAG, validation, DNS + VLT overlay, EVM commitments |
| `consensus/core/src/vlt.rs` | VLT types, decay, weight, quorum, sortition, epoch snapshot |
| `consensus/core/src/dns_finality.rs` | Prevote/precommit rounds, credit rule, quorum denominator |
| `crypto/` | Hashes, addresses, scripts, Merkle and UTXO commitments |
| `wallet/` | Native wallet libraries, CLI, WASM, key handling |
| `pq-miner/`, `misaminer/` | BLAKE2b-512 Layer-0 CPU miner and mining client |
| `kaspa-pq-validator/` | Stake-bonded validator sidecar |
| `kaspa-pq-signer/` | Isolated validator signing daemon |
| `misaka-cli/` | Unified operator CLI (`misaka` binary) |
| `kaspa-evm/`, `rpc/eth/` | Feature-gated EVM executor and Ethereum JSON-RPC adapter |
| `bridge/` | Stratum mining bridge and operator dashboard |
| `misaka-dnsseeder/` | DNS peer seeder |
| `docs/` | Specifications, ADRs, runbooks, compatibility notes |

## Key protocol documents

| Document | Subject |
|---|---|
| [ADR-0019](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0019-mldsa87-migration.md) | ML-DSA-87 migration (governing PQ record) |
| [ADR-0009](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0009-dns-probabilistic-finality.md) | DNS probabilistic finality overlay |
| [ADR-0016](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0016-stake-locked-bond-utxos.md) / [ADR-0017](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0017-all-active-staker-attestation.md) | Stake-locked bonds, all-active attestation |
| [ADR-0024](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0024-verified-llm-token-weighted-bft.md) | **Verified LLM Token-weighted BFT** |
| [ADR-0025](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0025-chain-participation-and-ibd-candidate-selection.md) | Chain participation and IBD candidate selection |
| [ADR-0020](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0020-selected-parent-evm-lane.md) / [ADR-0023](https://github.com/MISAKA-BTC/misakas/blob/main/docs/adr/0023-base-three-lane-execution.md) | Selected-parent EVM lane, three-lane execution |
| [Compute Token Program v0.1](https://github.com/MISAKA-BTC/misakas/blob/main/docs/misaka-compute-token-program-design-v0.1.md) | Compute-backed TOK emission (draft) |
| [`docs/kaspa-pq-spec.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/kaspa-pq-spec.md) | Consensus-level PQ specification |

## Testing and development

```bash
cargo test --release
```

```bash
cargo nextest run --release
```

```bash
./check
```

`./check` runs formatting, clippy, dependency and PQ-only CI guards, including
`scripts/pq-ci-guard.sh`, which hard-gates that neither `kaspa-consensus` nor `kaspad` links
secp256k1. Consensus, cryptographic, address, activation, overlay, and bridge changes should include
deterministic regression tests. Changes that alter accepted transactions, commitments, state roots,
receipts, or activation behavior must be treated as consensus changes and documented accordingly.

## Contributing

Contributions are welcome from protocol engineers, wallet and RPC developers, node operators,
miners, GPU/inference operators, auditors, documentation writers, and testers. Read
[`CONTRIBUTING.md`](https://github.com/MISAKA-BTC/misakas/blob/main/CONTRIBUTING.md), open an issue
for design-sensitive changes, and include tests and migration/activation notes in pull requests.

## Security

Do not publish suspected vulnerabilities as public issues. Follow
[`SECURITY.md`](https://github.com/MISAKA-BTC/misakas/blob/main/SECURITY.md) and report them
privately to the maintainers with the affected component, impact, and reproduction steps.

## Upstream attribution

misakas is derived from [`rusty-kaspa`](https://github.com/kaspanet/rusty-kaspa). The project keeps
many upstream `kaspa-*` crate and binary names for compatibility and to preserve source history.
Credit for the original Rust Kaspa implementation belongs to the Kaspa developers; the MISAKA
contributors maintain the independent network changes, post-quantum native transaction path,
validator and verified-compute overlay, and optional EVM integration.

## License

Distributed under the **ISC License**. See
[`LICENSE`](https://github.com/MISAKA-BTC/misakas/blob/main/LICENSE).

```text
Copyright (c) 2026 MISAKA contributors
Copyright (c) 2022-2024 Kaspa developers
```

Third-party dependencies and copied components remain subject to their respective licenses and
notices.

---

<sub>misakas — post-quantum-native UTXO authorization on a Kaspa-derived PoW BlockDAG, with finality
weighted by verified LLM compute.</sub>
