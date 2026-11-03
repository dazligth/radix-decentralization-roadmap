[← Index](../../README.md) · [Next: Chapter 3: what the node offers today and what it lacks](03-what-the-node-offers.md)

<a id="scope"></a>

# Scope

**Goal.** The Radix Babylon network should serve **any user, in any use case**, without depending on the Babylon Network Gateway: the REST + DataAggregator + PostgreSQL service published today at `mainnet.radixdlt.com`. The use cases are:

- wallets;
- dApps;
- ROLA login;
- transaction explorers;
- governance;
- agents;
- developer tools.

**Design principle: everything on Layer 1.** Radix's goal is to scale and deliver all its features on Layer 1, without Layer-2 solutions (rollups, sidechains or partner chains). Every proposal in this report respects that principle:

- The light indexer, read nodes and access-point discovery are infrastructure for **reading** Layer-1 state. They do not execute transactions or keep any state of their own that the network has to trust.
- Pruning, verified snapshots, the migration to Rust and verifiable reads are changes to Layer-1 node software.
- Incentives for serving reads would be paid on Layer 1 itself (for example, through an on-ledger component), as governance decides.
- Hyperscale scales **on Layer 1 itself**, through sharding with atomic composability, which is consistent with that goal.
- The bridge in strategy M2 (chapter [11](11-hyperscale-evolution.md#s11)) would only connect two Layer-1 networks during the transition, and it is optional.

References to Layer-2 or partner-chain mechanisms in other networks (chapter [12](12-references-other-networks.md#s12)) are included only as a contrast, not as a model.

**Repositories analysed.**

- **Local:**
  - `babylon-wallet-android` and its forks;
  - `radix-dapp-toolkit`;
  - `connector-extension` and `connector-extension-iroh`;
  - `radix-connect-iroh` and `radix-connect-webrtc`;
  - `radixdlt-rust-sdk`;
  - `consultation_v2` and `consultation_v3`;
  - `governance-framework`;
  - `radixdlt-scrypto` (`develop` branch, commit `a405d7ae48`, Jan 2026): engine and transactions, for chapter [13](13-xian-with-scrypto.md#s13);
  - `wallet-agent-ai`;
  - `Radix-Desktop-Tool`;
  - `olympia-wallet`;
  - `wallet-compatible-derivation`;
  - `reference/hyperscale-rs`.
- **Cloned from GitHub** (not present on this machine):
  - `radixdlt/babylon-node`;
  - `radixdlt/babylon-gateway`;
  - `radixdlt/sargon`;
  - `radixdlt/babylon-wallet-ios`;
  - `radixdlt/rola`;
  - `radixdlt/babylon-nodecli`;
  - `radixdlt/explorer`, the Olympia explorer (see [2.2](#s2-2)).
- **Not public:** the source of the Radix Dashboard and the Developer Console is not in the `radixdlt` GitHub organisation. They are treated as black-box clients.

## Current infrastructure

```mermaid
flowchart TB
    subgraph APPS["Users and applications"]
        W["iOS and Android wallets (Sargon)<br/>and Connector Extension"]
        DAPP["dApps with RDT and backends<br/>with ROLA login"]
        EXP["Radix Dashboard, explorers<br/>and Developer Console"]
        OLYW["olympia-wallet"]
    end
    subgraph CENTRAL["Central services: a single operator"]
        AUX["Wallet services<br/>WebRTC signalling, TURN,<br/>Radix Connect Relay, prices,<br/>images, dApp directory"]
        GW["<b>Babylon Gateway</b><br/>Gateway API + DataAggregator<br/>(single instance)<br/>PostgreSQL ~2.5 TB, +60 GB/month"]
        OGW["Olympia Gateway<br/>2021-2023 history"]
    end
    subgraph NET["Radix network"]
        FN[("Full nodes<br/>Java + Rust, 16 GB RAM")]
        VAL[("~100 validators<br/>Java + Rust, 16 GB RAM")]
    end
    W -.-> AUX
    DAPP -.-> AUX
    W --> GW
    DAPP --> GW
    EXP --> GW
    OLYW --> OGW
    GW <-->|"transaction stream,<br/>preview and submit"| FN
    FN --- VAL
    classDef central fill:#ffd6d6,stroke:#c00,color:#000
    class GW,AUX,OGW central
```

In red, services that depend on a single operator.

<a id="s1"></a>

# 1. Current situation: data

<a id="s1-1"></a>

## 1.1 The Gateway and its funding

| Data point | Value | Source |
|---|---|---|
| Gateway database size (Feb 2026) | **~2 TB, growing ~60 GB/month** | Foundation RFP, 3 Feb 2026 |
| Estimated size today (Nov 2026) | ~2.5 TB (2 TB + 9 months × 60 GB) | calculation |
| RFP requirements | 99.9 % uptime, <1 s latency, blue/green deployment | RFP |
| DataAggregator | **Single instance**: transactions are processed sequentially | Foundation, radixtalk 2202 posts 15 and 22 |
| Current operator | Former Foundation DevOps team, as a standalone operation | Blog, 28 Apr 2026 |
| Funding | **Until end of December 2026** | Blog, 28 Apr 2026 |
| Cost for a third party | ~€300-400/month for the Gateway alone; ~€500/month including relay and signalling | Community proposal, radixtalk 2202 post 26 |
| Estimated load | ~250,000 requests/hour (~69 req/s) | Community estimate, radixtalk 2202 post 29 |
| Xi'an / Hyperscale | "would require building a completely new gateway"; the current one is not reusable | Foundation, radixtalk 2202 post 15 |
| WebRTC signalling | ~6,000 concurrent peak connections | RFP |

<a id="s1-2"></a>

## 1.2 Mainnet measurements for this report (3 Nov 2026, via the public API)

| Measure | Value |
|---|---|
| Current `state_version` | 567,005,616 (epoch 355,117) |
| State-version rate | **~3.6/s on average** since June 2026 (4.4/s in today's sample). Each consensus round is one version. |
| User transactions per day (100-transaction sample at each date) | Nov 2023: 1.5k · Mar 2024: 5.5k · Jul 2024: 11.6k · Nov 2024: 20.7k · **Mar 2025: 34.7k (peak)** · Jul 2025: 8.4k · Nov 2025: 5.9k · Mar 2026: 3.4k · Jul 2026: 2.2k · Oct 2026: 4.4k |
| Estimated user transactions since genesis | **~10-15 million** (rough integration, ±50 %) |
| Share of user transactions | **~2-3 % of state versions** over the whole history (~10-15 million of ~567 million); today, under 1 %. The rest (~97-98 %) are round and epoch changes. |
| Node historical window | `state_version_history_length = 60000` versions by default, **≈ 4-5 hours** at the average of 3.6 versions/s (≈ 3.8 h at today's measured rate) |

**Key consequence.** Most of the Gateway's volume probably comes from storing the full history of every substate, including ~555 million round updates. (This is an inference: the composition of its database has not been published.) An index that records only what clients need deals with ~10-15 million user transactions. That is a problem **about 40-55 times smaller**.

<a id="s1-3"></a>

## 1.3 What real queries to the Gateway show (3 Nov 2026)

To better understand what the Gateway does and what would have to be replaced, its public API was queried:

| Query | Result | What it implies |
|---|---|---|
| State of the XRD resource at version 10,000 (`at_ledger_state`, 28 Sep 2023) | It answers: total supply of 12,660,434,555 XRD then, versus 13,565,348,962 today | The Gateway serves **full history from genesis**. The node only keeps a few hours (section [3.1](03-what-the-node-offers.md#s3-1)). That arbitrary history is what takes most space and what the light indexer does not replicate; whoever needs it would run an archive node (sections [6.2](06-recommended-solution.md#s6-2) and [9.6](09-lighter-nodes.md#s9-6)). |
| XRD holders (`/extensions/resource-holders/page`) | `total_count`: **435,605** | It gives the order of magnitude of the holder index (h). At ~100 bytes per row (an assumption), XRD would take about 44 MB: a tiny fraction of the light indexer's estimated total (10-50 GB). |
| Response headers | `server: cloudflare`, `cf-cache-status: DYNAMIC` | The public Gateway sits behind Cloudflare and its responses are not served from cache. It is another central dependency; read nodes need their own protection against attacks (N6). |
| `/status/gateway-status` and `/status/network-configuration` | Version 1.10.6 (image v1.10.7); network `mainnet`, id 1 and well-known addresses | The light indexer can obtain the same data from the node (`/status/network-configuration`). |

**Conclusion:** the Gateway's functions fall into three groups, and all three have a replacement:

1. current state → the node itself;
2. indexes (per-entity history, holders, NFT location, search) → the light indexer;
3. arbitrary historical state → archive nodes, only for those who need it.

That is why the Gateway **can be removed completely**.

<a id="s2"></a>

# 2. Who depends on the Gateway, and for what

| Client | Gateway endpoints | Purpose |
|---|---|---|
| **Sargon** (Rust core shared by iOS and Android) | `transaction/{construction, preview, preview-v2, submit, status, subintent-status}`, `status/gateway-status`, `state/entity/details`, `state/entity/page/{fungibles, non-fungibles, non-fungible-vaults, non-fungible-vault/ids}`, `state/account/page/{resource-preferences, authorized-depositors}`, `state/non-fungible/{location, data}`, `extensions/entities-by-role-requirement/lookup` | Transaction review and fees, submit and track, pre-authorisations, securified or deleted accounts, MFA and shields, account deletion, RNS, NFT pricing |
| **Android wallet** (Kotlin, on top of Sargon) | `state/entity/details`, `state/entity/page/{fungibles, non-fungibles, non-fungible-vault/ids, metadata}`, `state/non-fungible/data`, `state/account-lockers/touched-at`, `state/account-locker/page/vaults`, `transaction/{construction, account-deposit-pre-validation}`, `stream/transactions`, `status/gateway-status` | Balances, NFTs, pools, validators, LSUs, stake claims, dApp metadata, account recovery, lockers, deposit pre-validation, **history** |
| **iOS wallet** (Swift, on top of Sargon) | Same, plus `state/entity/page/fungible-vaults` | Same |
| **Radix dApp Toolkit** | `transaction/status`, `transaction/subintent-status`, `state/entity/page/metadata` | Track the transaction the wallet submits; dApp name on the connect button |
| **ROLA** (`radixdlt/rola`) | `state/entity/details` (`owner_keys` metadata) | **Sign-in for every dApp.** It is queried on every sign-in (`typescript/src/rola.ts:49-87`, `gateway.ts:14-38`) and uses the public URL by default. |
| **Connector Extension** | `transaction/status` (via the RDT's `GatewayModule`) | "Transaction committed" notification |
| **Transaction explorers** (Radix Dashboard, closed source; community explorers) | Cannot be verified for the Dashboard; from its features, practically the whole API, including the **resource holders** extension, NFT location, filtered history of **any** entity, validator uptime, KV stores, packages, schemas and `at_ledger_state` | See section [2.1](#s2-1) |
| **Developer Console** (closed source) | Cannot be verified; from its features, at least preview, submit, status and entity details | Package deployment, manifests |
| **Consultation** (governance) | `state/entity/details`, `state/key-value-store/{keys, data}`, `stream/transactions`, `transaction/committed-details`, `status/gateway-status`, balances and NFTs **with historical `at_ledger_state`** | Proposals, votes, snapshot tally |
| **radixdlt-rust-sdk** | `status/gateway-status`, `state/entity/page/fungibles`, `transaction/{status, committed-details, submit, construction, preview}`, `state/entity/details` | Generic signing and submission SDK |
| **wallet-agent-ai** | ~25 calls: entity details, preview, submit, status, stream, validators, uptime | Agent with its own wallet |
| **Exchanges and integrators** | According to the Core API specification, they are recommended to use the node's own `/lts/` endpoints (`core-api-schema.yaml:83-89`, v1.4.0.0) | Deposits, withdrawals and transaction tracking. Those who follow that recommendation **do not depend on the Gateway**. Those using the Gateway or the Mesh API (removed in v1.4.0.0) must move to the LTS. |
| **Radix-Desktop-Tool** | The whole API, via `@atlantis-l/radix-tool` | Desktop tooling, history |

<a id="s2-1"></a>

## 2.1 Transaction explorers: an explicit requirement

> [!IMPORTANT]
> - **Is it possible?** Yes. Explorers can work without the Gateway.
> - **Difficulty:** Medium.
> - **Effort and timeline:** ~1-2 person-months of indexes + ~2-3 for the reference explorer (estimate).

An explorer is the **most demanding** client: it needs global indexes that neither the node nor the wallets need. Each explorer feature has a different source: the node itself or a new index.

| Explorer feature | How it is served without the Gateway |
|---|---|
| Look up a transaction by intent hash: receipt, events, state changes, fees | Yes, in the node: `/core/transaction/receipt` (by `intent_hash`) and `/core/lts/transaction/status`. Balance changes are computed from the receipt. |
| Latest transactions; transactions by epoch or round | Yes, in the node: `/core/stream/transactions` (ascending). Descending order or "from a date" needs a `timestamp → state_version` index. |
| Account, component or resource page: current state, balances, NFTs, metadata, roles, royalties | Yes, in the node: Core and Engine State APIs |
| Packages: blueprints, code, schemas | Yes, in the node: `/core/state/package`, `/blueprint/info`, `/entity/schema/entry` |
| Component KV stores | Yes, in the node: `/kv-store/iterator` |
| **Transaction history of any entity**, filtered by type, resource or event | No: E3 index (a), or N3 |
| **Holders of a resource**, ranked by balance | No: new index (h): `resource → (entity, balance)` |
| **Where an NFT is** | No: E3 index (b), or N2 |
| Validators: uptime, stake history, APY | No: E3 index (g), or N9. Current stake comes from the node. |
| Account or resource state at a past date | No: archive node (N5), or balance-change index (a) + (h) |
| Statistics: TPS, transactions per day, active accounts, fees burned | No: new index (i): hourly and daily aggregates |
| Search by name or symbol (metadata) | No: new index (j): text of `name`, `symbol`, `tags` → entity |
| Search by address | Yes: the address encodes its type; `/entity/info` is enough |

**Consequences for the strategy:**

- Explorers **rule out E1**. With the node alone, history, holders and statistics pages are lost.
- **E3 must include indexes (h), (i) and (j).** They remain small:
  - (h) scales with the number of resource-holder pairs, not with state versions;
  - (i) is a few rows per hour;
  - (j) covers only entities that have `name` or `symbol` metadata.
- **The official explorer (Radix Dashboard) is not open source.** For the network to have an explorer that depends on nobody, an **open-source reference explorer** is also needed. It would run against `radix-lite-indexer` or any access point discovered on ledger (section [7.16](07-per-repository-changes.md#s7-16)).

<a id="s2-2"></a>

## 2.2 The Olympia legacy

> [!IMPORTANT]
> - **Is it possible?** Yes. Olympia does not prevent removing the Gateway. Archiving its history is also possible.
> - **Difficulty:** Low.
> - **Effort and timeline:** weeks for the archive (estimate); it is urgent, because it must be done before the Olympia Gateway is switched off.

```mermaid
flowchart LR
    O[("Olympia ledger<br/>frozen on 28 Sep 2023")] -- "migrated state:<br/>balances, stakes, tokens" --> G["Babylon genesis<br/>versions 1 to 647"]
    G --> B[("Babylon ledger<br/>kept by every node")]
    O -- "transaction history<br/>(not in Babylon)" --> OG["Olympia Gateway<br/>single service"]
    O -. "proposal: export once" .-> AR["Verifiable archive<br/>IPFS, BitTorrent or Arweave"]
    classDef risk fill:#ffd6d6,stroke:#c00,color:#000
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class OG risk
    class AR good
```

Radix had an earlier network, **Olympia**, which was migrated to Babylon on 28 Sep 2023. This was checked against the live ledger and the code: it **does not block the solution**, but it adds four considerations, three of which require work (items 1, 3 and 4).

1. **Olympia's state is inside the Babylon genesis.**
   - Mainnet state versions 1 to 647 are genesis transactions (648-669 are already round updates). The first user transaction is version 670 (2023-09-28 07:26 UTC).
   - Those transactions load, in chunks (`GenesisDataChunk`: `Validators`, `Stakes`, `Resources`, `ResourceBalances`, `XrdBalances`; `radix-engine/.../genesis_helper/src/lib.rs:34-46`), the validators, stakes, tokens and balances that existed on Olympia. Accounts are derived from the Olympia secp256k1 keys.
   - **Consequence for the indexer (E3/E2):**
     - the first sync must start at version 1 and process genesis, whose receipts are very large;
     - otherwise every migrated account that has not moved since 2023 would be missing from the indexes: XRD holders, the account's first movement, migrated stakes.
   - This is a one-off job and can be included in the signed snapshots.
2. **Olympia accounts keep working without the Gateway.**
   - Importing Olympia accounts into the Babylon wallet is **offline**: export with `olympia-wallet`, and `MigrateOlympiaAccountsUseCase` in the Android wallet.
   - ROLA accepts address derivation for accounts without `owner_keys`, including Olympia's secp256k1 ones, although it always queries the ledger to check this (section [7.9](07-per-repository-changes.md#s7-9)).
   - The **recovery scan** with an Olympia seed phrase (BIP44 path, `ResolveDerivationPathsForRecoveryScanUseCase.kt:42-79`) uses the same ledger queries as everything else. It works against a node: genesis-created accounts exist and are detected with `/entity/info`, even without `last_updated_at_state_version`.
3. **Olympia transaction history (before 28 Sep 2023) is not in the Babylon ledger.**
   - It is served only by the **Olympia Gateway**, `olympia-gateway.radixdlt.com`. It is still live today, running Gateway API 1.1.14, frozen at version 337,759,839 from 2023-09-28T06:33:34Z.
   - It is used by `olympia-wallet` (`src/helpers/network.ts:14-39`) and the Olympia explorer (`radixdlt/explorer`).
   - It is **another central service**, and it does not appear in the transition service lists published by the Foundation that were reviewed.
   - If it is switched off, Olympia's public history (audits, taxes, research) **is lost**, unless someone keeps a copy of an Olympia node or Gateway database.
   - **Advantage:** it is frozen and no longer changes, so it needs no indexer. A **single, verifiable archive** is enough:
     - export the Olympia ledger (node database or a dump of the Olympia Gateway);
     - publish it with its hash on distributed storage (IPFS, BitTorrent or Arweave);
     - add a static index by `rdx1…` address that any explorer can serve as plain files.
   - **Priority P0, because it is urgent:** this must be done before the Foundation's infrastructure is switched off.
4. **Explorers must understand both eras.**
   - An Olympia address (`rdx1…`) maps deterministically to its Babylon account (`account_rdx…`), because both come from the same key. Sargon already does this conversion, so no index is needed.
   - The reference explorer ([7.16](07-per-repository-changes.md#s7-16)) should accept both addresses, showing Olympia history from the static archive and Babylon history from the nodes.

---

[← Index](../../README.md) · [Next: Chapter 3: what the node offers today and what it lacks](03-what-the-node-offers.md)
