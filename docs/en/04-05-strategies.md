[← Index](../../README.md) · [Previous: Chapter 3: what the node offers today and what it lacks](03-what-the-node-offers.md) · [Next: Chapter 6: recommended solution, storage, ideal solution and risks](06-recommended-solution.md)

<a id="s4"></a>

# 4. Strategies evaluated

## E1. Clients talk directly to the Core and Engine State APIs, with no node changes

> [!IMPORTANT]
> - **Is it possible?** Only partly: it fully covers 9 of the 19 capabilities in table 3.2 and 2 partly (the most common ones), but not the rest.
> - **Difficulty:** Medium-high.
> - **Effort and timeline:** 15-25 person-months in clients (estimate).

- **Coverage:** partial (9 of 19 capabilities fully and 2 partly, section [3.2](03-what-the-node-offers.md#s3-2)). It loses N1-N5: pre-authorisation status (N1), NFT location (N2), full history (N3), reverse role lookup (N4) and historical state (N5).
- **Client changes:** large. Every client needs a new backend.
- **Cost for the operator:** zero if a node already exists. It needs a public proxy.
- **Problem:** degraded user experience and use cases that stop working. It **does not meet "every use case"**.

## E2. "Full read node": indexes and a public mode inside `babylon-node`

> [!IMPORTANT]
> - **Is it possible?** Yes.
> - **Difficulty:** High (it touches the node core).
> - **Effort and timeline:** 6-10 person-months in the node + ~4-6 in clients (estimate); long lead time.

- **Coverage:** 100 % of what clients use today, except arbitrary substate history (see E3).
- **Client changes:** many, as in E1, unless the node also answers in the API format wallets use today; in that case none are needed to keep working.
- **Cost per operator:** marginal (optional indexes).
- **Decentralisation:** maximum. Any full node can serve reads (discouraged on validators, section [6.1](06-recommended-solution.md#s6-1)).
- **Problem:**
  - It needs Java and Rust work inside the node, and a node release.
  - With the Foundation in maintenance mode, it depends on community developers and governance.
  - Long lead time.

## E3. A light indexer next to the node, replacing the Gateway

> [!IMPORTANT]
> - **Is it possible?** Yes (without arbitrary historical state).
> - **Difficulty:** Medium.
> - **Effort and timeline:** 4-7 person-months (+1-2 for explorers); months (estimate).

It is a separate process, written in Rust with RocksDB or SQLite. It works like this:

- It consumes `/core/stream/transactions` from the local node.
- It keeps **only the indexes the node lacks** (detailed in section [7.1](07-per-repository-changes.md#s7-1)):
  - (a) entity → transaction markers, with fungible and NFT changes, manifest class and events;
  - (b) NFT → vault → global entity;
  - (c) subintents;
  - (d) a reverse index of role rules;
  - (e) metadata `last_updated`;
  - (f) locker touches;
  - (g) validator emissions;
  - (h), (i) and (j) for explorers: holders, statistics and search.
- **It contains none of the Gateway software:** no DataAggregator, no PostgreSQL, no substate history. It only keeps the **API format, requests and responses** (the OpenAPI contract), so already-released wallets work without an update. Once clients adopt the node's native API (E2), that compatibility layer can be removed.
- It answers requests **with the same routes and the same JSON that wallets and dApps use today** (those in chapter [2](01-02-current-situation.md#s2), about 25 routes). Already-released apps then work by changing only the URL. It is a matter of format: there is no Gateway behind it. Current-state reads are delegated live to the node (Core and Engine State APIs); its own indexes are used only for what the node cannot answer.

Assessment:

- **Coverage:** ~100 % of what wallets, the RDT, ROLA, the extension and the SDK use.
  - It does **not** include arbitrary `at_ledger_state`. Today the only significant use is consultation's. Sargon uses it to keep paged reads consistent, which is solved by pinning the version.
- **Client changes:** **none to get it working.** The wallets already let users add a custom URL (Settings → Gateways) pointing to the indexer, and the RDT has `gatewayBaseUrl`. Small changes are needed for automatic endpoint selection.
- **Size:** **~10-50 GB**, versus ~2.5 TB. The basis is ~10-15 million user transactions × ~0.5-1 KB per marker (an assumption to validate with a prototype), giving ~5-15 GB; the rest of the range covers the NFT-location, holder, search and auxiliary tables, skipping the ~555 million round updates. *This is an estimate to validate with a prototype.*
- **First sync:** walk ~567 million stream versions. Assuming an ingestion rate of 2,000-5,000 versions/s for a process without Postgres (an assumption to validate), that is ~1.3-3.3 days. Signed snapshots can shorten it. *Estimate.*
- **Cost per operator:** marginal on top of an existing node (disk and a little CPU).
- **Decentralisation:** high. Any node operator can run it.
- **Problem:**
  - That request format (a stable OpenAPI contract) must be kept while apps use it. Once clients adopt the node's native API, it can be retired.
  - Trust: the client trusts the node, as it trusts the Gateway today.

## E4. Indexing inside each client

> [!IMPORTANT]
> - **Is it possible?** Not for history; only as a complement to watch transactions.
> - **Difficulty:** —
> - **Effort and timeline:** — (not a solution on its own).

Each wallet walks the stream to build its own history.

- **Data:** 3.6-4.4 versions/s continuously, and ~567 million versions of history.
- **Viability:** not viable on mobile for history. It only works for watching forward during a session.
- **Reasonable use:** only as a complement, for example watching a transaction or subintent from the version at which it was submitted.

## E5. A light client with cryptographic proofs

> [!IMPORTANT]
> - **Is it possible?** Yes, in the long term.
> - **Difficulty:** Very high.
> - **Effort and timeline:** ~40-80 more person-months; several years (estimate; see section [6.2](06-recommended-solution.md#s6-2)).

- **Babylon:** today there is no endpoint that returns a proof for a specific piece of data; `/stream/proofs` only gives the signed ledger proofs. But those proofs already include the Merkle roots of state, transactions and receipts, so this can be built without changing consensus (section [6.2](06-recommended-solution.md#s6-2)).
- **Hyperscale:** it has JMT multiproofs and QC-certified headers, but only between validators, and it is not compatible with Babylon.
- **Value:** in the long term, this is how to **avoid trusting** the node, and it is part of the ideal solution ([6.2](06-recommended-solution.md#s6-2)). It does not solve the 2026 problem.

## E6. Redesigning data on ledger

> [!IMPORTANT]
> - **Is it possible?** Only partly, application by application.
> - **Difficulty:** Low-medium per application.
> - **Effort and timeline:** depends on each application.

Change blueprints so the information lives on ledger and needs no indexing. Examples: a holder registry, tallies inside the component, complete events.

- **Scope:** partial. Only for specific applications such as consultation, and it cannot change native resources or accounts.
- **Value:** useful as a complement, but not general.

<a id="s5"></a>

# 5. Comparison with data

Scoring scale: 1 = bad, 5 = very good.

| Criterion | E1 Direct node | E2 Read node | **E3 Light indexer** | E4 Client | E5 Light client | E6 On ledger |
|---|---|---|---|---|---|---|
| Use-case coverage | 2 (partial: 9 of 19 fully and 2 partly) | 5 | **5 (~100 %; no arbitrary history)** | 1 | 2 | 1-2 |
| Full transaction explorers | 1 (no history, holders or statistics) | 4 (if h, i, j are added) | **5 (with h, i, j)** | 1 | 1 | 1 |
| Removes the Gateway as a central service | 5 | 5 | **5** (no Gateway remains; only the request format is reused) | 5 | 5 | 3 |
| Storage per operator | 0 extra | Tens of GB (est.) | **~10-50 GB (est.)** | — | — | — |
| Monthly cost per operator | ~0 on top of the node | ~0 | **~€0-20 on top of the node (est.)** | — | — | — |
| Client changes required | Many | Many (none if the node also serves the current format) | **None to get it working** | Many | Many | Per dApp |
| Available before December 2026? | Partly | No | **Prototype of the critical subset: possible** | No | No | No |
| Development effort (est.) | 15-25 person-months | 6-10 person-months (node) + ~4-6 (clients) | **4-7 person-months** (+1-2 for explorer indexes) | — | ~40-80 person-months; years | Per dApp |
| Technical risk | Medium | Medium-high (node core) | **Medium** | High | Very high | Low |
| Valid for Xi'an / Hyperscale | No | No | Partly (interface reusable) | — | Partly (would need adapting) | No (they must be rewritten) |

**Reading the table:**

- **E1** does not meet the "every use case" requirement.
- **E2** is the medium-term destination, but it is slow to build and touches the node core.
- **E3** gives full coverage at E1's running cost, without waiting for wallet releases. Its index code is also what later moves into the node (E2).

---

[← Index](../../README.md) · [Previous: Chapter 3: what the node offers today and what it lacks](03-what-the-node-offers.md) · [Next: Chapter 6: recommended solution, storage, ideal solution and risks](06-recommended-solution.md)
