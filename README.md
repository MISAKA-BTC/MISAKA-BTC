# MISAKA

MISAKA is a post-quantum UTXO Layer 1 and Kaspa-derived BlockDAG whose current public network is **Testnet-11 Relaunch 5f**. The source of truth for protocol behavior is the [`MISAKA-BTC/misakas`](https://github.com/MISAKA-BTC/misakas) repository, its `main` branch, and the documents under [`docs/`](https://github.com/MISAKA-BTC/misakas/tree/main/docs).

## Current operator path

Use current `main`; do not reuse commands from retired testnet-10, testnet-21, or earlier Testnet-11 relaunches.

```bash
git clone https://github.com/MISAKA-BTC/misakas.git
cd misakas
git switch main
cargo build --release -p kaspad -p misaka-cli
```

Check an existing PALW Bond without supplying a class ID:

```bash
misaka --network testnet-11 bond status --bond <txid>:<index>
```

Set up and run the BASE-0/Floor producer:

```bash
misaka --network testnet-11 mining setup \
  --model floor --key-file ~/.misaka/miner.seed \
  --bond <registered-bond-txid>:<index> --peer <peer-ip>:26311
misaka --network testnet-11 mining start --print-command
misaka --network testnet-11 mining start
```

The setup wizard distinguishes a registered Bond from collateral sufficient for sustained production. Do not run `--palw-register-bond` again for an already registered key; a second Bond for the same producer key is rejected. For the complete node, Bond, A16, dashboard, SSH forwarding, and stop procedures, read [`docs/testnet11-join-mining.md`](https://github.com/MISAKA-BTC/misakas/blob/main/docs/testnet11-join-mining.md) and the [operator wiki](https://github.com/MISAKA-BTC/misakas/wiki/Testnet-11-Operator-UI-JA).

## What PALW is

PALW (Proof of Advanced LLM Work) is the block-production lane. Testnet-11's BASE-0/Floor class is deterministic, integer-only, built into the node, and needs no model download. Other classes require the matching registered artifact and enough class-specific collateral. Full nodes validate commitments and chain rules; they do not need to load an LLM merely to sync.

Native transaction authorization uses ML-DSA-87. The optional EVM lane is separate and retains Ethereum-compatible signing semantics; post-quantum claims do not apply to transport or that EVM signature domain.

## Resources

- [Source repository](https://github.com/MISAKA-BTC/misakas)
- [Current Testnet-11 runbook](https://github.com/MISAKA-BTC/misakas/blob/main/docs/testnet11-join-mining.md)
- [Node operator guide](https://github.com/MISAKA-BTC/misakas/blob/main/docs/testnet11-node-operator.md)
- [Protocol documents](https://github.com/MISAKA-BTC/misakas/tree/main/docs)
- [Releases](https://github.com/MISAKA-BTC/misakas/releases)
- [Explorer](https://misakascan.com/)
- [Wiki](https://github.com/MISAKA-BTC/misakas/wiki)
- [Issues](https://github.com/MISAKA-BTC/misakas/issues)
