[← Index](../../README.md) · [Previous: Sources, methodology and open questions](sources-and-methodology.md)

# Glossary

Terms used in the report, briefly explained.

| Term | Meaning |
|---|---|
| **Babylon** | The current version of the Radix network, live since September 2023. |
| **Olympia** | The previous version of the Radix network (2021-2023). Its state was migrated to Babylon. |
| **Hyperscale** | Radix's next-generation design: a sharded network that scales by adding shards. There is an independent open-source Rust implementation, `hyperscale-rs` (chapter [11](11-hyperscale-evolution.md#s11)). |
| **Xi'an** | The name the Radix roadmap gave to the Hyperscale-based stage of the network. |
| **Gateway (Babylon Network Gateway)** | A central service that indexes the network and offers an API to wallets, dApps and explorers. It is a PostgreSQL database (~2.5 TB), an indexer (DataAggregator) and an API. |
| **Full node** | A computer that keeps a copy of the ledger and validates every transaction. If it also takes part in consensus, it is a validator. |
| **Validator** | A full node that also takes part in consensus and signs blocks. Mainnet has about 100 active ones. |
| **Read node** | Proposed in this report: a full node with a light indexer that serves queries from wallets, dApps and explorers. |
| **Archive node** | A node that keeps the full history and produces verifiable snapshots for other nodes. |
| **Light indexer (`radix-lite-indexer`)** | Proposed in this report: a process that replaces the Gateway, stores only the indexes the node lacks, and takes ~10-50 GB (estimated). |
| **Core API, LTS, Engine State API** | APIs offered by the node itself. LTS is the part of the Core API with long-term compatibility. The Engine State API reads the internal state of any entity. |
| **Ledger** | The complete record of committed transactions on the network. |
| **State version** | The sequence number of each transaction in the ledger. Mainnet is past 567 million. |
| **Round update** | A system transaction generated every consensus round. About 97-98 % of all state versions are round updates. |
| **Epoch** | A period (about 5 minutes on average) after which the validator set may change. |
| **Receipt** | The result of executing a transaction: state changes, events and fees. |
| **Subintent / pre-authorisation** | Part of a transaction signed in advance by one user and completed later by another party. |
| **ROLA** | The protocol a dApp uses to verify that a user controls an account or persona, used for sign-in. |
| **RDT (Radix dApp Toolkit)** | The library dApps use to connect to the wallet. |
| **Sargon** | The Rust library shared by the iOS and Android wallets. |
| **Radix Connect** | The channel between the mobile wallet and the browser or dApp: relay, WebRTC signalling or iroh. |
| **Scrypto** | Radix's Rust-based language for writing smart contracts (*blueprints*). |
| **Radix Engine** | The virtual machine that executes transactions and Scrypto contracts on Babylon. |
| **Manifest** | A human-readable description of what a Radix transaction does. |
| **SBOR** | Radix's data encoding format. |
| **RocksDB** | The embedded database the node uses to store the ledger and state. |
| **Column family** | Each of RocksDB's internal "tables" (the node uses 26). |
| **JMT (Jellyfish Merkle Tree)** | The Merkle tree the node uses to compute the state root. It allows proving that a piece of data belongs to the state. |
| **Merkle proof** | A short cryptographic proof that a piece of data belongs to a set whose root is signed. |
| **Ledger proof** | A header signed by validators that includes the roots of state, transactions and receipts. |
| **Light client** | A program that verifies a node's responses with cryptographic proofs, without storing the ledger and without trusting the node. |
| **Verified snapshot** | A copy of the state at a given version that the node checks against the validator-signed root before using it. |
| **Pruning** | Deleting old data the node no longer needs. |
| **JVM** | The Java virtual machine. It runs the part of the node written in Java. |
| **JNI** | The bridge between Java and native (Rust) code inside the same process. |
| **Shard** | In Hyperscale, each partition of the state with its own consensus. |
| **Beacon chain** | In Hyperscale, the slow chain that manages validators and the shard topology. |
| **BLS12-381** | The signature scheme Hyperscale uses for validators; many signatures can be aggregated into one. |
| **WASM** | A portable executable code format. Scrypto compiles to WASM, and the Hyperscale VM also runs WASM. |
| **RAC (Radix Accountability Council)** | An elected council that bridges governance while the DAO is being constituted. |
| **DAO** | The community organisation that governs the network after the Foundation, with decisions and funding on ledger. |
| **Mainnet / Stokenet** | Mainnet is Radix's main network. Stokenet is its public test network. |
| **Radix Foundation** | The organisation that developed Radix and ran its central services; in maintenance mode since 2026. |
| **Radix Dashboard** | The network's official explorer, run by the Foundation; its source code is not public. |
| **Developer Console** | The official web tool to deploy packages and submit manifests; its source code is not public. |
| **Connector Extension** | A browser extension that connects desktop dApps to the mobile wallet. |
| **Mesh API (Rosetta)** | A standard API for exchanges offered by the node; removed in release `v1.4.0.0`. |
| **DataAggregator** | The part of the Gateway that reads the ledger from nodes and stores it in PostgreSQL. |
| **Layer 1 / Layer 2** | Layer 1 is the network itself (protocol and nodes). Layer 2 covers solutions that run on top of or beside it, such as rollups or partner chains. |
| **Partner chain** | An independent network linked to another, as Midnight is to Cardano. |
| **LSU (Liquid Stake Unit)** | A token representing XRD staked with a specific validator. |
| **Pool unit** | A token representing a share of a liquidity pool. |
| **Badge** | A resource used as a credential to obtain permissions. |
| **Securified account** | An account whose control moves from a single key to more complex access rules (multiple factors, recovery). |
| **MFA / shields** | The multi-factor security configuration of Radix wallets. |
| **Persona** | A user's sign-in identity on Radix, separate from their accounts. |
| **RNS (Radix Name Service)** | A service of human-readable names for Radix addresses. |
| **Account locker** | A native component that holds resources for an account to claim. |
| **KV store** | A key-value store inside a component. |
| **Blueprint / component / package** | A package contains blueprints (contract templates); a component is an instance of a blueprint. |
| **Vault** | A container of resources (tokens or NFTs) inside an account or component. |
| **Royalties** | A fee a developer can charge when their package or component is used. |
| **Intent hash** | The identifier of a transaction on Radix. |
| **Moratorium** | A mechanism added in `v1.4.0.0` (commit `8ea6cf97`) to temporarily halt user transactions. |
| **Report codes** | E1-E6: strategies (chapter [4](04-05-strategies.md#s4)). N1-N9 and N5b: capabilities missing from the node (chapter [3](03-what-the-node-offers.md#s3) and section [7.2](07-per-repository-changes.md#s7-2)). (D): on-ledger discovery; (L): the `LedgerBackend` interface (chapter [6](06-recommended-solution.md#s6)). R1-R11: changes to lighten nodes (chapter [9](09-lighter-nodes.md#s9)). M1-M3: migration strategies to Hyperscale (chapter [11](11-hyperscale-evolution.md#s11)). (a)-(j): light-indexer indexes (section [7.1](07-per-repository-changes.md#s7-1)). |
| **Priority P0-P2 / effort S-XL** | P0 essential, P1 needed, P2 improvement. S days, M 1-3 weeks, L more than a month, XL several months or more. |
| **Access point (endpoint)** | A network address to which an application sends its requests. |
| **OpenAPI** | A standard format for describing a web API (routes, requests and responses). |
| **CORS** | A mechanism that lets a web page query an API hosted on another domain. |
| **TLS** | Encryption of web connections (HTTPS). |
| **Rate limit** | The maximum number of requests accepted from one origin within a period. |
| **Failover** | Automatically switching to another server when the current one fails. |
| **Backoff** | Retrying after waits that grow with each attempt. |
| **Mempool** | The set of transactions a node has received and that are awaiting commitment. |
| **Heap / RSS / swap** | Heap: memory reserved for Java. RSS: physical memory a process actually uses. Swap: disk space the system uses as extra memory. |
| **SST / memtable** | RocksDB's on-disk data files / in-memory tables where RocksDB accumulates writes before flushing them to disk. |
| **Vertex store** | The consensus structure that keeps proposed blocks not yet committed. |
| **HotStuff / BFT / finality** | HotStuff is Babylon's consensus algorithm, a BFT (Byzantine fault tolerant) type. Finality: a committed transaction is never reverted. |
| **QC (quorum certificate)** | A set of validator signatures attesting to a consensus agreement. |
| **Accumulator** | A hash structure summarising all committed transactions or receipts. |
| **`state_tree_hash`** | The root of the state tree, included in the ledger proofs that validators sign. |
| **QUIC / iroh** | QUIC is a network transport protocol; iroh is a library that uses it for direct connections between devices. |
| **WebRTC / signalling / TURN / STUN** | WebRTC connects browser and wallet; signalling puts them in contact; TURN and STUN help traverse private networks. |
| **IPFS / BitTorrent / Arweave** | Decentralised file storage and distribution systems. |
| **ZK / zk-SNARK / Halo2** | Zero-knowledge proofs: they prove something without revealing the data. A zk-SNARK is a kind of short proof; Halo2 is a specific system. |
| **HBOR / Blake3 / ML-DSA** | In Hyperscale: a data encoding format, a hash function and a post-quantum signature scheme, respectively. |
| **Person-month** | A unit of effort: one person working full time for one month. |

---

[← Index](../../README.md) · [Previous: Sources, methodology and open questions](sources-and-methodology.md)
