[← Index](../../README.md) · [Previous: Section 8: roadmap](06-roadmap.md) · [Next: Section 10: step-by-step implementation guide](08-implementation-guide.md)

# 9. Lighter nodes: target of 4 GB RAM and 500 GB disk

**Goal.** A mainnet validator or full node should run on **4 GB of RAM and 500 GB of disk**, and the network should keep working without the Gateway.

**Code reviewed:**

- `babylon-node`, branch `develop` (12919a01, 12 Mar 2025);
- the **latest published release, `v1.4.0.0` "Eagle Ray"** (7400951e, 8/10 Sep 2026);
- `babylon-nodecli`.

`file:line` paths refer to `develop` unless stated otherwise. Differences in `v1.4.0.0` that matter here are called out explicitly.

## 9.1 Current situation: data

| Data point | Value | Source |
|---|---|---|
| Official requirements | 4 CPU, **16 GB RAM**, "500 GB of SSD (initially)" | docs.radixdlt.com, *Node setup* |
| Default memory configuration on mainnet | `-Xms12g -Xmx12g -XX:MaxDirectMemorySize=2048m`, **14 GB** container (`mem_limit: 14000m`) | `babylon-nodecli/node-runner-cli/config/CoreDockerConfig.py:23-32`, `templates/radix-fullnode-compose.yml.j2:17,46` |
| Recommended swap | 8 GB | docs.radixdlt.com, *Optimizing performance* |
| Ledger size: **compressed** (zstd) community snapshot | 6.3 GiB (Oct 2023) → 69 GiB (Sep 2024) → 180 GiB (Sep 2025) → 259 GiB (Jul 2026) → **274 GiB (29 Sep 2026)** | snapshots.radix.live |
| Recent growth | **~5-8 GiB/month** compressed | calculated from the row above |
| Another published snapshot | 343.9 GB (PublicNode, v1.4.0.0, height 566,915,789; not stated whether compressed) | publicnode.com/snapshots |
| Share of the optional indexes | ~4-5 % (274 vs 262 GiB without indexes) | snapshots.radix.live |
| Transaction-history pruning | **Does not exist** | code: only state-tree GC and ledger-proof GC exist |
| Bootstrapping from a verified snapshot | **Does not exist.** A new node replays ~567 million transactions from genesis, or downloads a third-party snapshot with no cryptographic verification. | code; `babylon-nodecli/.../ledger_snapshot/__init__.py:26-80` |
| RocksDB tuning | **None**: `Options::default()` for the database and all 26 column families, also in `v1.4.0.0` | `core-rust/state-manager/src/store/rocks_db.rs:164-170` |
| Mempool | 512 MiB of transactions × factor 2.3 ≈ **1.18 GB** by default | `core-rust/node-common/src/config/mod.rs:74-75`, `RadixNodeModule.java:281-304` |
| Java code remaining | ~55,500 hand-written lines in `core` + ~35,000 in `common` + ~14,000 of JNI bridge | `wc -l` count |
| Official "all-Rust" node | Xi'an / hyperscale-rs is a **different network** (not Babylon). No date since its lead developer withdrew (3 Sep 2026). | radixdlt.com (roadmap 30 Nov 2024), radix.wiki |

**Starting point:** with what is shipped today, a mainnet node **does not fit in 4 GB of RAM**.

- **Disk:** the compressed ledger is already 274 GiB. Uncompressed it is noticeably larger; the exact figure has not been published. So **500 GB of disk is already tight, or soon will be**.

## 9.2 Where the memory goes

### The node today

```mermaid
flowchart LR
    subgraph JVM["Java process (JVM): 12 GB heap by default"]
        CONS["HotStuff consensus<br/>~17,400 lines"]
        P2P["Netty P2P networking<br/>~9,300 lines + DSON/CBOR"]
        SYNC["Ledger sync<br/>and BFT sync"]
        REL["Mempool relay"]
        CFG["Guice + RxJava configuration"]
    end
    JNI{{"JNI bridge<br/>92 functions, SBOR data copied"}}
    subgraph RUST["Rust library (libcorerust.so)"]
        ENG["Radix Engine<br/>execution"]
        SM["State manager"]
        MP["Mempool ~1.18 GB"]
        API["Core API and Engine State API"]
        DB[("RocksDB with no memory limits<br/>full history, no pruning")]
    end
    CONS <--> JNI
    SYNC <--> JNI
    REL <--> JNI
    JNI <--> SM
    SM --> ENG
    SM --> DB
    SM --> MP
    API --> DB
    classDef heavy fill:#ffd6d6,stroke:#c00,color:#000
    class JVM,DB heavy
```

