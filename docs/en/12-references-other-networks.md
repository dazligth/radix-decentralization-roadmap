[← Index](../../README.md) · [Previous: Chapter 11: final evolution towards Hyperscale](11-hyperscale-evolution.md) · [Next: Sources, methodology and open questions](sources-and-methodology.md)

<a id="s12"></a>

# 12. References from other decentralised networks

Several proposals in this report have no precedent in Radix, but they do in other networks that have published their design and experience. This chapter collects those precedents with their sources, consulted on 3 Nov 2026. **A precedent does not guarantee that the same solution will work the same way in Radix**: each one must be validated here.

Since Radix's goal is to solve everything on **Layer 1**, each reference states whether it is a Layer-1 mechanism (protocol or node software), an external read service, or a Layer-2 or partner-chain solution, which is cited only as a contrast.

<a id="s12-1"></a>

## 12.1 Summary by proposal

| Proposal in this report | Precedents | What they show |
|---|---|---|
| **R9.** Bootstrapping from a verified snapshot | **Sui:** "formal" snapshots of state at the end of each epoch, automatically verified against a validator-signed commitment. **Polkadot:** warp sync, which downloads and verifies finality proofs and then the state. **CometBFT/Cosmos:** state sync, verified with a light client. **Cardano:** Mithril, snapshots certified with stake-weighted threshold signatures. **Kaspa:** pruning-point proof plus the UTXO set. **Ethereum:** checkpoint sync. | It is a standard Layer-1 technique in almost every modern network. Solana is the counter-example: snapshot trust relies on choosing known validators. |
| **R8.** Pruned nodes and archive nodes | **Polkadot:** pruned by default (last 256 blocks). **NEAR:** garbage collection (5 epochs by default) and archive nodes with two-tier storage. **Sui** and **Aptos:** configurable pruning, with history in an archive or in the indexer. **Kaspa:** pruning by design, with an optional archive mode. **Cosmos SDK:** `default`, `nothing`, `everything` and `custom` options. **Ethereum:** partial history expiry (EIP-4444) in every execution client since July 2025, saving 300-500 GB per node. | Pruning by default, and leaving full history to a few archive nodes, is common practice. |
| **9.5.** Nodes with little RAM | **Cardano:** moving the ledger to disk (UTxO-HD with LSM) reduced resident memory from ~15-16 GB to ~5.0-5.8 GB (~65 %) in the node's beta release (IOG report, Q2 2026); replaying the chain needed only 4 GB in mainnet tests (Q3 2025). **Ethereum:** the Nimbus consensus client runs in 4 GB. | Moving data from memory to disk greatly reduces RAM. See in 12.9 why Radix is better placed. |
| **E2.** A read API inside the node | **Kaspa:** wRPC API built into the node. **Sui:** a gRPC index in the full node. **Cardano:** Acropolis (IOG), a modular Rust node with a Blockfrost-compatible API inside the node, which synced in ~1 hour instead of 4 days (project wound down in Q2 2026; the code remains published). | Building reads into the node itself, as E2 proposes, has been done. |
| **E3.** An indexer next to the node | **Solana:** the Geyser interface, where plugins loaded into the validator send accounts, transactions and blocks to external databases. **Aptos:** a transaction stream service (gRPC) for indexers. **Cardano:** Kupo, a lightweight indexer. | Feeding light indexers from the node is a common pattern. |
| **10.2.** Access-point discovery | **Cardano: Koios**, a community API with "no central instance", where independent operators add themselves to a topology file on GitHub and a monitor only marks up-to-date instances as active. **Kaspa:** a network of public nodes with a *Resolver* service that directs clients to the least-loaded node. **Polkadot:** light clients find nodes over P2P, without relying on RPC servers. | Federated read networks exist, but **none of those analysed publishes its access points in a registry on Layer 1 itself**. The on-ledger discovery proposed in this report would be a new contribution. |
| **6.2.** Verifiable reads and light client | **Polkadot:** smoldot, a light client that runs in the browser and "does in no way trust" full nodes. **Sui:** a light client that follows end-of-epoch checkpoints signed by validators. **Ethereum:** `eth_getProof` (EIP-1186), sync committees and Helios. | Light clients in the browser or wallet exist in production. |
| **6.2, item 6.** Incentives for serving reads | **Pocket Network** and **The Graph** pay whoever serves reads or indexes, with proofs of the work done. | They are useful models, but they run as separate networks. In Radix they would have to be implemented on Layer 1 itself. |
| **R1.** RocksDB memory limits | **Official RocksDB documentation:** by default each column family has its own block cache (32 MB since version 8.2; 8 MB before); index and filter blocks are not counted inside that cache; `max_open_files = -1` preloads all indexes and filters; the Write Buffer Manager caps memtable memory across column families and can charge it to the block cache. | It confirms the memory-use causes described in 9.2 and R1's solution. |
| **R11.** Rewriting the node in Rust | **Kaspa:** `rusty-kaspa` replaced the Go node as a "drop-in replacement", coexisting on the P2P network, with gradual adoption and later deprecation of the Go node. **Solana:** Frankendancer, a hybrid client. **Cardano:** new nodes (Amaru in Rust, Dingo in Go) that join first as relays and later produce blocks. **Ethereum:** client diversity. | Rewriting a node in phases, coexisting on the network, is a well-trodden path. Kaspa is the closest precedent to R11. |
| **M1.** Migration with claims | **Midnight (Glacier Drop):** a balance snapshot across eight networks; the user signs a message with the key of the snapshot address and names a new destination address, with a later window for those who did not claim in time. **Cosmos Hub:** state migration into a new genesis. **Uniswap:** the `MerkleDistributor` contract. | Signed claims and genesis migration are proven techniques. |
| **Chapter [11](11-hyperscale-evolution.md#s11).** Evolving Layer 1 | **Cardano: Ouroboros Leios**, which increases Layer-1 capacity through a protocol hard fork **on the same chain**, with a long-running dedicated testnet. | It shows the alternative to migrating to a new network. See 12.9. |

<a id="s12-2"></a>

## 12.2 Cardano

| Mechanism | Type | What it does | Status |
|---|---|---|---|
| **Mithril** | Layer 1 (signed by stake-pool operators, outside consensus) | Certifies snapshots with stake-weighted threshold signatures; the client verifies the certificate chain and restores a mainnet node "in less than 20 minutes", according to its documentation | In production (IOG report, Q2 2026). The latest participation figure found is from 2024 (~22 % of stake). |
| **UTxO-HD / LSM** | Layer 1 | Moves the ledger from memory to disk | Node beta 10.7.1/11.0.1: RAM from ~15-16 GB to ~5.0-5.8 GB. Official requirements: 24 GB in memory; 8 GB with the ledger on disk ("pending confirmation"); 250-350 GB disk. |
| **Koios** | Federated read service | A community API with several operators and no central instance | Running; discovery via a file on GitHub, no on-chain incentives |
| **cardano-db-sync / Blockfrost** | Read service | A full indexer on PostgreSQL (~438 GB, 64 GB RAM recommended); Blockfrost publishes its backend so anyone can host it | Running. It is the equivalent of Radix's Gateway, and shows its cost. |
| **Amaru (Rust), Dingo (Go)** | Layer 1 | New node implementations that join first as relays | Amaru runs as a mainnet relay; its block-production date is not verified |
| **Ouroboros Leios** | Layer 1 (protocol hard fork) | Adds endorser blocks voted by a committee, to increase capacity without changing the main chain (CIP-164, proposed) | Testnet since 23 Jun 2026; code complete targeted for Q4 2026; mainnet date not set. Needs more disk, and more relay resources for those with large stake. |

<a id="s12-3"></a>

## 12.3 Sui

| Mechanism | Type | What it does |
|---|---|---|
| **Configurable pruning** | Layer 1 | Epochs to retain for objects and for checkpoints; without pruning a node can exceed 7 TB |
| **Formal snapshots** | Layer 1 | Live state at the end of each epoch, verified against a validator-signed commitment; Mysten keeps those of the last 90 epochs |
| **Light client** | Layer 1 | Follows end-of-epoch checkpoints, with aggregated BLS validator signatures, from genesis |
| **Indexer and GraphQL** | Read service | A PostgreSQL indexer and a GraphQL API that users or providers can host; JSON-RPC marked as deprecated |
| **Full-node hardware** | — | 128 GB RAM and 4 TB NVMe |

<a id="s12-4"></a>

## 12.4 Aptos

- **Fast sync** (Layer 1): skips history and relies on validator signatures; it usually completes in a few hours.
- **Pruning** (Layer 1): keeps the latest 150 million transactions by default; archive nodes are deprecated in favour of the indexer.
- **Indexer** (read service): a gRPC transaction stream and a GraphQL API, which third parties can host.
- **Validator hardware:** 128 GB RAM and 3 TB NVMe (official estimate for 2024).

<a id="s12-5"></a>

## 12.5 Solana

- **Geyser** (Layer 1): plugins inside the validator that send data to external indexers. It is the closest precedent to E3's light indexer.
- **Snapshots** (Layer 1): they should be downloaded only from known validators to avoid malicious state; trust is social, not cryptographic.
- **History** (external service): for years only in Google Bigtable; the Old Faithful project packages it into files that can be hosted on IPFS or Filecoin.
- **Hardware:** an RPC node with 256 GB RAM. It is a counter-example of a light node.

<a id="s12-6"></a>

## 12.6 NEAR and Polkadot

- **NEAR** (Layer 1): RPC nodes with 5 epochs by default, and archive nodes with hot (3 TB SSD) and cold (115 TB HDD) storage. Its NEAR Lake data service, run by a single operator, stopped indexing on 24 Mar 2026. It illustrates the risk of depending on a central read service.
- **Polkadot** (Layer 1):
  - nodes pruned by default;
  - warp sync (finality proofs + state);
  - the **smoldot** light client in the browser, which its documentation describes as a "trust-minimized" alternative to public RPC servers, which are "convenient but centralized".
  - Validator: 32 GB RAM and 2 TB NVMe.

<a id="s12-7"></a>

## 12.7 Kaspa

| Mechanism | Type | What it does |
|---|---|---|
| **rusty-kaspa** | Layer 1 | A Rust rewrite of the Go node, as a "drop-in replacement". It first coexisted with Go nodes on the P2P network, with a small share of hash power; it then became the recommended node and the Go node was deprecated. The Crescendo hard fork (5 May 2025) moved the network from 1 to 10 blocks per second, already on the Rust node. |
| **Pruning by design** | Layer 1 | A new node downloads the pruning-point proof, headers, the UTXO set at that point and then blocks. Headers include a commitment to the UTXO set. There are retention and archive options. |
| **API and public nodes** | Read | A wRPC API in the node; a network of community-provided public nodes and a *Resolver* that distributes clients |
| **Hardware** | — | Minimum 16 GB RAM and 256 GB SSD; preferred 32 GB and 512 GB |

Kaspa is the most direct precedent for two of this report's proposals: **R11** (phased rewrite of the node in Rust) and **R8 + R9** (pruning by default with verifiable bootstrap).

<a id="s12-8"></a>

## 12.8 Midnight: how it implements privacy (contrast)

**Type:** a separate network, a **partner chain** of Cardano, built with the Partner Chains toolkit (Substrate-based); its block producers must be Cardano stake-pool operators. **It is not a model for Radix**, whose goal is Layer 1. It is analysed for how it implements privacy.

| Element | How it works |
|---|---|
| **Zero-knowledge proofs** | zk-SNARKs with the Halo2 system (which replaced Plonk in 2024), adapted for efficient recursion. According to its documentation, proofs are 128 bytes regardless of the computation and are verified in milliseconds. |
| **Public and private state** | Public state is on chain; private state is kept locally by each user and never exposed. |
| **Tokens** | NIGHT (public, UTXO model, Schnorr signatures); shielded tokens (with ZK proofs); DUST, a shielded, non-transferable resource generated by holding NIGHT and consumed to pay for transactions. |
| **Compact language** | A TypeScript-like language that compiles to ZK circuits. Private data enters as *witnesses* from the client; any private value that is to be made public must be marked with `disclose()`, or the compiler rejects the program. |
| **Proof generation** | In a local "proof server" run by the user; the documentation advises running it on a machine one controls, because it receives private data. |
| **Selective disclosure** | Viewing keys that let an authorised indexer read a user's shielded history. |
| **NIGHT distribution (Glacier Drop)** | A balance snapshot across eight networks (11 Jun 2025); the user signs a message with the key of the snapshot address and names a new address; later, additional windows for those who did not claim. It supported only single-key accounts, not multisig or contracts. |
| **Launch** | NIGHT was issued as a Cardano asset in December 2025; mainnet was planned for the end of March 2026 (the exact date has not been verified). |

**What could be applied on Radix's Layer 1** (it needs deep engine work and a protocol update):

1. **Native ZK proof verification** in the Radix Engine, callable from Scrypto.
2. **Shielded resources** alongside public resources (commitments and nullifiers), with fees paid through a generated, non-transferable resource in the style of DUST.
3. **Explicit disclosure discipline** for private data, checked by the compiler, in a Scrypto extension.
4. **Proof generation in the wallet**, locally.
5. **Signed claims with a long window** for any token migration (relevant to M1). The restriction to single-key accounts shows that Radix must plan for securified accounts.

What would **not** carry over: the separate chain, the federated operators, the dependency on Cardano and the bridge between networks.

<a id="s12-9"></a>

## 12.9 Lessons for Radix

1. **Bootstrapping from a verified snapshot (R9) is standard** on Sui, Polkadot, Cosmos, Cardano, Kaspa and Ethereum. It reinforces its priority.
2. **Pruning by default and leaving history to archive nodes (R8)** is common practice.
3. **4 GB of RAM: Radix is better placed than other networks, but it must be measured.**
   - None of the networks analysed runs a full node doing consensus and execution in 4 GB today. Published requirements range from 8 GB (Cardano with the ledger on disk, pending confirmation) through 16 GB (Kaspa, Geth) to 128-256 GB (Sui, Aptos, Solana). The closest precedent is **Cardano**, with ~5.0-5.8 GB after moving the ledger to disk, in its beta release.
   - However, **Radix works differently** in three ways that reduce the memory needed:
     - **its state is already on disk** (RocksDB, `substates` column family). Cardano's big reduction came precisely from moving an in-memory ledger to disk; Radix does not need that structural change, and its current consumption comes mostly from configuration (12 GB reserved for Java, unbounded RocksDB and a ~1.2 GB mempool), which can be tuned (R1-R2);
     - **its validator set is small** (about 100), so consensus state takes little space (the vertex store is capped at 150 MB);
     - **its current load is low** (~4,400 user transactions a day, section [1.2](01-02-current-situation.md#s1-2)), far from networks such as Solana or Sui, whose requirements reflect much higher volumes.
   - **Assessment:** with R1-R2, a node of **at most 8 GB looks achievable**, and **4 GB is plausible**, especially after the migration to Rust (R11). Both figures are estimates that must be confirmed by measuring a real node, including during high-load periods such as the March 2025 peak (~35,000 user transactions a day).
4. **Reads built into the node (E2) and an indexer next to the node (E3)** have direct precedents (Kaspa, Sui, Acropolis; Geyser, Aptos).
5. **Federated read networks exist** (Koios, Kaspa's public nodes), but they coordinate off chain. **Publishing access points on Layer 1 itself**, as this report proposes, would be a new contribution from Radix.
6. **Depending on a single read service is a real risk:** NEAR Lake stopped indexing in March 2026.
7. **The phased rewrite in Rust (R11)** has a very similar precedent in **Kaspa**, which moved from Go to Rust while coexisting on the network, with gradual adoption.
8. **Evolving Layer 1 without changing network is possible** when the new design is compatible with the old one, as Cardano does with Leios. Hyperscale, by contrast, is not compatible with Babylon (section [11.2](11-hyperscale-evolution.md#s11-2)). That is why it requires a migration, and why the community should also weigh the option of improving Babylon on its own chain before deciding.
9. **Privacy** can be approached on Layer 1 with ZK proof verification in the engine and shielded resources. It is a major undertaking, which Midnight solves with a separate network.

**Could not be verified:**

- Mithril stake participation in 2025-2026;
- Amaru's block-production date;
- the exact shutdown date of Sui's JSON-RPC;
- how Solana verifies snapshots on load;
- Kaspa's exact pruning window;
- the exact Midnight mainnet launch date;
- the per-network signature scheme of the Glacier Drop;
- the status of Ethereum's Portal Network in 2026;
- the date full Firedancer reached mainnet.

None of these points is used as an argument in the report.

---

[← Index](../../README.md) · [Previous: Chapter 11: final evolution towards Hyperscale](11-hyperscale-evolution.md) · [Next: Sources, methodology and open questions](sources-and-methodology.md)
