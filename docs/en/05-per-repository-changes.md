[← Index](../../README.md) · [Previous: Section 6: recommended solution, storage, and the ideal long-term solution](04-recommended-solution.md) · [Next: Section 8: roadmap](06-roadmap.md)

# 7. Per-repository changes

Priority: **P0** essential · **P1** needed for every use case · **P2** improvement.
Effort: **S** days · **M** 1-3 weeks · **L** more than a month.

## 7.1 New repository: `radix-lite-indexer` (strategy E3)

**Components:**

- **Ingestion (P0/M):**
  - `/core/stream/transactions` with receipts and events;
  - persistent cursor that survives restarts;
  - round changes are filtered out of the entity indexes.
- **Indexes (P0/M):**

  | Index | Purpose |
  |---|---|
  | (a) `entity → [state_version]` with fungible and NFT changes, manifest class, event emitters and manifest resources | History |
  | (b) `(resource, nf_id) → vault → global entity` | NFT location |
  | (c) `subintent_hash → state_version` | Pre-authorizations (or read it from the node once N1 exists) |
  | (d) `badge → entities` | Role rules, for MFA |
  | (e) `(entity, metadata key) → last_updated_state_version` | Account recovery |
  | (f) `locker → account → last touch` | Lockers |
  | (g) emissions per validator and epoch | Uptime |
  | (h) `resource → (entity, balance)` | Resource holders (explorers) |
  | (i) hourly and daily aggregates: transactions, fees, active accounts, TPS | Statistics (explorers) |
  | (j) text of `name`, `symbol` and `tags` → entity, plus `timestamp → state_version` | Search and date navigation (explorers) |

- **Read API in the apps' current format (P0/L).** Implement these routes, with the JSON that clients expect today:
  - `status/gateway-status`, `status/network-configuration`;
  - `transaction/{construction, preview, preview-v2, submit, status, subintent-status, committed-details, account-deposit-pre-validation}`;
  - `state/entity/details` (with `aggregation_level`, `opt_ins` and `explicit_metadata`);
  - `state/entity/page/{fungibles, non-fungibles, non-fungible-vaults, non-fungible-vault/ids, fungible-vaults, metadata}`;
  - `state/non-fungible/{data, location, ids}`;
  - `state/key-value-store/{keys, data}`;
  - `state/validators/list`;
  - `state/account/page/{resource-preferences, authorized-depositors}`;
  - `state/account-lockers/touched-at`, `state/account-locker/page/vaults`;
  - `stream/transactions` (with the filters the wallets use);
  - `extensions/entities-by-role-requirement/lookup`;
  - `extensions/resource-holders/page` (explorers);
  - `statistics/validators/uptime`;
  - its own statistics and search endpoints for explorers (indexes i and j).

  Current-state reads are resolved live against the node. The reference models are in `babylon-gateway/src/RadixDlt.NetworkGateway.GatewayApi`.
- **Transaction resubmission (P0/S).** Port `PendingTransactionResubmissionService` (`babylon-gateway`, `PendingTransactionResubmissionService.cs:123-307`; defaults in `MempoolOptions.cs:74-99`).
- **Consistency (P0/S).** Pin the `state_version` of each paged request, and reject the request if the node has moved past the cursor.
- **Public mode (P0/S):**
  - CORS;
  - per-IP rate limiting;
  - page limits;
  - TLS;
  - endpoint allow-list;
  - Docker image and a `babylon-nodecli` template.
- **Snapshots (P2/M).** Export and import signed indexes so a new operator does not have to sync from genesis.
- **Conformance (P0/M).** Compare responses against the public Gateway with continuous sampling, before it is switched off.

## 7.2 `radixdlt/babylon-node` (strategy E2)

| # | Change | P / Effort |
|---|---|---|
| N1 | `/core/transaction/subintent-status` (and an LTS version), reading the existing `finalized_subintent_hashes` | P0 / S |
| N6 | **Public read mode:** `api.public.*` with allow-list, CORS, rate limiting, page limits, `enable_unbounded_endpoints=false` enforced | P0 / M |
| N5b | `last_updated_at_state_version` in `/object/attached-modules/metadata/{entry, iterator}` | P1 / S |
| N2 | Optional index `db.non_fungible_location_index.enable` and `/core/state/non-fungible-location` | P1 / M |
| N3 | Change index per global entity (not just accounts), with NFT changes, manifest class and filters | P1 / L |
| N4 | Reverse role-rule index and `/core/extra/entities-by-role-requirement` | P1 / M |
| N5 | Archive node: `state_version_history_length` with no practical limit, `LedgerStateSelector::ByTimestamp` (already anticipated in `engine-state-api-schema.yaml:592`), `at_ledger_state` on the LTS | P1 / M |
| N7 | Batch endpoints (metadata and state for several entities at once) | P2 / S |
| N8-N9 | Locker touches; validator emissions | P2 / S |
| D | Validator metadata convention `api_endpoints` (list of URLs with capabilities), and operator documentation | P0 / S |