| Consumer | Today | Why | Can be reduced to |
|---|---|---|---|
| Java heap | 12 GB reserved up front (`-Xms12g`) | Installer default. Actual use has not been published; 1.5-7 GB was measured on betanet (2023). | **1-1.5 GB** with limits on Java structures (vertex store, message queues); **0 GB** after moving to Rust |
| Direct memory (Netty) | up to 2 GB | Network buffers | 256 MB; **0** after moving to Rust |
| RocksDB: index and filter blocks of every SST file | **Unbounded**; grows with the ledger | `max_open_files=-1` and `cache_index_and_filter_blocks=false` (defaults) keep the indexes of **all** open files in memory, ~0.5-1.5 % of database size (*estimate*) | Counted inside a shared **512 MB** cache |
| RocksDB: memtables | 64 MB × 2 per column family with no global cap; ~0.6-1.3 GB in practice (*estimate*) | Defaults | **256 MB** in total with `WriteBufferManager` |
| RocksDB: block cache | one per column family (up to ~26 × 32 MB) | Defaults | A single shared cache (the one above) |
| Mempool | ~1.18 GB | 512 MiB default | **64-128 MB** on small validators |
| Vertex store (consensus, Java) | up to 150 MB, fully re-serialized on every change | `VertexStoreJavaImpl.java:223-238` | Same, without re-serialization, once it moves to Rust |
| Result and execution caches (Rust) | 3 LRUs of 10,000 entries and the staging cache | `state_manager.rs:257-264` | Configurable size |

**The decisive factor is untuned RocksDB.** Its memory grows with the size of the ledger and nothing caps it. It can be fixed without touching consensus, with a few dozen lines in `rocks_db.rs` (change R1).

## 9.3 Where the disk goes

Over 99 % of the ledger's ~567 million versions are **round updates**: system transactions, one per consensus round. Each one **permanently** writes:

| Data stored per round update | Uncompressed bytes (estimated from the schemas) | Pruned? | What needs it? |
|---|---|---|---|
| Raw transaction (`raw_ledger_transactions`) | ~50 | No | Serving sync to other nodes |
| Identifiers (`committed_transaction_identifiers`) | ~200 (hashes, barely compress) | No | Node startup, APIs; for user transactions, duplicate validation |
| **Receipt** (`transaction_receipts`) | **~4-4.5 KB**. ~3.8 KB is the proposal statistics of ~100 validators, stored **twice** (new and previous value, `receipt.rs:105-124`) | No | APIs only, except epoch-change receipts (used by protocol-update readiness) |
| Local execution (`local_transaction_executions`) | ~0.7-1.5 KB | No (enabled by default) | APIs only |
| Accumulator slices (`transaction/receipt_accu_tree_slices`) | ~0.9-1.3 KB (hashes) | **No** | Only the latest slice of the current epoch is read |
| Transaction-hash index (`ledger_transaction_hashes`) | 40 | No | Duplicate check; no API reader |
| Ledger proofs | ~5-8 KB | Yes, after 12 epochs ~1 in 1,000 remain | Sync |
| State tree (JMT) nodes | ~3-6 KB | Yes, 60,000-version window | Consensus (state root) |

