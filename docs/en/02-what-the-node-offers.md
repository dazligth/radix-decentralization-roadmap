[← Index](../../README.md) · [Previous: Sections 1-2: current situation, who depends on the Gateway, explorers and Olympia](01-current-situation.md) · [Next: Sections 4-5: the six strategies evaluated and their data-driven comparison](03-strategies.md)

# 3. What the node offers today (verified in `babylon-node/develop`)

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
- **Mesh API (Rosetta):** off by default. It serves exchanges, not dApps.
- **Exposure:** all of them bind to `127.0.0.1` with no CORS (`RadixNodeModule.java:127-131`).
- **Flags:**

  | Flag | Default | Purpose |
  |---|---|---|
  | `db.account_change_index.enable` | `true` | Per-account history, fungibles only |
  | `db.historical_substate_values.enable` | `false` | Historical reads |
  | `state_hash_tree.state_version_history_length` | `60000` | Size of the historical window |
  | `db.entity_listing_indices.enable` | — | `/extra/entity-search` |
  | `api.core.flags.enable_unbounded_endpoints` | `true` | Endpoints with unbounded cost |

## 3.2 Mapping to the Gateway

| Capability | With today's node |
|---|---|
| Epoch, network, transaction construction | ✅ `/status/network-status`, `/lts/transaction/construction` |
| Preview v1/v2 (the Gateway only forwards it) | ✅ `/transaction/preview`, `/transaction/preview-v2` |
| Submit and status | ✅ `/lts/transaction/{submit, status}`, with status mapping. LTS statuses are `CommittedSuccess`, `CommittedFailure`, `PermanentRejection`, `InMempool`, `NotSeen`, `FateUncertain` and `FateUncertainButLikelyRejection`. |
| Fungible balances | ✅ `/lts/state/account-all-fungible-resource-balances` |
| An account's NFTs and their data | ✅ in 2-3 calls: the account's `resource_vaults` collection, then vault contents, then `/state/non-fungible` |
| Component state, KV stores, metadata, roles | ✅ Engine State API |
| Validators, LSUs, pools | ✅ `/state/validator`, `/state/consensus-manager`, `/state/component`. The client computes redemption values. |
| Deposit pre-validation | ✅ `/lts/state/account-deposit-behaviour` |
| Transaction result | ✅ `/transaction/receipt` |
| **N1** Subintent status | ❌ No endpoint, but the node **already indexes it**: column family `finalized_subintent_hashes` (`state-manager/src/store/column_families.rs:154-162`, `rocks_db.rs:705-708`) |
| **N2** NFT location (which vault or account holds id X) | ❌ |
| **N3** History filtered by entity, with NFT changes, manifest class and events | ⚠️ Only `/lts/stream/account-transaction-outcomes`, which gives fungible changes for accounts |
| **N4** Entities that use a badge in their rules | ❌ |
| **N5** Historical state of any age | ⚠️ Only ≈ 4-5 h, behind a flag that is off by default |
| `last_updated_at_state_version` for metadata | ❌ Used by account recovery |
| Batch reads (20 entities per call) | ❌ One call per entity |
| Validator uptime, resource holders, locker `touched-at` | ❌ |
| **N6** Safe public access (CORS, read-only, rate limiting, TLS) | ❌ |
| Resubmission of pending transactions | ❌ The Gateway does it. With a node, it becomes the client's job: keep the payload and resubmit with backoff while the epoch window is valid. |

---

[← Index](../../README.md) · [Previous: Sections 1-2: current situation, who depends on the Gateway, explorers and Olympia](01-current-situation.md) · [Next: Sections 4-5: the six strategies evaluated and their data-driven comparison](03-strategies.md)
