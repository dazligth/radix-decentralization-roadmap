[← Index](../../README.md) · [Previous: Scope and chapters 1-2: current situation, who depends on the Gateway, explorers and Olympia](01-02-current-situation.md) · [Next: Chapters 4-5: the six strategies evaluated and their comparison](04-05-strategies.md)

<a id="s3"></a>

# 3. What the node offers today (verified in `babylon-node`: branch `develop` and release `v1.4.0.0`)

<a id="s3-1"></a>

## 3.1 The node's APIs

- **Core API (port 3333):**
  - `/lts/transaction/{construction, submit, status}`
  - `/lts/state/{account-all-fungible-resource-balances, account-fungible-resource-balance, account-deposit-behaviour}`
  - `/lts/stream/{transaction-outcomes, account-transaction-outcomes}`
  - `/status/{network-configuration, network-status}`
  - `/mempool/*`
  - `/transaction/{parse, submit, status, receipt, call-preview, preview, preview-v2}`
  - `/stream/{transactions, proofs}`
  - `/state/{consensus-manager, account, component, validator, access-controller, resource, non-fungible, package}`
- **Engine State API (port 3336):**
  - `/entity/info`, `/blueprint/info`
  - `/object/field`, `/object/collection/{iterator, entry}`
  - `/kv-store/{iterator, entry}`
  - `/object/attached-modules/{metadata/iterator, metadata/entry, role-assignment, royalty}`
  - `/extra/entity-search` (filters `Blueprint`, `SystemType`, `EntityType`)
- **Mesh API (Rosetta):** it was off by default, and **release `v1.4.0.0` (Sep 2026) removes it**. It served exchanges, not dApps.
- **Exposure:** all of them bind to `127.0.0.1` with no CORS (`RadixNodeModule.java:127-131`).
- **Flags:**

  | Flag | Default | Purpose |
  |---|---|---|
  | `db.account_change_index.enable` | `true` | Per-account history, fungibles only |
  | `db.historical_substate_values.enable` | `false` | Historical reads |
  | `state_hash_tree.state_version_history_length` | `60000` | Size of the historical window |
  | `db.entity_listing_indices.enable` | — | `/extra/entity-search` |
  | `api.core.flags.enable_unbounded_endpoints` | `true` | Endpoints with unbounded cost |

<a id="s3-2"></a>

## 3.2 Mapping to the Gateway

| Capability | With today's node |
|---|---|
| Epoch, network, transaction construction | Yes: `/status/network-status`, `/lts/transaction/construction` |
| Preview v1/v2 (the Gateway only forwards it) | Yes: `/transaction/preview`, `/transaction/preview-v2` |
| Submit and status | Yes: `/lts/transaction/{submit, status}`, with status mapping. LTS statuses are `CommittedSuccess`, `CommittedFailure`, `PermanentRejection`, `InMempool`, `NotSeen`, `FateUncertain` and `FateUncertainButLikelyRejection`. |
| Fungible balances | Yes: `/lts/state/account-all-fungible-resource-balances` |
| An account's NFTs and their data | Yes: in 2-3 calls: the account's `resource_vaults` collection, then vault contents, then `/state/non-fungible` |
| Component state, KV stores, metadata, roles | Yes: Engine State API |
| Validators, LSUs, pools | Yes: `/state/validator`, `/state/consensus-manager`, `/state/component`. The client computes redemption values. |
| Deposit pre-validation | Yes: `/lts/state/account-deposit-behaviour` |
| Transaction result | Yes: `/transaction/receipt` |
| **N1** Subintent status | No: there is no endpoint, but the node **already indexes it**: column family `finalized_subintent_hashes` (`state-manager/src/store/column_families.rs:154-162`, `rocks_db.rs:705-708`) |
| **N2** NFT location (which vault or account holds id X) | No |
| **N3** History filtered by entity, with NFT changes, manifest class and events | Partly: only `/lts/stream/account-transaction-outcomes`, which gives fungible changes for accounts |
| **N4** Entities that use a badge in their rules | No |
| **N5** Historical state of any age | Partly: only ≈ 4-5 h, behind a flag that is off by default |
| **N5b** `last_updated_at_state_version` for metadata | No: used by account recovery |
| **N7** Batch reads (20 entities per call) | No: one call per entity |
| **N8-N9** locker `touched-at` and validator uptime; resource holders (light-indexer index h) | No |
| **N6** Safe public access (CORS, read-only, rate limiting, TLS) | No |
| Resubmission of pending transactions | No: The Gateway does it. With a node, it becomes the client's responsibility: keep the payload and resubmit with backoff while the epoch window is valid. |

---

[← Index](../../README.md) · [Previous: Scope and chapters 1-2: current situation, who depends on the Gateway, explorers and Olympia](01-02-current-situation.md) · [Next: Chapters 4-5: the six strategies evaluated and their comparison](04-05-strategies.md)