- **Estimated permanent cost per round update:** ~2.5-4 KB on disk with local executions enabled (after default Snappy compression). Over ~560 million rounds, **in the order of 1-2 TB uncompressed**.
- **Order of importance:** receipts > accumulator slices > local executions > identifiers. Raw round-update transactions only add up to ~25-30 GB.
- **How to confirm:** the node already publishes per-column-family sizes (`raw_db_size` metrics, `SM/metrics.rs:590-640`). The **first step** of the plan is to publish those figures from a mainnet node.
- **Root cause:** the ConsensusManager's `CurrentProposalStatistic` field, ~1.9 KB, is rewritten in full every round. Its previous value is stored in the receipt and its path is rewritten in the state tree.
- **New in `v1.4.0.0`:**
  - It adds the option **`db.keep_previous_substate_values`**: with `false`, receipts stop storing previous values (`rocks_db.rs:705-714` in `v1.4.0.0`), which **roughly halves each round's receipt**.
  - It is **on by default** (`true`) on mainnet and only applies to data written from then on.
  - `v1.4.0.0` also **removes the Mesh API**.

## 9.4 Changes: what to do, where and how

Each change states:

- **where** to make it;
- **how** to make it;
- **how to test** it;
- estimated **savings**;
- **risk**.

Changes R1-R7 **do not touch consensus** and can ship in any node release. R8 and R9 change how nodes sync. R10 is a protocol update. R11 is the Java-to-Rust migration.

### R1. Configure RocksDB with memory limits (P0, effort S-M)

- **Where:**
  - `core-rust/state-manager/src/store/rocks_db.rs:164-170` (and the read-only and secondary opens, `:201-207`, `:237-243`);
  - `core-rust/p2p/src/rocks_db.rs:117-123`;
  - new properties in `RadixNodeModule.java` and `DatabaseConfig`.
- **How:**

```rust
// One cache and one memtable budget for the whole node.
let cache = Cache::new_lru_cache(cfg.block_cache_bytes);            // e.g. 512 MiB
let wbm = WriteBufferManager::new_write_buffer_manager_with_cache(
    cfg.memtable_budget_bytes, /*allow_stall=*/ true, cache.clone()); // e.g. 256 MiB

let mut db_opts = Options::default();
db_opts.create_if_missing(true);
db_opts.create_missing_column_families(true);
db_opts.set_max_open_files(cfg.max_open_files);                   // e.g. 1,024
db_opts.set_write_buffer_manager(&wbm);
db_opts.set_max_background_jobs(cfg.background_jobs);              // e.g. 2

let cf_opts = |cf: &str| {
    let mut t = BlockBasedOptions::default();
    t.set_block_cache(&cache);
    t.set_cache_index_and_filter_blocks(true);
    t.set_pin_l0_filter_and_index_blocks_in_cache(true);
    if is_point_lookup_cf(cf) { t.set_bloom_filter(10.0, false); } // substates, JMT, hash indexes
    let mut o = Options::default();
    o.set_block_based_table_factory(&t);
    o.set_level_compaction_dynamic_level_bytes(true);
    o.set_write_buffer_size(if is_hot_cf(cf) { 32 << 20 } else { 8 << 20 });
    o.set_compression_type(DBCompressionType::Lz4);
    if is_append_only_cf(cf) {                                      // receipts, executions, transactions
        o.set_bottommost_compression_type(DBCompressionType::Zstd);
    }
    o
};
```

- **Note on the code:** the node uses the `rocksdb` crate 0.21 (`core-rust/node-common/Cargo.toml:32`). `WriteBufferManager` arrives in later versions of the crate, so it must be **upgraded**. The snippet is indicative: check the exact API names in the chosen version.
- **New properties:**
  - `db.rocksdb.block_cache_mb` (512);
  - `db.rocksdb.memtable_budget_mb` (256);
  - `db.rocksdb.max_open_files` (1,024);
  - `db.rocksdb.compression` (`lz4+zstd`).
- **Tests:**
  - sync a Stokenet node, and a mainnet node from a snapshot;
  - measure RSS, commit latency, sync throughput and compaction;
  - compare the state hash with an unmodified node.
- **Savings:**
  - RocksDB RAM capped at **~0.8 GB**, instead of several GB and growing;
  - **25-40 % less disk** for receipts and executions as they get compacted (*estimate*).
- **Risk:** low; cold reads somewhat slower.

### R2. A "small node" profile in the installer (P0, S)