Every index should be an **optional flag**, like the existing ones, so validators who do not want to serve reads are not penalised.

## 7.3 `radixdlt/babylon-nodecli` (node deployment)

- **P0/S:** add a "public read node" profile with:
  - the Engine State API enabled (today the wizard defaults to `N`, `node-runner-cli/utils/Prompts.py:252-257`);
  - `db.historical_substate_values` and `db.entity_listing_indices` per profile;
  - the `radix-lite-indexer` sidecar;
  - a TLS proxy.
- **P1/S:** a wizard step to publish `api_endpoints` in the validator's metadata.

## 7.4 `radixdlt/babylon-gateway`

- **No changes needed for the goal.** It serves as the **specification** of the API, of the resubmission logic and of submit-error classification (`TransactionSubmitter.cs:115-230`).
- **P2:** split the OpenAPI contract into a standalone versioned package shared with `radix-lite-indexer`, to guarantee compatibility.

## 7.5 `radixdlt/sargon` (wallet core)

1. **P0/M — `LedgerBackend` trait.**
   - `GatewayClient` (`crates/gateway/client-and-api/src/client/gateway_client.rs:11-40`) is a concrete struct. Extract a trait with the methods in `methods/state_methods.rs` and `methods/transaction_methods.rs`, and return it from `SargonOS::gateway_client()` (`system/os/os/src/sargon_os_gateway_client.rs:8-17`).
   - Five places build their own client and must switch to the trait:
     - `transaction_intent_builder.rs:58`
     - `xrd_balances_fetcher.rs:46`
     - `access_controller_state_repository_client.rs:27`
     - `radix-name-service/src/service.rs:30`
     - `app/home-cards/src/manager.rs:45`
2. **P0/S — Several endpoints with failover and on-ledger discovery.**
   - `SavedGateways` (`crates/profile/models/gateway/src/saved_gateways.rs:16-24`) stores a list but only uses the current entry.
   - Add failover and a network check.
   - Add discovery (D): read `api_endpoints` from the active validators via `/state/consensus-manager` and their metadata.
