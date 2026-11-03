[← Index](../../README.md) · [Previous: Chapter 9: lighter nodes (RAM and disk) and the Java to Rust migration](09-lighter-nodes.md) · [Next: Chapter 11: final stage B, migration to Hyperscale](11-hyperscale-evolution.md)

<a id="s10"></a>

# 10. Step-by-step implementation guide

The sections of this chapter inherit the assessment (is it possible?, difficulty and effort) of what they implement:

- 10.1 → the box of strategy E3 (chapter [4](04-05-strategies.md#s4));
- 10.2 → its own box;
- 10.3 → the box of strategy E2 (chapter [4](04-05-strategies.md#s4));
- 10.4 → the box of section [10.2](#s10-2) and the conclusion table;
- 10.5 → its own box;
- 10.6 → the box of section [6.2](06-recommended-solution.md#s6-2);
- 10.7 → no assessment of its own (it is a procedure);
- 10.8 → its own box.

This chapter explains **how** to build the main pieces. Code snippets are **indicative**: they show the shape of the solution, not production-ready code, and must be adjusted to the exact version of each dependency.

<a id="s10-1"></a>

## 10.1 `radix-lite-indexer`: the Gateway replacement

The diagrams of how a read node works and how transactions are submitted are in chapter [6](06-recommended-solution.md#s6).

### Architecture

```
radix-lite-indexer/
├── crates/
│   ├── core-client/   # Typed client for the Core API and Engine State API (generated from the node's YAML)
│   ├── ingest/        # Stream ingestion loop and index writing
│   ├── store/         # Indexes in RocksDB (or SQLite): schemas and queries
│   ├── api/           # HTTP server (axum): routes in the format apps use today
│   ├── submit/        # Transaction submission and resubmission
│   └── snapshot/      # Export and import of signed indexes
├── conformance/       # Tests comparing responses with the public Gateway
└── deploy/            # docker-compose, systemd, babylon-nodecli template
```

One important advantage: **Babylon has immediate finality** (BFT consensus), so a committed transaction is never reverted. The indexer **does not need to handle chain reorganisations**, which makes it much simpler than indexers for other networks.

### Data model

All keys are byte-ordered so they can be scanned by range.

| Table | Key | Value | Written on | Serves |
|---|---|---|---|---|
| `entity_tx` | `entity_address ‖ state_version (BE)` | bits: manifest class, balance changed?, event emitted?, deposit/withdrawal? | every user transaction and genesis | filtered history |
| `entity_tx_by_resource` | `entity ‖ resource ‖ state_version` | delta (fungible) or ids added/removed (NFT) | same | per-resource history, `balance_changes` |
| `nf_location` | `resource ‖ local_id` | `vault ‖ global_owner ‖ state_version` | NFT vault changes | `state/non-fungible/location`, explorers |
| `holders` | `resource ‖ inverted balance ‖ owner` (to scan them ordered by balance), with an auxiliary `resource ‖ owner → balance` index for updates | balance | vault changes | `extensions/resource-holders`, explorers |
| `subintents` | `subintent_hash` | `state_version ‖ intent_hash` | subintent nullifications | `transaction/subintent-status` (until N1 exists) |
| `role_badges` | `badge (resource[:id]) ‖ entity` | — | `RoleAssignment` changes | `entities-by-role-requirement` |
| `metadata_updated` | `entity ‖ key` | `state_version` | metadata-module changes | `last_updated_at_state_version` |
| `ts_index` | `timestamp_ms` (per minute) | `state_version` | every transaction | date navigation |
| `stats_hourly` | `hour` | counters | every transaction | statistics |
| `search_terms` | `normalised term ‖ entity` | — | `name`, `symbol`, `tags` metadata | search |
| `validator_epoch` | `validator ‖ epoch` | proposals made and missed | epoch-change events | uptime |
| `locker_touch` | `account ‖ locker` | `state_version` of the last change | account-locker changes | `state/account-lockers/touched-at` (index f) |
| `cursor` | `"cursor"` | last processed `state_version` | each batch | resume |

### Ingestion loop

Pseudocode:

```rust
loop {
    let from = store.cursor()? + 1;
    let page = core.stream_transactions(from, LIMIT /* e.g. 1000 */).await?;   // /core/stream/transactions
    if page.transactions.is_empty() { sleep(500ms).await; continue; }

    let mut batch = store.batch();
    for tx in page.transactions {
        match tx.ledger_transaction.kind() {
            RoundUpdate => { stats.count_round(&mut batch, &tx); continue; }   // entities are not indexed
            Genesis | Flash | ProtocolUpdate | User => {}
        }
        let changes = decode_state_changes(&tx.receipt);                       // created, updated, deleted
        index_entities(&mut batch, &tx, &changes);       // entity_tx, entity_tx_by_resource
        index_vaults(&mut batch, &tx, &changes);         // nf_location, holders (vault -> global ancestor via ancestry)
        index_roles_and_metadata(&mut batch, &tx, &changes);
        index_subintents(&mut batch, &tx);
        index_time_and_stats(&mut batch, &tx);
    }
    batch.set_cursor(last_state_version);
    store.write(batch)?;                                  // atomic: indexes and cursor together
}
```

**Important details:**

1. **Genesis.** Start at version 1. The genesis transactions (versions 1-647) contain Olympia's migrated state and have very large receipts. Process them as a stream, without loading the whole receipt into memory.
2. **A vault's global ancestor.** The receipt says which vault changed. To know which account or component owns it, use the node's ancestry: store it when the vault is created, or query `/entity/info` the first time.
3. **Atomic writes.** Indexes and cursor are written in the same batch. If the process crashes, it resumes from the cursor, and repeated work is harmless because writes are idempotent.
4. **Performance.** For the first sync (~567 million versions), fetch large batches in parallel and write in order. Round updates are only counted.
5. **Data it requests from the node.** Request `transaction_format_options.balance_changes=true` and use those balance changes, which come from the local execution and not from the receipt's previous values. That way the read node can apply R4. The node needs the local execution index, but not the account change index (section [9.6](09-lighter-nodes.md#s9-6)).
6. **Response cache.** Keep per-entity responses in memory together with their `state_version`, and invalidate them when the ingestion loop indexes a transaction that touches that entity (section [6.4](06-recommended-solution.md#s6-4)).

### API layer

An `axum` server with these layers:

- `tower_http::cors::CorsLayer`;
- a per-IP limiter (for example `tower_governor`);
- a request-size limit;
- a per-request timeout.

Each route resolves its data in one of these ways:

| Route (format apps use today) | Resolved with |
|---|---|
| `status/gateway-status`, `transaction/construction` | Core: `/status/network-status`, `/lts/transaction/construction` |
| `transaction/preview`, `preview-v2` | Core: forwarded directly |
| `transaction/submit` | `submit` module (see "Transaction resubmission" below) |
| `transaction/status` | Core `/lts/transaction/status` + the `submit` module's pending table, with status mapping |
| `transaction/subintent-status` | `subintents` index (or N1) |
| `transaction/committed-details` | Core `/transaction/receipt` |
| `transaction/account-deposit-pre-validation` | Core `/lts/state/account-deposit-behaviour` |
| `state/entity/details` | Core `/state/*` + Engine State (metadata, roles, fields); aggregates per resource; `last_updated` from `metadata_updated` |
| `state/entity/page/fungibles` | Core `/lts/state/account-all-fungible-resource-balances` (accounts) or the component's vaults |
| `state/entity/page/non-fungibles`, `non-fungible-vault/ids` | Engine State `/object/collection/iterator` |
| `state/non-fungible/data` | Core `/state/non-fungible` |
| `state/non-fungible/location` | `nf_location` index |
| `state/key-value-store/*` | Engine State `/kv-store/*` |
| `state/validators/list` | Core `/state/consensus-manager` + `/state/validator` |
| `stream/transactions` (with filters) | `entity_tx*` indexes + Core `/transaction/receipt` for details |
| `extensions/entities-by-role-requirement/lookup` | `role_badges` index |
| `extensions/resource-holders/page` | `holders` index |
| `statistics/validators/uptime` | `validator_epoch` index |
| `state/account-lockers/touched-at` | `locker_touch` index |
| `state/account-locker/page/vaults`, `state/account/page/{resource-preferences, authorized-depositors}`, `state/entity/page/{metadata, fungible-vaults, non-fungible-vaults}` | Engine State API (collections and metadata) and Core |

**Consistent reads.**

- When a paged request starts, pin `state_version = min(node, indexer cursor)` and include it in the paging cursor.
- If the node has moved on and the current read cannot be pinned at that version, return the page with an updated `ledger_state`; the API format already allows this.
- For exact reads in the recent past, enable `db.historical_substate_values.enable` on the read node.

### Transaction resubmission (`submit` module)

```text
on submit(payload):
  validate format and epoch window (start <= current_epoch+1 < end)
  store in `pending` table: {intent_hash, payload, first_sent, attempts=0, next_attempt}
  send to Core /lts/transaction/submit
every 5 s:
  for each pending item with next_attempt <= now:
    status = Core /lts/transaction/status
    if CommittedSuccess/CommittedFailure/PermanentRejection -> close
    if current_epoch >= end_epoch                          -> close as expired
    if attempts >= 5 or 5 min have passed since the first send -> stop resubmitting (status is still checked)
    if NotSeen or FateUncertain                            -> resubmit; attempts += 1;
                                                              next_attempt = now + 10 s * 2^attempts
```

These are the same parameters the Gateway uses today: 10 s base, ×2, up to 5 attempts, 5 minutes at most.

### Conformance tests

1. **Before the public Gateway is switched off:** record real requests and their responses, sampling the routes in the table above with varied addresses (old Olympia accounts, accounts with NFTs, validators, pools).
2. **Compare** with what the indexer returns, field by field, ignoring fields that change over time (`ledger_state`).
3. **Continuous integration:** every indexer change must pass these tests.
4. **Test with real apps:** Android/iOS wallets on Stokenet and mainnet pointed at the indexer via Settings → Gateways. Walk through balances, NFTs, staking, history, submission and pre-authorisations.

### Deployment next to the node

```yaml
# docker-compose (excerpt)
services:
  core:            # full node, as today
    image: radixdlt/babylon-node:v1.4.0.0
    environment:
      RADIXDLT_DB_HISTORICAL_SUBSTATE_VALUES_ENABLE: "true"   # consistent recent reads
      RADIXDLT_ENTITY_LISTING_INDICES_ENABLE: "true"
      RADIXDLT_ENGINE_STATE_API_PORT: "3336"
  indexer:
    image: radix-lite-indexer:latest
    environment:
      CORE_API_URL: http://core:3333/core
      ENGINE_STATE_API_URL: http://core:3336/engine-state
      PUBLIC_BIND: 0.0.0.0:8080
      RATE_LIMIT_RPS: "20"
    volumes: [ "indexer-data:/data" ]
  proxy:           # TLS
    image: caddy:2
    ports: [ "443:443" ]
```

<a id="s10-2"></a>

## 10.2 On-ledger discovery of read nodes

> [!IMPORTANT]
> - **Is it possible?** Yes.
> - **Difficulty:** Medium.
> - **Effort and timeline:** ~3-5 person-months in clients (estimate, including failover and resubmission).

**Proposed format.** An `api_endpoints` metadata entry on the validator component, which the owner can write with a normal transaction. Its value is a JSON string:

```json
{
  "v": 1,
  "endpoints": [
    { "url": "https://read.example-validator.org", "api": "radix-read-v1",
      "features": ["state", "submit", "history", "holders", "nf-location", "explorer"],
      "network": "mainnet" }
  ]
}
```

**Client algorithm:**

1. **Bootstrap list.** Ship 5-10 access points from different operators in the binary, only for first contact.
2. **Read the full list.** From any of them, read the active validator set (`/state/consensus-manager`) and each validator's `api_endpoints` metadata.
3. **Check.** Check each access point:
   - that the network matches (`status/network-configuration`);
   - that its `state_version` is close to the majority (not lagging);
   - its latency.
4. **Use.** Pick the best 2-3 and spread requests. If one fails, move to the next. Optionally, compare `state_version` and the accumulator hash between two to detect a node that returns incorrect data.
5. **Cache.** Store the list and refresh it every few hours.

<a id="s10-3"></a>

## 10.3 `babylon-node` changes to serve reads (N1, N5b, N6)

**N1. Subintent status.** A new handler in `core-rust/core-api-server/src/core_api/handlers/`:

```rust
pub(crate) async fn handle_transaction_subintent_status(
    state: State<CoreApiState>,
    Json(request): Json<models::TransactionSubintentStatusRequest>,
) -> Result<Json<models::TransactionSubintentStatusResponse>, ResponseError> {
    let hash = extract_subintent_hash(&request.subintent_hash)?;
    let db = state.state_manager.database.snapshot();
    // The index already exists (finalized_subintent_hashes column family, column_families.rs:153-163);
    // an `impl TransactionIndex<&SubintentHash>` must be added to query it.
    let committed_at = db.get_txn_state_version_by_identifier(&hash);
    let identifiers = committed_at.and_then(|v| db.get_committed_transaction_identifiers(v));
    Ok(Json(to_api_subintent_status(committed_at, identifiers)?))
}
```

It also needs:

- the route in `core_api/server.rs`;
- the schema in `core-api-schema.yaml`, and regenerated models;
- tests.

**N5b. `last_updated_at_state_version` for metadata.** To avoid a new index, it is worth studying whether the version of the entry's last change can be obtained from the state tree's leaf node, whose database keys include the version at which it was written (to be confirmed in the `radix-substate-store-impls` code). If that is not viable, a small index is added.

**N6. Public read mode.**

- A new `api.public.*` configuration section.
- On the axum router of the Core API and the Engine State API:
  - a CORS layer;
  - a per-IP limiter;
  - a route allow-list: reads, preview, submit and status, **never** `/mempool/*` or unbounded routes;
  - force `enable_unbounded_endpoints=false`.

<a id="s10-4"></a>

## 10.4 Client changes

**Sargon: `LedgerBackend` interface and resubmission** (what to change: section [7.5](07-per-repository-changes.md#s7-5)).

1. Define the trait in a new crate, `crates/gateway/backend`, with the high-level methods used in `methods/state_methods.rs` and `methods/transaction_methods.rs`.
2. Implement it for the current `GatewayClient`, without changing its behaviour.
3. Change `SargonOS::gateway_client()` (`system/os/os/src/sargon_os_gateway_client.rs:8-17`) to return `Arc<dyn LedgerBackend>`, and the five places that build their own client to receive it.
4. **Access-point list:**
   - `SavedGateways` uses `current` + `other` as a list with failover;
   - add the discovery from 10.2 behind an option.
5. **Resubmission:**
   - store the notarised payload in the host's secure storage (the driver already exists) until it is committed or its epoch expires;
   - the loop in `sargon_os_transaction_status.rs:38-105` resubmits when the status shows the transaction is unknown or its fate uncertain (`Unknown` or `CommitPendingOutcomeUnknown` with the Gateway; `NotSeen` or `FateUncertain` with the node's LTS).
6. **Tests:** a mock of the trait for unit tests, and integration tests against an indexer on Stokenet.

**Android: fix the lost port and path** (`di/HttpModule.kt:128-144`):

```kotlin
val target = currentGatewayUrl.toHttpUrl()
val newUrl = original.url.newBuilder()
    .scheme(target.scheme).host(target.host).port(target.port)
    .encodedPath(target.encodedPath.trimEnd('/') + original.url.encodedPath)
    .build()
```

**Radix dApp Toolkit:** a list of access points with failover:

```ts
const gatewayApiService = FailoverGatewayApiService({
  endpoints: ['https://read.a.org', 'https://read.b.org'],   // or discovered on ledger
  networkId,
})
const rdt = RadixDappToolkit({ /* ... */, providers: { gatewayModule: GatewayModule({ clientConfig, providers: { gatewayApiService } }) } })
```

**ROLA:** an injectable `owner_keys` reader:

```ts
const rola = Rola({
  networkId, applicationName, expectedOrigin, dAppDefinitionAddress,
  ownerKeysReader: (address) => readMetadataEntry(endpoints, address, 'owner_keys'), // Engine State or indexer
})
```

<a id="s10-5"></a>

## 10.5 Verifiable Olympia archive

> [!IMPORTANT]
> - **Is it possible?** Yes, if a copy of the Olympia database is obtained or while its Gateway is still running.
> - **Difficulty:** Low.
> - **Effort and timeline:** weeks (estimate). Urgent.

1. **Get the data.** Ask the Foundation, or any operator that still has one, for a copy of the database of an Olympia node or of its Gateway (`olympia-gateway.radixdlt.com`, version 337,759,839). Alternatively, walk its per-account transaction endpoints for every account while it is still running.
2. **Export.**
   - Transactions in order, in JSON Lines or Parquet files chunked by version range.
   - A **static index by address**: `rdx1…` → list of chunks and positions.
   - A file mapping each Olympia address to its Babylon account.
3. **Seal.**
   - Compute a Merkle tree over the files and publish the root.
   - Have **several independent people** sign the root and publish it, for example on RadixTalk and in a Babylon transaction with a message.
4. **Publish** on IPFS (with several independent pinning services), BitTorrent and, if possible, Arweave.
5. **Integrate** into the reference explorer and `olympia-wallet`: read the static index and download only the chunks needed.

<a id="s10-6"></a>

## 10.6 Verifiable reads: the long-term solution

**Node.** A `POST /core/state/proof` endpoint that takes a list of substate keys and returns:

- the values;
- the state tree's **Merkle multiproof** against `state_tree_hash` at version V;
- the **signed ledger proof** for V.

The state tree window already keeps recent versions (`state_hash_tree.*`). For transactions and receipts, proofs are against `transaction_tree_hash` and `receipt_tree_hash`; this requires keeping the accumulators that R5 prunes, so that information would live on archive nodes.

**Light client** (a Rust crate, also compiled to WebAssembly):

1. It starts from a **checkpoint**: the proof of a recent epoch shipped with the app release.
2. **It follows epoch changes:** it downloads epoch proofs from the checkpoint onwards (`/stream/proofs`) and checks that each is signed by more than 2/3 of the previous set's stake. That way it knows the current validators.
3. **It verifies every read:**
   - the ledger proof is signed by more than 2/3 of the epoch's stake;
   - the Merkle multiproof matches `state_tree_hash`.
4. If anything fails, it drops the node and asks another ([10.2](#s10-2)).

**Verifiable indexes (protocol update).** For history and holder lists to be provably complete too, the engine would have to keep a per-entity root in state (for example, an accumulator of the transactions that affect it). This needs a protocol update and should be designed together with R10.

<a id="s10-7"></a>

## 10.7 How to test and roll out each phase without downtime

1. **Stokenet first.** Every change goes to Stokenet first, which the community has run since June 2026 according to radix.wiki. According to the same source, most of its nodes are hosted by a single operator, so nodes from other operators should be added for testing.
2. **Metrics.** Per-column-family size metrics (`raw_db_size`), process RSS, missed rounds per validator, API latency and indexer lag are published on a shared dashboard.
3. **Options off by default.** Storage changes ship as options. They are enabled on full nodes first, then validators.
4. **Rollback.**
   - For the indexer, point apps at another access point.
   - For node options, take a checkpoint before enabling those that delete data (R3, R4, R5, R7, R8).
5. **Protocol updates** (R10, verifiable indexes) follow the usual validator readiness-signalling process.

<a id="s10-8"></a>

## 10.8 History exporter: the first experiment

> [!IMPORTANT]
> - **Is it possible?** Yes, without changing the node.
> - **Difficulty:** Low-medium.
> - **Effort and timeline:** ~1-2 person-months; weeks (estimate).

**What it sets out to show:** the property the whole design in section [6.5](06-recommended-solution.md#s6-5) depends on. **Two independent exporters, next to two different nodes, produce byte-identical segments.** This test was proposed by Daffy, who contributed the design. Babylon has a single shard, so the test isolates the export itself.

**Steps:**

1. **Read from the node:** `/core/stream/transactions` with `transaction_format_options.raw_ledger_transaction=true` (raw transactions) and `/core/stream/proofs` (signed proofs, including epoch changes).
2. **Fix the boundaries:** a segment covers a fixed number of epochs, decided only by the epoch number. For example, 12 epochs are about an hour. The node's clock is never used.
3. **Encode deterministically:**
   - a header with the network, the epochs and state versions covered and the hash of the previous segment (a hash chain);
   - the raw transactions, in order;
   - the end-of-epoch header (state, transaction and receipt roots, and version);
   - the signatures, in a separate file that is not part of the compared hash, because each node may keep a different subset of signatures on the same proof (`core-api-schema.yaml:844-849`);
   - a binary encoding with explicit lengths (for example SBOR), with no local timestamps or optional fields that could differ between nodes.
4. **Verify** each segment before uploading it:
   - each proof's signatures add up to more than 2/3 of the stake of that epoch's validator set;
   - each epoch's transaction tree, recomputed from the previous root, matches the signed `transaction_tree_hash`.
5. **Upload** to S3-compatible storage, named by the segment's hash. Uploading is idempotent: if two exporters upload the same thing, the result is the same.
6. **Run two exporters** next to two Stokenet nodes, ideally run by different operators (today most of Stokenet is hosted by a single operator, according to radix.wiki), for several weeks, and compare the hash of every segment. Repeat with nodes on different versions.
7. **Receipt prototype:** a service that downloads each segment, verifies it and publishes a signed receipt (hash and epochs). This tests the export-then-prune rule without touching the node.
8. **Metrics:** last exported epoch, last receipted epoch, lag and pending bytes.

**Success criterion:**

- 100 % of segments are identical across exporters;
- verification passes;
- the measured size per epoch is published.

**Afterwards:** add receipts in canonical form once N10 exists, move the exporter into the node and connect the receipt rule to R8's pruning.

---

[← Index](../../README.md) · [Previous: Chapter 9: lighter nodes (RAM and disk) and the Java to Rust migration](09-lighter-nodes.md) · [Next: Chapter 11: final stage B, migration to Hyperscale](11-hyperscale-evolution.md)