- **Where:** `babylon-nodecli` (`CoreDockerConfig.py`, `CoreSystemDConfig.py`, templates `radix-fullnode-compose.yml.j2` and `systemd-environment.j2`).
- **How:** a new `--profile small-4gb` with:
  - `JAVA_OPTS="-Xms1g -Xmx1536m -XX:MaxDirectMemorySize=256m -XX:+UseG1GC -XX:MaxGCPauseMillis=200"`;
  - `mem_limit: 3500m`;
  - `RADIXDLT_MEMPOOL_MAX_MEMORY=134217728` (128 MB);
  - lower P2P channel limits (`network.p2p.max_inbound_channels` and `max_outbound_channels` from 1,024 to ~64);
  - the R1 and R3 options.
- **Same repository, security bug:** the snapshot download uses `verify=False` (no TLS verification) in `ledger_snapshot`. TLS must be verified, along with a **published signature or hash** of the snapshot.
- **Tests:** a Stokenet validator with 4 GB for several weeks, measuring missed rounds and RSS.
- **Risk:** low. If the heap is too small the node restarts; the long test prevents that.

### R3. Validators without API indexes (P0, S)

- **What:** on validators, `db.local_transaction_execution_index.enable=false` and `db.account_change_index.enable=false`.
- **Where:** today they cannot be disabled on an existing database (`traits.rs:124-133` rejects it). Allow the `true` → `false` switch:
  - drop the column families (`drop_cf` or `delete_range`);
  - run a manual compaction;
  - store the new option state in `extensions_data`.
- **Savings:** ~0.7-1.5 KB per version, **possibly hundreds of GB** on mainnet.
- **What is lost:** on that node, `/stream/transactions`, `/transaction/receipt` and the LTS history. Read nodes (section 6) keep them.
- **Risk:** low; the API handlers already check the option.

### R4. Receipts without previous values, old ones included (P0, S)

- **What:**
  1. Make **`db.keep_previous_substate_values=false`** the default on validators (already exists in `v1.4.0.0`).
  2. Add a RocksDB **compaction filter** on the `transaction_receipts` column family that strips `previous` from old receipts as they are compacted. It is the same loop as `rocks_db.rs:705-714`, applied in the filter.
- **Savings:** ~1.9 KB per round update uncompressed; **in the order of hundreds of GB** across history (*estimate*, to be confirmed with `raw_db_size`).
- **Risk:** APIs stop returning the previous value; `v1.4.0.0` already accepts that.

### R5. Prune accumulator slices from closed epochs (P1, S)

- **Where:** the ledger-proof GC (`SM/store/proofs_gc.rs`) or a new GC.
- **How:** `delete_range` on `transaction_accu_tree_slices` and `receipt_accu_tree_slices` up to the start of the epoch before the current one. Each epoch starts a new tree from the previous root (`staging/epoch_handling.rs:174-200`), and only the latest slice is read (`accumulator_tree/tree_builder.rs:110-113`).
- **Savings:** ~0.9-1.3 KB per round, of data that **does not compress**: **hundreds of GB** (*estimate*).
- **Risk:** low. It removes the ability to build historical transaction-inclusion proofs, which no API offers today. Archive nodes can keep them.

### R6. Do not index round-update hashes (P2, S)

- **Where:** `rocks_db.rs:676-690`.
- **How:** skip the `put` and the duplicate check when the transaction is a `RoundUpdate`. The engine already rejects non-increasing rounds.
- **Savings:** ~20-30 GB and some index RAM.

### R7. Optional round-update receipts (P1, S-M)

- **What:** an option `db.round_update_receipts=epoch_only` that keeps only epoch-change receipts, the only ones the core reads (`protocol_state.rs:599-616`).
- **Why it is safe:** the receipt root stays in the identifiers and in the signed proofs, so consensus is unchanged. Any receipt can be recomputed by replaying the transaction.
- **Savings:** the **largest remaining component** after R4.
- **Risk:** receipt APIs for old rounds, and the rebuild of entity indexes (`rocks_db.rs:1541`), would have to replay. That is why it is optional and aimed at validators.

### R8. "Pruned node" mode (P1, M-L)

- **What:** an option `ledger.retain_epochs=N`. It range-deletes transactions, identifiers, receipts and executions older than N epochs. It keeps:
  - epoch and protocol-update proofs;
  - epoch-change receipts;
  - the `intent_hashes` index and the identifiers of user transactions within the validity window, which `validation.rs:74-81` needs.
