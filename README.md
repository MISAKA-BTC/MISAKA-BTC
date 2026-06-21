# Hello from MISAKA ⚡️

**Post-quantum-native UTXO Layer 1 · Kaspa-derived PoW BlockDAG · Optional EVM execution lane**

[![License: ISC](https://img.shields.io/badge/License-ISC-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.88%2B-orange.svg)](Cargo.toml)
[![Network](https://img.shields.io/badge/network-testnet--10-yellow.svg)](https://misakascan.com)

**misakas** is an independent Layer 1 network written in Rust and derived from
[`rusty-kaspa`](https://github.com/kaspanet/rusty-kaspa). Its native UTXO transaction path is
post-quantum-only: transaction authorization uses **ML-DSA-87** (FIPS 204, NIST security category
5), native addresses and scripts accept only the ML-DSA-87 P2PKH form, and legacy
secp256k1/Schnorr/ECDSA and P2SH paths are excluded from native consensus, mempool, and wallet
operation.

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
| Protocol documents | [`docs/`](docs/) |
| Whitepaper / specification | [MISAKA-BTC/specification](https://github.com/MISAKA-BTC/specification) |
| Issue tracker | [GitHub Issues](https://github.com/MISAKA-BTC/misakas/issues) |
| Security policy | [`SECURITY.md`](SECURITY.md) |
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
| Finality extension | Stake-bonded DNS finality overlay on top of PoW |
| EVM | Feature-gated, Shanghai-compatible execution lane on testnet |
| Mainnet | Parameters defined; not launched or endorsed for production |

The latest testnet release includes the **gas-pool v2** EVM execution rules and
`misaka_getEvmTxStatus`. Operators must run matching node and miner builds from the same release
whenever a release changes consensus-committed EVM header fields.

## Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│              PoW BlockDAG + GHOSTDAG ordering                  │
│             10 BPS target · Kaspa-derived core                 │
├─────────────────────────────────────────────────────────────────┤
│      DNS finality overlay: bonded stake + ML-DSA attestations  │
│      PoW still produces blocks and drives GHOSTDAG selection   │
├───────────────────────────────┬─────────────────────────────────┤
│ Native UTXO lane              │ Optional EVM lane               │
│                               │                                 │
│ ML-DSA-87 only                │ revm / Shanghai                 │
│ Transparent UTXO accounting   │ EIP-155 / EIP-1559             │
│ 64-byte BLAKE2b-512 identity  │ secp256k1/ECDSA domain         │
│ misaka / misakatest addresses │ Selected-parent acceptance     │
├───────────────────────────────┴─────────────────────────────────┤
│ gRPC · wRPC Borsh/JSON · Ethereum HTTP JSON-RPC                │
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
lock native UTXOs, sign epoch attestations with ML-DSA-87, and contribute to stake-confirmed
canonical anchors. The overlay adds reorg protection, validator rewards, slashing rules, and
anti-equivocation state while GHOSTDAG and blue-work remain the underlying ordering mechanism.

Validator operation is documented in [`docs/validator-runbook.md`](docs/validator-runbook.md). A
separate [`kaspa-pq-signer`](kaspa-pq-signer/) daemon can keep the validator key outside the
validator process and enforce signing policy through a protected Unix-domain socket.

### Optional EVM lane

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
API, Ethereum devp2p, or Ethereum PoS consensus.

The current gas-pool rules reserve space before execution, debit accepted transactions by actual gas
used, allow a non-fitting transaction to be skipped without starving smaller later transactions,
and do not charge the block gas pool for nonce/funds/base-fee admission skips. See:

- [`docs/misaka-evm-design-v0.4.md`](docs/misaka-evm-design-v0.4.md)
- [`docs/connecting-ethereum-tooling.md`](docs/connecting-ethereum-tooling.md)
- [`docs/ethereum-rpc-compat-matrix.md`](docs/ethereum-rpc-compat-matrix.md)
- [`docs/evm-differences-from-ethereum.md`](docs/evm-differences-from-ethereum.md)

## Post-quantum security scope

The following are valid descriptions of the present implementation:

- Native transaction authorization uses **ML-DSA-87**.
- Native secp256k1/Schnorr/ECDSA authorization is disabled in PQ consensus mode.
- Validator attestations and the remote-signer path use ML-DSA-87.
- Consensus identity is 64-byte BLAKE2b-512-based `Hash64`.
- The default `kaspad` build does not link the optional EVM/secp256k1 execution stack.

The following claims would be inaccurate:

- “All network traffic is post-quantum encrypted.”
- “The EVM lane is post-quantum.”
- “misakas uses Narwhal/Bullshark consensus.”
- “misakas is a pure PoS chain.”
- “ML-DSA-65 is the current native signature scheme.”

Transport-layer post-quantum confidentiality remains outside the current public PQ claim; an
ML-KEM-based transport design would require a separate implemented and audited protocol change.

## Native supply

The native network parameters define a maximum supply of **28 billion MSK**:

- **13 billion MSK** in genesis allocations; and
- **15 billion MSK** in network emission over 20 years, following the configured 5% annual
  exponential-decay schedule.

### Ecosystem token note

The Solana SPL mint used by the broader MISAKA ecosystem is a separate market representation; it is
**not** a native misakas UTXO address and is **not** an EVM contract deployed on this testnet.

| Resource | Value |
|---|---|
| SPL mint | `4e2DhohUAJ9EbrLey3rVVgFQzLCAeeBirSbdhqrh9snX` |
| Market page | [DEX Screener](https://dexscreener.com/solana/hvcuswpugjg8omexyaexjs7wxf8izdwqgljqyxfehhqc) |

Always verify token information through the official website before interacting with a contract or
mint.

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
  -p kaspa-wallet \
  -p kaspa-pq-miner \
  -p kaspa-pq-validator \
  -p kaspa-pq-signer
```

Prebuilt Linux binaries and checksums are available on the
[Releases](https://github.com/MISAKA-BTC/misakas/releases) page.

### Build an EVM-enabled node

```bash
cargo build --release -p kaspad --features evm
```

Use a miner built from the **same source tag or commit** as the EVM-enabled node.

## Run a `testnet-10` node

```bash
./target/release/kaspad \
  --testnet \
  --utxoindex \
  --rpclisten-borsh=default \
  --rpclisten-json=default
```

The node discovers public testnet peers through the configured MISAKA DNS seeders. The explorer is
available at [misakascan.com](https://misakascan.com/).

### Default `testnet-10` ports

| Interface | Port | Default state | Used by |
|---|---:|---|---|
| P2P | `26211` | Enabled | Node-to-node BlockDAG traffic |
| gRPC | `26210` | Loopback, enabled | Miner and protobuf clients |
| wRPC Borsh | `27210` | Disabled until configured | CLI wallet and validator sidecar |
| wRPC JSON | `28210` | Disabled until configured | JSON WebSocket clients / explorer backends |
| Ethereum JSON-RPC | `8545` | EVM build only; disabled until configured | ethers, viem, Hardhat, Foundry |

`--rpclisten-borsh=default` and `--rpclisten-json=default` resolve to the correct loopback ports for
the selected network. Do not point a WebSocket wallet at the gRPC port.

### Start the native wallet

```bash
./target/release/kaspa-wallet
```

In the wallet REPL, select the Borsh wRPC endpoint and connect:

```text
server 127.0.0.1:27210
connect
```

Wallet storage accepts both `open name` and `open name.wallet`; the `.wallet` suffix is normalized
internally.

### Start the bundled miner

Mine only after the node is synchronized with the public testnet:

```bash
./target/release/kaspa-pq-miner \
  --rpc 127.0.0.1:26210 \
  --network-id testnet-10 \
  --blocks 0 \
  --min-block-interval-ms 250 \
  --pay-address <misakatest:...>
```

The payout address must be a native 64-byte ML-DSA-87 `misakatest:` address. Use
`--enable-unsynced-mining` only for a deliberately isolated network; using it while joining the
public testnet can create a local fork from genesis.

### Run the EVM JSON-RPC adapter

```bash
./target/release/kaspad \
  --testnet \
  --utxoindex \
  --rpclisten-borsh=default \
  --rpclisten-json=default \
  --evm-rpc-listen=127.0.0.1:8545
```

Example client settings:

```text
Network name : MISAKA Testnet (EVM)
RPC URL      : http://127.0.0.1:8545
Chain ID     : 5067595
Currency     : MSK
EVM version  : Shanghai
```

## Repository map

| Path | Purpose |
|---|---|
| [`kaspad/`](kaspad/) | Full node daemon |
| [`consensus/`](consensus/) | GHOSTDAG, validation, DNS overlay, EVM commitments |
| [`crypto/`](crypto/) | Hashes, addresses, scripts, Merkle and UTXO commitments |
| [`wallet/`](wallet/) | Native wallet libraries, CLI, WASM, key handling |
| [`pq-miner/`](pq-miner/) | Bundled BLAKE2b-512 Layer-0 CPU miner |
| [`misaminer/`](misaminer/) | MISAKA mining client |
| [`kaspa-pq-validator/`](kaspa-pq-validator/) | Stake-bonded validator sidecar |
| [`kaspa-pq-signer/`](kaspa-pq-signer/) | Isolated validator signing daemon |
| [`kaspa-evm/`](kaspa-evm/) | Feature-gated EVM executor and bridge logic |
| [`rpc/eth/`](rpc/eth/) | Ethereum HTTP JSON-RPC compatibility adapter |
| [`bridge/`](bridge/) | Stratum mining bridge and operator dashboard |
| [`misaka-dnsseeder/`](misaka-dnsseeder/) | DNS peer seeder |
| [`docs/`](docs/) | Specifications, ADRs, runbooks, compatibility notes |

## Testing and development

```bash
# Full test suite
cargo test --release

# Faster parallel runner, when cargo-nextest is installed
cargo nextest run --release

# Formatting, clippy, dependency and PQ-only CI guards
./check

# Benchmarks
cargo bench
```

Consensus, cryptographic, address, activation, and bridge changes should include deterministic
regression tests. Changes that alter accepted transactions, commitments, state roots, receipts, or
activation behavior must be treated as consensus changes and documented accordingly.

## Contributing

Contributions are welcome from protocol engineers, wallet and RPC developers, node operators,
miners, auditors, documentation writers, and testers. Read [`CONTRIBUTING.md`](CONTRIBUTING.md),
open an issue for design-sensitive changes, and include tests and migration/activation notes in pull
requests.

## Security

Do not publish suspected vulnerabilities as public issues. Follow [`SECURITY.md`](SECURITY.md) and
report them privately to the maintainers with the affected component, impact, and reproduction steps.

## Upstream attribution

misakas is derived from [`rusty-kaspa`](https://github.com/kaspanet/rusty-kaspa). The project keeps
many upstream `kaspa-*` crate and binary names for compatibility and to preserve source history.
Credit for the original Rust Kaspa implementation belongs to the Kaspa developers; the MISAKA
contributors maintain the independent network changes, post-quantum native transaction path,
validator overlay, and optional EVM integration.

## License

This repository is distributed under the **ISC License**, not Apache-2.0. See [`LICENSE`](LICENSE).

```text
Copyright (c) 2026 MISAKA contributors
Copyright (c) 2022-2024 Kaspa developers
```

Third-party dependencies and copied components remain subject to their respective licenses and
notices.

---

<sub>misakas — post-quantum-native UTXO authorization on a Kaspa-derived PoW BlockDAG.</sub>