3. **P0/M — Local resubmission.** Today `sargon_os_transaction_status.rs:38-105` only polls. Keep the notarized payload and resubmit with backoff until it is committed or its epoch expires. That way a transaction is not lost if the access point changes or restarts.
4. **P1/M — Native `NodeBackend`** (Core and Engine State APIs, with DTOs generated from the node's YAML specs), for when E2 exists.
   - It implies a Profile schema change: `Gateway { network, url }` (`gateway.rs:21-27`) gains `kind: GatewayApi | NodeApi`.
   - This must be coordinated with iOS and Android, with a migration.
5. **P1/S — Hard-coded services.** Make `nft-pricing-service.radixdlt.com` configurable (`non_fungibles_prices_service/remote_fetcher.rs:4-5`).

## 7.6 `babylon-wallet-android` (and the forks `-iroh`, `wallet-pr-*`)

- **P0/S.** `BaseUrlInterceptor` (`di/HttpModule.kt:128-144`) rewrites only the scheme and host, so it **drops the port and path**. An access point like `https://node:8443/gw` breaks after switching gateways.
- **P0/S.** `HomeCardsManager` is hard-coded to `Gateway.mainnet` (`di/ApplicationModule.kt:74-87`); it should use the current access point.
- **P0/S.** In Settings → Gateways (`GatewaysViewModel.kt:105-133`), show the list discovered on ledger and probe several access points.
- **P1/M.** Once Sargon has `LedgerBackend`, write the node implementation of the repositories bound in `di/DataModule.kt:48-137`:
  - `StateRepository`
  - `StreamRepository`
  - `TransactionRepository`
  - `NetworkInfoRepository`
  - `AccountLockersRepository`
  - `ResolveAccountsLedgerStateRepository`

  Paging and aggregation live in `data/gateway/extensions/StateEntityDetailsApiHelper.kt`.
- **P1/S.** Make these configurable, with a fallback:
  - `image-service.radixdlt.com` (`app/build.gradle:70,112`, `Thumbnail.kt:523`), loading the original image when it is unavailable;
  - `dapps-list.radixdlt.com` (`DAppDirectoryRepository.kt:81-87`);
  - dashboard links (`NetworkIdExtensions.kt:50-56`).

## 7.7 `radixdlt/babylon-wallet-ios`

Symmetric to Android. The DI clients (`DependencyKey`) allow the `liveValue` to be replaced:

- **P0/S:**
  - failover and discovery in `GatewaysClient` and `NetworkSwitchingClient` (`NetworkSwitchingClient+Live.swift:30`).
- **P1/M:**
  - `GatewayAPIClient` (`Clients/GatewayAPI/GatewayAPIClient/GatewayAPIClient+Interface.swift:5`, endpoints in `+Live.swift:122-217`);
  - `OnLedgerEntitiesClient`;
  - `TransactionHistoryClient`;
  - `AccountLockersClient`;
  - `TransactionClient`.
- **P1/S:**
  - `image-service` (`URLFormatterClient+Live.swift:20-22`);
  - `dapps-list` (`DAppsDirectoryClient+Live.swift:4`);
  - dashboard links (`Radix+Dashboard.swift:48-102`).

## 7.8 `radix-dapp-toolkit`

- **P0/S.** Accept **a list** of access points with failover in `gatewayBaseUrl`, or on-ledger discovery. The default URL is hard-coded in `modules/gateway/types.ts:60-97`.
- **P1/S.** A `NodeGatewayApiService` provider for the node's native API (E2). It is injected via `GatewayModule({ providers: { gatewayApiService } })` (`gateway.module.ts:16-26`) and only needs three functions:
  - `getTransactionStatus`
  - `getSubintentStatus`
  - `getEntityMetadataPage`
- **P1/S.** The Radix Connect relay is hard-coded at `modules/wallet-request/wallet-request.ts:159`; add a `relayUrl` option.
- **P2/S.** Hard-coded links (`connect-button.module.ts:305,307`, `packages/connect-button/src/config.ts:3`).

## 7.9 `radixdlt/rola` (login for every dApp)

- **P0/S.** `GatewayService` (`typescript/src/gateway.ts:14-38`) defaults to `RadixNetworkConfigById[networkId].gatewayUrl`. It should:
  - accept a list or discovery;
  - add an `ownerKeysReader` option that reads `owner_keys` via the Engine State API (`/object/attached-modules/metadata/entry`).
- **P0/S.** Same change in the `python/` package.
- **P1/S.** If the ledger cannot be queried and the account is virtual (no `owner_keys`), address derivation alone is enough (`rola.ts:85`). Document that this case needs no network.

## 7.10 `connector-extension` and `connector-extension-iroh`

- **P1/S.** `src/chrome/background/create-gateway-module.ts:1-13` uses the hard-coded URL from `RadixNetworkConfigById`. Add an access-point URL or list to the options page (`src/options/index.ts:20-23`).
- **P1/S.** With a custom signalling server, the extension still uses RDX's TURN servers with hard-coded credentials (`src/config.ts:44-53, 97-101`). Make TURN configurable.
- **P2/S.** Configurable explorer (`src/utils/explorer.ts:6`).

## 7.11 Radix Connect transport: `radix-connect-iroh`, `radix-connect-webrtc`, and the official `radix-connect-relay` and `signaling-server`

They do not use the Gateway, but they are P1 infrastructure that also depends on a single operator:

- **`radix-connect-iroh`:** already supports a custom relay (`Relays::Custom`, `crates/radix-connect-iroh/src/config.rs:15-18`). P2: several custom relays and a custom pkarr.
- **`radix-connect-webrtc`:** P2/S. Allow **replacing** Google's STUN servers (`src/connector/connector-client.ts:115-131`), not just adding others.
- **`radix-connect-relay` and `signaling-server`:** P1/S. Publish images and self-hosting guides, and announce instances through the same discovery mechanism (D).

## 7.12 `radixdlt-rust-sdk`

- **P0/M.** Create a lightweight `ledger-api` crate with a `LedgerBackend` trait and two implementations: Gateway API and node API.
- **P0/S.** Several changes in `gateway-tx`:
  - `Gateway` (`crates/gateway-tx/src/lib.rs:145`) implements the trait.
  - **Bug:** `fungible_balance` (`:213-228`) reads only the first page, so a balance on page 2 shows as 0.
  - The hard-coded URLs (`:38-40`) become a list with discovery.
- **P1/S.** `connect-iroh`: `Wallet.gateway` (`crates/connect-iroh/src/protocol.rs:249`) becomes `Box<dyn LedgerBackend>`.
- **P1/S.** `connector-mcp/src/gateway.rs:10-15` has hard-coded URLs.
- **P1/S.** `rola`: implement the `owner_keys` read (`crates/rola/src/lib.rs:8-9`).
- **P1/S, security.** `connect-webrtc/src/connector.rs:56-70` has TURN credentials in source code.

## 7.13 `consultation_v3` (identical to `consultation_v2`)

- **P0/S.** Make the access point configurable:
  - `packages/shared/src/gateway.ts:10-42`;
  - `envVars.ts:8-15`, which has no `GATEWAY_URL`, so the frontend always uses the public Gateway;
  - the RDT in `apps/consultation/src/lib/dappToolkit.ts:46-50`.
- **P0/M.** **Tally without arbitrary history (E6):**
  - Handle `TemperatureCheckCreatedEvent` and `ProposalCreatedEvent`, which are ignored today (`governanceEvents.ts:66-69`).
  - At that moment, store the full snapshot in the database: XRD, LSU, LSULP and pool-unit balances of participants, redemption rates and pool state.
  - For fungibles, the "reverse delta" also works: current balance minus the `fungible_entity_balance_changes` from `/lts/stream/account-transaction-outcomes`.
  - Today `voteCalculation.ts:178-216` and `scripts/tally.ts` depend on old `at_ledger_state`.
- **P1/S, existing bug.** `accountBalanceState.ts:110-124` reads the validator list at current state, not at the snapshot.
- **P1/M.** With the node's native API, `poll.ts:30-45` loses `affected_global_entities_filter`: it must read the whole stream and filter by emitter.
- **P2.** In the blueprint (`scrypto/`), emit the snapshot parameters in `Created` and consider on-ledger tallies. This affects the `governance-framework` rules on "snapshot at opening" and reruns.

## 7.14 `wallet-agent-ai`

- **P0/S.** Hard-coded URLs in `src/network-config.ts:14-22`; the network is inferred from the URL (`:33-36`, `weft.ts:211-225`).
- **P0/S, existing bug.** It checks for `"Rejected"` / `"PermanentlyRejected"` (`src/core/RadixClient.ts:122`).
- **P1/M.** Create a `LedgerReader` interface for its ~25 `fetch` calls.
- **P1/S.** `waitForOwnerApproval` (`AgentWallet.ts:697-722`) should watch component state instead of filtering the stream.
- **P2.** The Astrolescent API should accept a self-hosted instance.

## 7.15 `Radix-Desktop-Tool`

- The URL can already be changed in the UI (`src/views/LocalSettings.vue`), so it works with E3 unchanged.
- **P2/M.** The node's native API needs a fork of `@atlantis-l/radix-tool`.
- **P1.** History (`HistoryCheck.vue:999`) depends on E3 index (a), or N3.

## 7.16 Transaction explorers: Radix Dashboard, community explorers, and a new reference explorer

- **Radix Dashboard (closed source).** It uses the Gateway API, so it works with E3 if its API URL is configured and E3 includes indexes (h), (i) and (j). This must be confirmed with whoever operates it. **P0:** ask for its source to be published, or for a community deployment.
- **Community explorers.** I could not verify their backends. If they use the Gateway API, E3 serves them by changing the URL. If they run their own indexer, they no longer depend on the Gateway.
- **P0/L — New repository `radix-explorer` (open-source reference explorer, static site).** No single entity then controls the only explorer.
  - **Data source:** the read API of `radix-lite-indexer` and of nodes, with the access-point list discovered on ledger (validator `api_endpoints` metadata) and failover.
  - **Minimum pages:**
    - transaction (receipt, manifest, events, balance changes, fee);
    - account, component, resource and package;
    - NFT, with its location;
    - resource holders;
    - validators;
    - latest transactions;
    - statistics;
    - search.
  - **Deployment:** a static site that can be published on any hosting or on IPFS, with no server of its own.
  - **Verification:** show which node each piece of data came from and, optionally, compare two access points (`state_version` and accumulator hash).
- **Developer Console (closed source).** It only needs preview, submit, status and entity details, so it works with E3 or the node. **P1:** publish its source, or provide a community replacement.

## 7.17 Repositories that need no changes

- **`olympia-wallet`:** its current use (exporting accounts) is offline. **P2/S:** once the Olympia archive exists (2.2), point to it instead of `olympia-gateway.radixdlt.com` (`src/helpers/network.ts:14-39`).
- **`radixdlt/explorer`** (the Olympia explorer): **P1/S:** serve history from the static Olympia archive instead of the Olympia Gateway, or fold it into the reference explorer (7.16).
- **`wallet-compatible-derivation`:** offline key derivation, no network access.
- **`governance-framework`:** only the points in 7.13.
- **`reference/hyperscale-rs`:**
  - A different network, not compatible with Babylon.
  - Its RPC (`crates/production/src/rpc/routes.rs:13-35`) has no state reads.
  - According to the Foundation, Xi'an needs a new access layer anyway. The clients' `LedgerBackend` interface is where that third implementation would go.

---

[← Index](../../README.md) · [Previous: Section 6: recommended solution, storage, and the ideal long-term solution](04-recommended-solution.md) · [Next: Section 8: roadmap](06-roadmap.md)