- **Code changes:**
  1. **Gap guard** in `get_syncable_txns_and_proof` (`rocks_db.rs:1011-1066`): today it does not check that the first transaction returned is at the requested version, so it could assemble an inconsistent response.
  2. **Advertise the oldest servable version** in sync status messages (`RemoteSyncService.java:128-166`), so peers do not ask for what is gone.
  3. Review `max_state_version` and `get_first_proof` (`rocks_db.rs:990-995, 1134-1138`).
- **Savings:** almost all of the history. A validator would hold state plus N epochs (**tens of GB up to ~200 GB**, depending on N; *estimate*).
- **Risk:** the network needs **enough archive nodes** for new nodes to bootstrap from. Without R9, new nodes depend on them.

### R9. Bootstrap from a verified state snapshot: "fast sync" (P0 for the goal, L)

A new node starts **in hours, without replaying 567 million transactions and without trusting whoever provides the snapshot**.

- **Export** (on any node, at a version V that ends an epoch):
  - the `substates` and `substate_node_ancestry_records` column families;
  - identifiers and the last accumulator slice at V;
  - epoch proofs since genesis;
  - protocol state;
  - a manifest with per-chunk hashes.
- **Distribution:** HTTP, BitTorrent or IPFS. Where it comes from does not matter, because it is verified.
- **Import and verification:**
  1. Verify the **chain of epoch proofs** from genesis, or from a checkpoint shipped in the binary: each proof is signed by more than 2/3 of the previous validator set's stake.
  2. Load the substates and **rebuild the state tree** (JMT).
  3. Check that the rebuilt root equals the `state_tree_hash` in the signed proof at V. If it does not match, the snapshot is rejected.
  4. Continue normal sync from V.
- **Where:**
  - a new `babylon-node snapshot export` / `import` command (Rust, `state-manager`), reusing the checkpoint infrastructure (`node-common/src/store/rocks_db.rs:214`);
  - in Java, `LocalSyncService` must accept starting from V.
- **Risk:** the trust model (initial checkpoint) and the consistency of protocol state must be designed carefully. It is the most delicate change in this group, but it **does not change consensus**.
- **Effect:**
  - No node needs the full history except archive nodes.
  - The dependence on unverified third-party snapshots disappears.
  - 500 GB stops being a long-term problem.

### How a node bootstraps with R9

```mermaid
sequenceDiagram
    autonumber
    participant NEW as New node
    participant SRC as Any source: archive, IPFS, another node
    participant NET as Radix network
    NEW->>SRC: download the state snapshot at version V
    NEW->>SRC: download epoch proofs from the checkpoint
    NEW->>NEW: verify each proof: signed by more than 2/3 of stake
    NEW->>NEW: rebuild the state tree from the substates
    alt the root matches the signed state_tree_hash at V
        NEW->>NET: sync normally from V
        NET-->>NEW: new transactions
    else it does not match
        NEW->>NEW: discard the snapshot and try another source
    end
```

### R10. Protocol update: per-validator proposal statistics (P2, M; needs a validator vote)

- **What:** split `CurrentProposalStatistic` (~1.9 KB rewritten every round) into a collection with one entry per validator, so each round only rewrites the proposer's entry (~20 bytes).
- **Where:**
  - the ConsensusManager blueprint in `radix-engine`;
  - a new protocol update in `babylon-node` (`protocol/protocol_updates/definitions/`), like Bottlenose, Cuttlefish or Eagle Ray.
- **Savings:** round receipts and state-tree rewrites drop by an order of magnitude **for everything written after the update**.
- **Risk:** it is a protocol update; it needs testing and validator readiness signalling.

### R11. Migrate the node core from Java to Rust (P1 for 4 GB with headroom, L-XL)

**What remains in Java** (`develop`):

