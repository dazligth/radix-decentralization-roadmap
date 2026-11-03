[← Index](../../README.md)

# Glossary

Terms used in the report, briefly explained.

| Term | Meaning |
|---|---|
| **Babylon** | The current version of the Radix network, live since September 2023. |
| **Olympia** | The previous version of the Radix network (2021-2023). Its state was migrated to Babylon. |
| **Hyperscale** | Radix's next-generation design: a sharded network that scales by adding shards. Its Rust implementation, `hyperscale-rs`, is open source. |
| **Xi'an** | The name the Radix roadmap gave to the Hyperscale-based stage of the network. |
| **Gateway (Babylon Network Gateway)** | A central service that indexes the network and offers an API to wallets, dApps and explorers. It is a PostgreSQL database (~2.5 TB), an indexer (DataAggregator) and an API. |
| **Full node** | A computer that keeps a copy of the ledger and validates every transaction, without taking part in consensus. |
| **Validator** | A full node that also takes part in consensus and signs blocks. Mainnet has about 100 active ones. |
| **Read node** | Proposed in this report: a full node with a light indexer that serves queries from wallets, dApps and explorers. |
| **Archive node** | A node that keeps the full history and produces verifiable snapshots for other nodes. |
| **Light indexer (`radix-lite-indexer`)** | Proposed in this report: a process that replaces the Gateway, stores only the indexes the node lacks, and takes ~10-50 GB (estimated). |
| **Core API, LTS, Engine State API** | APIs offered by the node itself. LTS is the part of the Core API with long-term compatibility. The Engine State API reads the internal state of any entity. |
| **Ledger** | The complete record of committed transactions on the network. |
| **State version** | The sequence number of each transaction in the ledger. Mainnet is past 567 million. |
| **Round update** | A system transaction generated every consensus round. Over 99 % of state versions are round updates. |
| **Epoch** | A period (~5 minutes) after which the validator set may change. |
| **Receipt** | The result of executing a transaction: state changes, events and fees. |
| **Subintent / pre-authorization** | Part of a transaction signed in advance by one user and completed later by another party. |
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
| **RAC / DAO** | The Radix Accountability Council and the community organisation that governs the network after the Foundation. |
| **Person-month** | A unit of effort: one person working full time for one month. |
