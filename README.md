# LLM inference work on a post-quantum BlockDAG

**PALW · Native UTXO accounting · Model-free node verification · LMC**

LMC is an independent Layer 1 derived from [rusty-kaspa](https://github.com/kaspanet/rusty-kaspa). Its protocol connects registered LLM execution to block-production rights, using a BlockDAG for ordering and a native UTXO ledger for value transfer.

The native asset is **LLMarket coin**, ticker **LMC**. Native transaction authorization uses **ML-DSA-87**. Producers execute registered models; selected Panels verify claims; nodes authenticate the evidence and adjudicate bounded computational disputes.

The governing verification principle is:

> **Probabilistic detection. Public localization. Deterministic adjudication. Economic deterrence.**

This README describes the protocol's architecture, responsibilities and rules. Detailed execution, governance and economic contracts are defined in the linked specifications.

## Resources

| Resource | Link |
| --- | --- |
| Source repository | [MISAKA-BTC/LLM-chain](https://github.com/MISAKA-BTC/LLM-chain) |
| Website | [misakachain.com](https://misakachain.com/) |
| Whitepaper | [misakachain.com/whitepaper](https://misakachain.com/whitepaper) |
| Protocol principles | [Design principles][principles] |
| Specifications | [Documentation index][docs] · [RFC index][rfcs] · [ADR index][adrs] |

## 1. Responsibilities

| Participant | Responsibility |
| --- | --- |
| **Miner / Producer** | Execute the registered computation, sign its canonical claim, supply the required execution materials and consume an eligible right when producing a reward block. |
| **Panel** | Verify the assigned claim under its profile, obtain the required materials and submit an approval or an authenticated challenge. |
| **Node** | Validate signatures, commitments, claim context, local Court relations, certification, rights and block state transitions. |
| **Model-admission authority** | Decide model adoption and model-level stops within the BFT governance scope. This does not replace individual claim verification. |

A consensus node does not need to hold all model weights or rerun the entire model. It can receive the small weight, input and intermediate-state fragments required for a particular Court relation, together with openings to the registered commitments.

## 2. Register an execution contract, not just a model name

A model registration identifies an immutable execution instance. A name such as “Qwen” or a mutable download URL is insufficient.

The registration binds:

- Model artifacts, weights, configuration and content roots.
- Tokenizer, input adapter and challenge encoding.
- Program graph, supported kernels and tensor layouts.
- Numerical types, quantization, rounding, overflow and reduction rules.
- Generation, sampling and stopping rules.
- Readout positions and encoding.
- Trace schema, initial-state dependencies and local Court checkers.
- Resource bounds, model origin, economic classification and admission basis.

The input, program, weights and execution rules must identify the same computation throughout claim verification. Changing an owner, URL, wrapper or metadata does not by itself create a new computational work identity.

### Qwen execution profile

The **QF1** contract specifies a Qwen3-8B execution profile built around its 36 Transformer blocks. It fixes an integer A16 execution, a challenge-bearing HEX64 input frame, a StateMap, readout material and terminal relations.

The profile defines the precise tensor positions and numerical values used by the mining readout. Its registered integer execution is the reference computation; arbitrary GPU floating-point results are not substituted for those values.

The natural-language answer and the mining value have different roles. Two useful answers may have identical text. Mining identity and work accounting are determined by the registered execution contract.

See [QF1: Qwen execution and readout][qf1] and [FOW1: function output and work admissibility][fow1].

## 3. PALW work, readout and Ticket

PALW connects mining to a declared model execution:

1. Fix the job, owner, execution profile, attempt context, target and applicable deadlines.
2. Derive the execution input using the profile's authenticated challenge.
3. Execute the registered computation and form its canonical output and trace commitments.
4. Derive the mining value from the prescribed execution states.
5. Evaluate the Ticket using the profile's canonical transcript and target.
6. Verify and certify the claim before enabling the associated economic effects.

An attempt identifier distinguishes contexts. It is not a substitute for the LLM-derived mining value. A nonce, address or salt added only after execution must not provide an independent way to redraw the same work.

The protocol distinguishes:

| Item | Meaning |
| --- | --- |
| **Answer** | The result supplied to the user. |
| **Readout / mining value** | A value derived from the execution states selected by the profile. |
| **Claim** | A signed commitment to the execution, its context and obligations. |
| **Ticket** | The profile's eligibility or lottery evaluation. |
| **Work right** | A state-tracked entitlement derived from eligible, verified work. |
| **Award block** | A block that consumes the required entitlement through the normal validation path. |

Ticket and work-quantum accounting are profile-specific. One inference is not defined universally as one guaranteed block, and the same receipt or right cannot be spent repeatedly.

Correctness, distinct attempt identity and computational cost are separate properties. A matching output or trace does not establish which physical machine computed it, how much energy it used or whether an equivalent algorithm was faster. Admissibility rules address those considerations through the selected review or governance route. A correct optimization is not treated as arithmetic fraud.

## 4. TB1: separate randomness for separate decisions

The **TB1** design uses BGW-style verifiable secret sharing, post-quantum authenticated communication and signed shares to reconstruct a public seed.

Execution and Panel selection use separate purposes:

| Purpose | What it controls |
| --- | --- |
| **EXEC** | The challenge used by the declared execution. |
| **PANEL** | Selection of verifiers after the relevant claims and selection context have been fixed. |
| **SCHEDULE** | Assignment of eligible execution rights to future slots. |
| **ROLE** | Selection of the next designated participant set. |
| **ADMISSION / MODEL_AUDIT** | Additional control or evaluation selection where the corresponding contract requires it. |

Each purpose and round has its own secret material, fixed context and release conditions. Rehashing a seed that is already public does not create new future unpredictability.

TB1 requires the consumer set and other relevant choices to be fixed before the corresponding seed is exposed. Reorg handling and durable custody prevent an exposed round from being reused under a different freeze.

A TB1 reconstruction certificate authenticates the randomness process. It is not a vote establishing ledger finality and is not proof that an LLM computation was correct.

See [TB1: internal threshold beacon][tb1].

## 5. Panel verification and model-free Court

### Panel verification

Each profile defines the Panel's inspection duties. In the **one-honest full-reexecution profile**, selected verifiers obtain the authenticated model and claim materials and independently rerun the complete declared computation.

Its safety conditions include an active honest verifier, usable materials, adequate inspection time, access to the challenge path and canonical inclusion and processing of the required evidence. An honest participant's mere presence does not replace those conditions.

A signature records an approval and its responsibility. It does not reveal how the signer obtained the result or prove that the signer personally performed every operation.

### Court adjudication

A computational refutation must identify a relation in the Miner's original committed execution. The submitting party supplies the relation's operands and the necessary authenticated fragments.

The node checks:

1. The evidence belongs to the registered model, program and original claim.
2. The openings match the relevant model, input or trace roots.
3. The selected position, operation and dependencies are the ones required by the execution contract.
4. The deterministic local computation contradicts the committed result.

A weight from another model cannot be substituted without satisfying the registered root and position bindings. Court cost is bounded by the admitted relation, evidence format and resource limits.

A valid computational refutation takes precedence over unsupported positive approval. A mismatch between two independently produced roots is not, by itself, an arithmetic conviction.

Missing materials, unresolved requests, invalid challenges and established computational errors have separate outcomes. A local download failure is not automatically a slashable computation fault.

See [Public materials and one-honest verification][one-honest] and [Execution integrity and unilateral refutation][integrity].

### Cryptography authenticates evidence

Hashes, signatures and Merkle openings bind data, identities and context. The verification architecture uses re-execution, local arithmetic adjudication and economic responsibility; it does not use cryptographic validity proofs of LLM computation.

## 6. Final, Gate C and one-use accounting

The following conditions serve separate purposes:

| Condition | Purpose |
| --- | --- |
| **Panel / Court outcome** | Establish the required verification result and resolve applicable disputes. |
| **Final** | Complete the claim lifecycle conditions defined by its profile. |
| **Gate C / SecurityCertified** | Provide the independent claim-level authorization required for credit or payment. |
| **Ticket result** | Establish the applicable mining eligibility. |
| **Maturity, dependencies and deadlines** | Restrict when the right may be used. |
| **Atomic consumption** | Prevent repeated use of the same right, work key or funding reservation. |
| **Block validation and settlement** | Apply only the authorized state changes and economic effects. |

Final alone does not replace Gate C. A successful local replay does not replace the required selected-verifier authorization. Model-admission approval does not replace either.

The normal block path verifies the claim's eligibility and consumes the required identifiers and reservations together with the corresponding settlement. A rejected candidate must not leave a partial debit or payment.

Reorg and restart handling restore these effects from the selected history. A locally remembered success is not sufficient to classify a claim as paid on a different branch.

Protocol deadlines use their specified chain context and clock. Waiting, retransmission, a new observer or a changed local log does not restart the original deadline. Block count, DAA progress and wall-clock time are not interchangeable with finality.

## 7. MAD2: two model-admission routes

The **MAD2** model separates registration, mining adoption and individual claim eligibility.

### KERNEL_AUTO

A candidate is lowered into supported kernels and checked against the prescribed automatic admission policy. The model must have a well-defined deterministic execution, valid bindings, supported local checkers and the required admissibility results.

The automatic policy distinguishes execution consistency (G-EXEC), the required collision property (G-COLL), repeated cost-path assessment (G-COST) and practical usefulness (G-USE). The cost assessment is scoped to its specified workloads, reuse paths and execution conditions.

Automatic lowering composes operations that nodes already understand. Model registration does not authorize loading arbitrary native code into consensus. An unknown primitive requires an explicit kernel, codec and checker upgrade.

### BFT_DECISION

An authorized, fixed participant set signs **YES** or **NO** on a specific immutable proposal.

The voting conditions do not require a mathematical cost theorem, a collision theorem or a submitted quality-proof package. Nodes check the proposal identity, authority, signatures, quorum, generation, deadlines and applicable state.

This route records **BFT policy approval**. It does not turn an unknown automatic assessment into a mathematical PASS.

Both routes require technical compatibility. A YES decision cannot authorize unsupported kernels, an ambiguous numerical specification, invalid roots or a fraudulent claim.

### Model origin and management authority

| Model category | Adoption | BFT model stop / resume | LLMarket treatment |
| --- | --- | --- | --- |
| **Original genesis model** | Fixed by the original genesis manifest | Outside BFT model-management authority | No Option market; eligible worker reward goes to the Miner. |
| **Additional model** | KERNEL_AUTO or BFT_DECISION | Applies to both adoption routes | Uses MOE2. |
| **BFT-designated fork model** | BFT_DECISION with the fork classification bound into the proposal | Applies, including after adoption | No Option market; eligible worker reward goes to the Miner. |

Only the original genesis entries receive the genesis exemption. A later model cannot acquire it by using the same name, an alias, an economic coefficient or a new “genesis” label.

BFT may add a fork-designated model with genesis-like economic treatment, while retaining authority to stop it. These model-governance powers do not authorize changing ledger finality or accepting an invalid individual claim.

All model categories remain subject to their execution, Panel, Court, certification, Ticket, one-use and block-validation rules.

See [MAD2: kernel-auto and BFT model adoption][mad2].

## 8. LLMarket / ModelPosition economics

MOE2 applies to **additional models**. Registration of an additional model and creation of its unique market form one atomic transition. Market existence does not grant mining eligibility.

Let **R** be the model-attributed worker reward after the applicable pricing and common caps.

| Allocation | Rule |
| --- | --- |
| Position purchase | `B = floor(300 × R / 1000)` |
| Permanent-liquidity LMC | `A = floor(200 × R / 1000)` |
| Remaining reward | `D = R − B − A` |

The nominal split is **30% purchase, 20% permanent liquidity and 50% remaining reward**. Integer rounding is resolved by the formulas above.

The settlement:

1. Uses B to purchase Position from the model's own market.
2. Combines A with the required portion of the purchased Position to add liquidity to that market.
3. Records the permanent protocol liquidity contribution.
4. Burns only the purchased Position remaining outside that liquidity contribution.
5. Distributes D according to the applicable worker and role contract.

The native quote kernel defines the reserve arithmetic, integer rounding and validity checks. Purchase, liquidity contribution, excess burn and reward effects are committed atomically. Position is not minted merely to fill a liquidity shortfall.

### Permanent liquidity

Permanent liquidity is represented by a native locked receipt. It has no withdrawal right, transferable LP token, administrator withdrawal key or separate LP-fee claim.

The underlying pool assets remain usable for ordinary swaps. The locked receipt represents the contribution and its restrictions, not a promise that current reserve balances remain unchanged.

### Trading and transfers

| Operation | LMC burn | Owner allocation | Remaining amount |
| --- | ---: | ---: | ---: |
| Normal AMM buy or sell | 5% | 3% total | 92%, subject to integer rounding and the trade quote |
| Protocol purchase and liquidity legs | No additional trade burn | No additional direct owner payment | Governed by the reward allocation above |
| Direct Position transfer | No AMM trade fee | No AMM owner fee | Normal network transaction rules apply |

Direct Position transfers do not transfer a claim, bond, work right, admission authority or past liability.

Original genesis and designated fork models have no Option market under this classification. Their eligible worker reward R belongs entirely to the Miner; verification-service funding is arranged separately.

See [MOE2: reward purchase and permanent liquidity][moe2] and [the economic policy contract][moe2-policy].

## 9. Diagnostics are separate from consensus decisions

Failures must retain enough information to explain what happened and determine the next permitted action.

TB1 custody distinguishes permanent halt, corrupt storage, enrollment conflict, unavailable native state, changed views, an unopened window and an elapsed deadline. MOE2 distinguishes missing references from artifact, quote, funding, reservation and state-root mismatches.

The diagnostic contract requires:

- Typed reasons and retained lower-level causes, rather than control flow based on message text.
- Retry decisions tied to a specific state change, data dependency or original deadline.
- A distinction between normal waiting, rejection, an unrecorded observation and a local fault.
- Explicit exclusion reasons for invalid candidates without preventing valid candidates from being processed.
- Bounded logs and failure records that do not expose secret keys, unpublished shares or private input material.

A claim diagnostic is derived from one selected-parent view. It identifies the network, genesis, profile, selected parent, state root and evaluation point, then describes admission, Panel, Court, Final, Gate C, Ticket, right consumption, award block and MOE2 settlement.

The `GetPalwKernelClaimResponse.diagnostic_json` field carries the structured diagnostic. A local refused attempt is kept separate from committed claim outcomes; an observation from another selected parent is not silently reused.

Diagnostics do not grant eligibility, change deadlines, clear a halted round or become part of consensus state.

See [claim diagnostic schema][diagnostics], [TB1 custody types][custody] and [the claim RPC response][claim-rpc].

## 10. Native authorization and the optional EVM lane

Native transactions use the ML-DSA authorization domain and LMC's native UTXO rules. Native addresses retain the `misaka`, `misakatest` and `misakadev` network prefixes.

The optional EVM execution lane uses a separate Ethereum-compatible signature and execution domain. Its compatibility does not replace LMC's BlockDAG consensus or native authorization.

Post-quantum native authorization is not a claim that every transport channel or optional execution environment uses post-quantum cryptography.

## 11. Versioned rules and economic continuity

Execution profiles, kernels, randomness purposes, governance authority and economic rules are versioned contracts.

A rule change has an explicit scope and activation boundary. Existing claims retain their signed meaning, original deadlines and responsibilities. New policy does not retroactively redraw a losing Ticket, revive an expired right, erase an existing stop or reissue already consumed work.

Model admission, individual computation verification, independent certification and final economic effects remain separate decisions even when they share implementation components.

## Source organization

| Area | Responsibility |
| --- | --- |
| `consensus/` | BlockDAG validation, PALW state transitions, Court, rights and settlement. |
| `misaka/palw/` | Execution runtimes, kernel contracts, model tooling, remote workflows and verification services. |
| `misaka/palw/tir/` | Tensor IR, execution, lowering, artifacts and conformance machinery. |
| `misaka/pq/` | Native signing and validator components. |
| `misaka/apps/` | Operator applications and CLI. |
| `rpc/` | State access, diagnostics and service interfaces. |
| `misaka/evm/` | Optional EVM execution and indexing. |
| `testing/reference/` | Independent reference implementations. |
| `docs/` | Principles, RFCs, ADRs and detailed contracts. |

## Attribution and license

LMC is derived from [kaspanet/rusty-kaspa](https://github.com/kaspanet/rusty-kaspa). Upstream `kaspa-*` crate and binary names are retained where appropriate to preserve source history.

The source is distributed under the [ISC License][license]. See the repository's [contribution guidelines][contributing] and [security policy][security].

[principles]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/PRINCIPLES.md
[docs]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/README.md
[rfcs]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/README.md
[adrs]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/adr/README.md
[qf1]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/0066-qwen3-fow1-execution-and-readout.md
[fow1]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/0062-palw-function-output-and-work-admissibility.md
[tb1]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/0061-palw-internal-threshold-beacon.md
[one-honest]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/0050-public-materials-and-one-honest-panel.md
[integrity]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/0049-palw-execution-integrity-and-unilateral-refutation.md
[mad2]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/0067-model-admission-kernel-auto-and-bft-decision.md
[moe2]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/rfc/0064-reward-buy-and-permanent-liquidity.md
[moe2-policy]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/docs/design/palw/economics/moe2-economic-policy.json
[diagnostics]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/consensus/core/src/palw_claim_diagnostic_v1.rs
[custody]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/misaka/palw/remote/src/tb1_exposure.rs
[claim-rpc]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/rpc/core/src/model/message/palw_kernel_route_reads.rs
[license]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/LICENSE
[contributing]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/CONTRIBUTING.md
[security]: https://github.com/MISAKA-BTC/LLM-chain/blob/f839ef0dc0307b58ed6cc280e631b1e1d34f733c/SECURITY.md

