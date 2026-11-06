---
title: "Mainnet node disk usage per RocksDB column family (measured, v1.4.0.0)"
author: "dazligth"
date: "2026-11-06"
type: "measurement"
related: "docs/en/09-lighter-nodes.md (Where the disk goes, Disk budget); docs/en/06-recommended-solution.md (Comparison with the Gateway)"
status: "proposal"
---

## Summary

Chapter 9 says the on-disk size of a mainnet node and its per-column-family split have not been
published, and lists measuring them as the most urgent step. I have those numbers from my own
mainnet full node and share them here with method and date.

Headline figures (babylon-node v1.4.0.0, mainnet, both optional API indexes enabled, measured
2026-10-10 at state version ~562.3 million):

| What | Size |
|---|---|
| Sum of all live SST files (25 column families, 53,139 files) | 730.4 GiB |
| `transaction_receipts` | 357.9 GiB (49%) |
| `local_transaction_executions` (execution index) | 120.8 GiB |
| `committed_transaction_identifiers` | 112.1 GiB |
| `transaction_accu_tree_slices` + `receipt_accu_tree_slices` | 41.1 GiB each |
| Minimal state-only core (substates, JMT nodes, ledger and epoch proofs) | 16.25 GiB (2.2%) |
| Same node without the two API indexes and without historical substate values (reported by the StakeSafe snapshot operator, 2026-10-25) | ~455 GB |

Two consequences for the report:

1. A read node that keeps full history for the light indexer is about 730 GB today, not under 500 GB.
   The honest comparison in section 6 is therefore "read node ~730 GB + indexer 10-50 GB" against
   "node + Gateway ~2-2.5 TB Postgres", roughly a factor of 3, not 50-250.
2. The direction of R3-R8 is confirmed by the data: 97.8% of the bytes are transaction history and
   derived indexes, and the state-only core is 16 GiB. A pruned validator profile is realistic; a
   read or archive node is not small.

## Assessment

- **Is it possible?** This note does not propose a change, it supplies a measurement.
- **Difficulty:** Not applicable.
- **Effort and timeline (estimate):** Not applicable. Reproducing the measurement takes about one
  hour on an existing node (checkpoint plus `ldb` listing).

## Details

### Setup

- babylon-node `v1.4.0.0` (Eagle Ray), mainnet, Docker image.
- Database restored from the StakeSafe "with indexes" snapshot (epoch 345914, published 2026-10-02),
  then synced to tip. The snapshot keeps substate values; the two optional indexes
  `local_transaction_execution_index` and `account_change_index` are enabled.
  `entity_listing_indices` was enabled afterwards and rebuilt by the node.
- `db.historical_substate_values` disabled (default). Other database options at their defaults.
- Storage: HDD. The sizes are RocksDB SST files on disk, so they are after RocksDB's own block
  compression.

### Method

1. Take a RocksDB checkpoint of the live database (hard links, about 17 s) while the node runs.
   Checkpoint timestamp 2026-10-10T21:42:30Z, tip at that moment about state version 562,269,194.
2. Run `ldb list_live_files_metadata` on the checkpoint to map every live SST file to its column
   family.
3. Sum file sizes per column family with `stat`.

Caveat: opening the checkpoint with `ldb` on the same HDD briefly pushed the live node into
`OUT_OF_SYNC`. On SSD this should not matter.

### Result per column family

| Column family | GiB | SST files |
|---|---:|---:|
| transaction_receipts | 357.94 | 35,856 |
| local_transaction_executions | 120.83 | 11,172 |
| committed_transaction_identifiers | 112.15 | 1,992 |
| transaction_accu_tree_slices | 41.11 | 870 |
| receipt_accu_tree_slices | 41.11 | 870 |
| ledger_transaction_hashes | 21.70 | 367 |
| raw_ledger_transactions | 17.46 | 926 |
| ledger_proofs | 6.48 | 122 |
| state_hash_tree_nodes | 4.54 | 79 |
| epoch_ledger_proofs | 3.66 | 786 |
| substates | 1.57 | 29 |
| intent_hashes | 0.57 | 11 |
| notarized_transaction_hashes | 0.57 | 14 |
| substate_node_ancestry_records | 0.27 | 8 |
| type_and_creation_indexed_entities | 0.14 | 3 |
| blueprint_and_creation_indexed_objects | 0.14 | 4 |
| account_change_state_versions | 0.12 | 5 |
| stale_state_hash_tree_parts | 0.04 | 5 |
| remaining (protocol proofs, vertex store, extensions, associated values, ...) | ~0 | |
| **Total** | **730.39** | **53,139** |

`associated_state_tree_values` is empty, which confirms that historical substate values were off.

### Grouping

| Group | Column families | GiB | Share |
|---|---|---:|---:|
| State-only core (what a node needs to keep executing and proving) | substates, state_hash_tree_nodes, associated_state_tree_values, ledger_proofs, epoch_ledger_proofs | 16.25 | 2.2% |
| Reconstructible history and indexes | receipts, executions, identifiers, accumulator slices, hash indexes, raw transactions, ancestry, listing | 714.09 | 97.8% |
| Other | stale tree parts, extensions | 0.04 | |

### Whole-directory sizes, for comparison

| Variant | Size | Source |
|---|---|---|
| StakeSafe "with indexes" archive, epoch 345914 | ~295 GB (tar, compressed) | download, 2026-10 |
| Same database unpacked | ~728 GB | `du`, 2026-10 |
| Same database synced to tip | ~732 GB | `du`, 2026-10-08 |
| Live node without the two API indexes and without historical substate values | ~455 GB | reported by the StakeSafe operator (Bart), 2026-10-25 |

So the two optional indexes plus related data account for roughly 455 GB to 730 GB. Recompressing
a sample of the SST files gave a ratio of about 0.68, so stronger compression (R1's bottommost
zstd) has around 30% left to gain, not more.

### How this relates to chapter 9

- The ranking in "Where the disk goes" (receipts > accumulator slices > executions > identifiers)
  is close but not exact on disk: measured it is receipts > executions > identifiers > slices.
  Executions and identifiers compress worse than the schema estimate suggests.
- Raw round-update transactions are indeed small: `raw_ledger_transactions` is 17.5 GiB for the
  whole ledger.
- The two optional indexes are not "4-5%" of the node. On this node they are the difference between
  ~455 GB and ~730 GB. The 4-5% figure in chapter 9 compares compressed snapshots, where index data
  compresses far better than it sits on disk.
- Growth: during the listing index rebuild the directory grew by up to ~20 GB per day before
  compaction reclaimed part of it. Steady-state growth at tip was not measured separately.

### Suggested edits to the report

- Section 6, "Comparison with the Gateway": replace "about 50 to 250 times less storage" with the
  node-inclusive comparison above.
- Chapter 9, "Disk budget": state that a full node with both indexes is ~730 GB today (measured) and
  ~455 GB without them (reported), so the 500 GB target already requires R3 or pruning for any
  node that is not an archive.
- Chapter 9, "Node types": the read node that serves the light indexer needs the full
  `transaction_receipts` and the execution index, so its disk budget starts at ~730 GB, not at
  "probably ≤ 500 GB".

## Sources

- Own measurement, method and date as described above. Raw per-file listing available on request.
- StakeSafe snapshots and documentation: https://snapshots.stakesafe.net/ and https://docs.stakesafe.net/
- babylon-node v1.4.0.0: https://github.com/radixdlt/babylon-node/releases/tag/v1.4.0.0
- Column family definitions: `core-rust/state-manager/src/store/column_families.rs` at v1.4.0.0