| Area | Approx. lines | Anything already in Rust? |
|---|---|---|
| Consensus: HotStuff, pacemaker, safety rules, vertex store, epochs | ~17,400 | Persistence (safety store, vertex store) and computing the next validator set |
| P2P networking: Netty, RLPx-style ECIES handshake, AES-CTR + Keccak MAC encryption, discovery, address book | ~9,300 | Only address-book storage (`core-rust/p2p`) |
| Messaging and DSON/CBOR serialization (47 message types) | ~5,400 + ~7,400 | No |
| Ledger sync and BFT sync | ~4,900 | The serving side (`getSyncableTxnsAndProof`) |
| Mempool relay | ~1,700 | The whole mempool |
| Configuration, Guice and RxJava | ~7,000 | `StateManagerConfig` |
| System API and Prometheus | ~3,300 | The metrics registry |
| Keys and keystore | ~750 + crypto | secp256k1 primitives |
| **Disappears with the migration:** Java SBOR, bridge DTOs, JNI layer | ~7,000 + ~14,000 + ~4,000 | — |

**How to do it without putting the network at risk** (*strangler* pattern: replace piece by piece, with the node always working):

1. **Phase 0: test foundation.**
   - Port the deterministic Java harness (`core/src/test-core`, ~17,800 hand-written lines) to Rust, or wrap it so the same scenarios run against both implementations.
   - The *sans-io* design of `hyperscale-rs/crates/core` and its simulator are a useful reference.
2. **Phase 1: pieces with no networking.**
   - Vertex store and safety rules in Rust, called from Java via JNI. This removes the full re-serialization of the vertex store on every change.
   - Mempool relay in Rust, which removes the `triggerMempoolRelay` callback.
   - Configuration in Rust.
3. **Phase 2: sync.** Ledger-sync client and BFT sync in Rust. The serving side is already in Rust.
4. **Phase 3: P2P networking in Rust, byte-compatible.**
   - Handshake, channel encryption and Snappy.
   - A DSON/CBOR codec for the 47 messages.
   - **Compatibility tests** with vectors captured from real Java nodes.
   - Deploy on full nodes first, then validators. Patterns from `hyperscale-rs/crates/network` (retries, backoff, handler registry) can be reused.
5. **Phase 4: consensus in Rust.**
   - Port HotStuff, pacemaker and epochs.
   - Run in **shadow mode**: the Rust node computes votes, but Java signs, and the two are compared.
   - Then a mixed Java/Rust testnet and, finally, mainnet with gradual adoption.
6. **Phase 5:** retire the JVM, the JNI bridge and the Java code.
7. **Optional, later:** a new transport (QUIC/iroh) negotiated by capability, once every node speaks Rust.

### The node after moving to Rust (R11)

```mermaid
flowchart LR
    subgraph RUST["A single Rust process, ~2 GB RAM"]
        CONS["HotStuff consensus in Rust"]
        P2P["P2P networking in Rust<br/>byte-compatible"]
        SYNC["Sync<br/>+ snapshot bootstrap"]
        ENG["Radix Engine"]
        SM["State manager"]
        MP["Mempool 64-128 MB"]
        API["Native read API<br/>+ Merkle proofs"]
        DB[("RocksDB with a shared 512 MB cache<br/>+ configurable pruning")]
    end
    CONS --> SM
    SYNC --> SM
    P2P --> CONS
    P2P --> SYNC
    SM --> ENG
    SM --> DB
    SM --> MP
    API --> DB
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class RUST good
```

**Benefit:**

- The Java heap, metaspace, JIT and Netty direct memory disappear: **~1.5-2.5 GB less** with a small profile (*estimate*).
- One process and one memory allocator.
- No more data copying across JNI.

**Effort:** in the order of **30-50 person-months**, with consensus experts (*estimate*).

**Risk:** high, because it touches consensus. Mitigated by shadow mode, a mixed testnet and gradual adoption.

**Relationship with Xi'an/hyperscale-rs:** it is a new network, with another virtual machine, BLS signatures and sharding. It does not replace Babylon in the short term and, since September 2026, has no date. Migrating the Babylon node to Rust still makes sense. Patterns from hyperscale-rs (networking, memory-bounded RocksDB storage, deterministic testing) can be reused, but not its protocol.

## 9.5 Memory budget with 4 GB

These figures are *estimates* to be validated with R2's long-running test.

