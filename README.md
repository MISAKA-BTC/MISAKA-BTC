# Hello from MISAKA! 👋

**MISAKA** is a post-quantum native Layer 1 blockchain built in Rust. It combines a Narwhal/Bullshark DAG consensus engine with NIST-standardized post-quantum cryptography — **ML-DSA-65 (FIPS 204)** for signatures and **ML-KEM-768 (FIPS 203)** for P2P key exchange — so that the network remains secure even against an adversary with a large-scale quantum computer.

MISAKA targets ~2 second block times, a transparent UTXO model, and a public PoS validator set. The project ships a full node, REST/Explorer API, CLI wallet, staking frontend, and a community mining client.

For a deeper technical overview, see the network repository: <https://github.com/MISAKA-BTC/Quantum-MISAKA/blob/main/README_ja.md>

---

## 🔗 Official Links

| Resource | URL |
|---|---|
| 🌐 **Website** | <http://misakabtc.com> |
| 📄 **Network (GitHub)** | <https://github.com/MISAKA-BTC/Quantum-MISAKA/blob/main/README_ja.md> |
| 💬 **Telegram** | <http://t.me/MISAKABTC> |
| 🎮 **Discord** | <http://discord.gg/C4nDFkJE4x> |
| 🏦 **Staking** | <https://misakastake.com> |
| ⛏️ **Mining (Web)** | <https://misakaminer.com> |
| ⛏️ **Mining (Source)** | <https://github.com/MISAKA-BTC/MISAKAminer> |

## 💱 Token

| | |
|---|---|
| 📊 **DEX Screener** | <https://dexscreener.com/solana/hvcuswpugjg8omexyaexjs7wxf8izdwqgljqyxfehhqc> |
| 🔑 **Contract Address (CA)** | `4e2DhohUAJ9EbrLey3rVVgFQzLCAeeBirSbdhqrh9snX` |

---

## 🛡️ Why Post-Quantum?

Most existing blockchains rely on elliptic-curve cryptography (Ed25519, ECDSA, secp256k1) and pairing-based zk systems (BLS12-381, Groth16, PLONK). All of these are broken by Shor's algorithm once a sufficiently large quantum computer exists. MISAKA was designed from day one to avoid that cliff:

- **Validator & transaction signatures** — ML-DSA-65 (NIST FIPS 204 / Dilithium3). ECC is **completely excluded** from the signature path.
- **P2P transport** — ML-KEM-768 key encapsulation + ChaCha20-Poly1305 with mutual ML-DSA-65 authentication.
- **State commitments** — SHA3-based MuHash for incremental UTXO accumulators.
- **Domain separation** — every signature is computed under an explicit DST tag.

## 🏗️ Architecture at a Glance

```
┌─────────────────────────────────────────────────┐
│         Narwhal/Bullshark DAG Consensus         │
│           Public PoS validator set              │
├─────────────────────────────────────────────────┤
│  ML-DSA-65 (FIPS 204)  │  ML-KEM-768 (P2P)     │
│  SHA3-256 / BLAKE3     │  Post-Quantum Safe     │
├─────────────────────────────────────────────────┤
│           Transparent UTXO Model                │
└─────────────────────────────────────────────────┘
```

## 🤝 Contributing

MISAKA is an open ecosystem and welcomes contributors of all kinds — protocol engineers, validator operators, miners, dApp builders, translators, and community members.

- **Join the conversation** — drop into our [Discord](http://discord.gg/C4nDFkJE4x) or [Telegram](http://t.me/MISAKABTC) and say hello.
- **Run a node / validator** — see the network repository for the public testnet quickstart.
- **Mine MISAKA** — grab the official miner at [MISAKAminer](https://github.com/MISAKA-BTC/MISAKAminer) or use the hosted client at <https://misakaminer.com>.
- **Stake** — delegate or run validator stake at <https://misakastake.com>.
- **Report a vulnerability** — please disclose security issues privately via Discord DM to a core team member rather than opening a public issue.

## 📜 License

Core repositories are released under **Apache-2.0** unless stated otherwise in the individual repo.

---

<sub>MISAKA — *Post-Quantum Native Layer 1.*</sub>