| Component | Validator with Java (R1-R4) | Validator without Java (after R11) | Read node with light indexer (section 6), no Java |
|---|---|---|---|
| Operating system and other | 0.4 GB | 0.4 GB | 0.4 GB |
| JVM: heap + metaspace + threads + direct memory | 1.5 + 0.4 GB | — | — |
| RocksDB: shared cache + memtables | 0.5 + 0.25 GB | 0.5 + 0.25 GB | 0.75 + 0.25 GB |
| Mempool | 0.13 GB | 0.13 GB | 0.13 GB |
| Execution, caches and Rust runtime | 0.4 GB | 0.5 GB | 0.5 GB |
| APIs and light indexer | — | — | 0.6 GB |
| **Total** | **~3.6 GB (tight)** | **~1.8 GB (with headroom)** | **~2.6 GB (with headroom)** |

**Reading the table:**

- With Java and the R1-R4 tweaks, a validator **fits in 4 GB, but with no headroom**.
- After the move to Rust it fits comfortably, **even a read node with an indexer**.
- **4 GB of swap** is recommended as a safety net.

## 9.6 Disk budget with 500 GB

- **Today:**
  - 274 GiB compressed (Sep 2026), +5-8 GiB/month.
  - The uncompressed size on a real node has not been published. The first task is to measure it with the per-column-family `raw_db_size` metrics.
  - With defaults, a full node **will exceed 500 GB**: it may already, or will within a few years (*estimate*).
- **With R3 + R4 + R5 + R6 + R7, without pruning history:**
  - Most per-round bytes go away: round receipts, executions, accumulators and the hash index.
  - What remains is mainly raw transactions (~25-30 GB of rounds + user transactions), identifiers (~110 GB uncompressed) and state.
  - Expected size: **in the order of 150-250 GB** for a validator with full transaction history (*estimate*).
- **With R8 + R9 (pruned node that bootstraps from a verified snapshot):**
  - state + N epochs, **tens of GB**;
  - disk stops growing with the age of the network.

```mermaid
flowchart LR
    subgraph V["Light validator: 4 GB RAM, 50-250 GB"]
        V1["Current state"]
        V2["Last N epochs"]
        V3["Epoch proofs"]
    end
    subgraph R["Read node: 4 GB after R11, ≤ 500 GB"]
        R1["Current state"]
        R2["Historical transactions"]
        R3["Light-indexer indexes"]
    end
    subgraph A["Archive node: 8-16 GB RAM, 1-2 TB"]
        A1["Full history"]
        A2["Receipts and accumulators"]
    end
    A -- "verifiable snapshots" --> V
    A -- "verifiable snapshots" --> R
    A -- "full sync" --> R
```

| Node type | RAM | Disk | What it stores |
|---|---|---|---|
| **Light validator** | 4 GB | ≤ 500 GB (estimated 50-250 GB depending on pruning) | Current state, N epochs, epoch proofs; no API indexes (R1-R9) |
| **Read node** | 4 GB after R11 (6-8 GB before) | ≤ 500 GB with R4, R5 and R7 + light indexes | Transaction history + light-indexer indexes |
| **Archive node** (a few, in several hands) | 8-16 GB | 1-2 TB | Full history; serves full sync and produces R9's verifiable snapshots |

## 9.7 Conclusion of this chapter

- **Is a 4 GB RAM / 500 GB disk node possible?** **Yes.**
  - With **R1-R4** (weeks of work, no consensus changes), a validator drops to ~3.6 GB of RAM and its memory stops growing with the ledger.
  - With **R5-R9** (months), disk stays well under 500 GB, and new nodes start in hours from a verified snapshot.
  - With **R11** (more than a year) there is comfortable headroom, and the read nodes that replace the Gateway also fit in 4 GB.
- **Most urgent:** measure the real per-column-family size on a mainnet node, and apply R1 and R2. They are cheap and remove the main reason for the 16 GB.
- **Difficulty:**

  | Changes | Difficulty |
  |---|---|
  | R1-R4, R6 | Low |
  | R5, R7 | Low-medium |
  | R8 | Medium |
  | R9 | Medium-high |
  | R10 | Medium (protocol update) |
  | R11 | **Very high** |

---

[← Index](../../README.md) · [Previous: Section 8: roadmap](06-roadmap.md) · [Next: Section 10: step-by-step implementation guide](08-implementation-guide.md)
