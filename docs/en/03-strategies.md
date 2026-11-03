[← Index](../../README.md) · [Previous: Section 3: what the node offers today and what it lacks](02-what-the-node-offers.md) · [Next: Section 6: recommended solution, storage, and the ideal long-term solution](04-recommended-solution.md)

# 4. Strategies evaluated

## E1. Clients talk directly to the Core and Engine State APIs, with no node changes

- **Coverage:** ~80-85 %. It loses N1-N5: full history, NFT location, reverse role lookup, snapshots and pre-authorization status.
- **Client changes:** large. Every client needs a new backend.
- **Cost for the operator:** zero if a node already exists. It needs a public proxy.
- **Problem:** degraded user experience and broken use cases. It **does not meet "every use case"**.

## E2. "Full read node": indexes and a public mode inside `babylon-node`

- **Coverage:** 100 % of what clients use today, except arbitrary substate history (see E3).
- **Client changes:** those of E1. The node can also answer in the request format that wallets use today.
- **Cost per operator:** marginal (optional indexes).
- **Decentralisation:** maximum. Any node, including the 100 active validators, can serve.
- **Problem:**
  - It needs Java and Rust work inside the node, and a node release.
  - With the Foundation in maintenance mode, it depends on community developers and governance.
  - Long lead time.

## E3. A lightweight indexer next to the node, replacing the Gateway

It is a separate process, written in Rust with RocksDB or SQLite. It works like this:

- It consumes `/core/stream/transactions` from the local node.
- It keeps **only the indexes the node lacks**:
  - entity → transaction markers, with fungible and NFT changes, manifest class and events;
  - NFT → vault → global entity;
  - subintents;
  - a reverse index of role rules;
  - metadata `last_updated`;
  - locker touches;
  - validator emissions.
- **It contains none of the Gateway software:** no DataAggregator, no PostgreSQL, no substate history. It only keeps the **response format** (the OpenAPI contract), so already-released wallets work without an update. Once clients adopt the node's native API (E2), that compatibility layer can be removed.
- It answers requests **with the same routes and the same JSON that wallets and dApps use today** (those in section 2, about 25 routes). Already-released apps then work by changing only the URL. It is a matter of format: there is no Gateway behind it. Current-state reads are delegated live to the node (Core and Engine State APIs); its own indexes are used only for what the node cannot answer.

Assessment:

- **Coverage:** ~100 % of what wallets, the RDT, ROLA, the extension and the SDK use.
  - It does **not** include arbitrary `at_ledger_state`. Today only consultation relies on that seriously. Sargon uses it to keep paged reads consistent, which can be solved by pinning the version.
- **Client changes:** **none to get it working.** The wallets already let users add a custom Gateway in Settings, and the RDT has `gatewayBaseUrl`. Small changes are needed for automatic endpoint selection.
- **Size:** **~10-50 GB**, versus ~2.5 TB. The basis is ~10-15 million user transactions × ~0.5-1 KB per marker, plus the NFT location table, skipping the ~550 million round changes. *This is an estimate to validate with a prototype.*
- **First sync:** walk ~567 million stream versions. At an ingestion rate of 2,000-5,000 versions/s for a process without Postgres, that is ~1.5-3.5 days. Signed snapshots can shorten it. *Estimate.*
- **Cost per operator:** marginal on top of an existing node (disk and a little CPU).
- **Decentralisation:** high. Any node operator can run it.
- **Problem:**
  - That request format (a stable OpenAPI contract) must be kept while apps use it. Once clients adopt the node's native API, it can be retired.
  - Trust: the client trusts the node, as it trusts the Gateway today.

## E4. Indexing inside each client

Each wallet walks the stream to build its own history.

- **Data:** 3.6-4.4 versions/s continuously, and ~567 million versions of history.
- **Viability:** not viable on mobile for history. It only works for watching forward during a session.
- **Reasonable use:** only as a complement, for example watching a transaction or subintent from the version at which it was submitted.

## E5. A light client with cryptographic proofs

- **Babylon:** today there is no endpoint that returns a proof for a specific piece of data; `/stream/proofs` only gives the signed ledger proofs. But those proofs already include the Merkle roots of state, transactions and receipts, so this can be built without changing consensus (section 6.2).
- **Hyperscale:** it has JMT multiproofs and QC-certified headers, but only between validators, and it is not compatible with Babylon.
- **Value:** in the long term, this is how to **avoid trusting** the node, and it is part of the ideal solution (6.2). It does not solve the 2026 problem.

## E6. Redesigning data on ledger

Change blueprints so the information lives on ledger and needs no indexing. Examples: a holder registry, tallies inside the component, complete events.

- **Scope:** partial. Only for specific applications such as consultation, and it cannot change native resources or accounts.
- **Value:** useful as a complement, but not general.

# 5. Comparison with data

Scoring scale: 1 = bad, 5 = very good.

| Criterion | E1 Direct node | E2 Read node | **E3 Light indexer** | E4 Client | E5 Light client | E6 On ledger |
|---|---|---|---|---|---|---|
| Use-case coverage | 2 (~80-85 %) | 5 | **5 (~100 %; no arbitrary history)** | 1 | 2 | 1-2 |
| Full transaction explorers | 1 (no history, holders or statistics) | 4 (if h, i, j are added) | **5 (with h, i, j)** | 1 | 1 | 1 |
| Removes the Gateway as a central service | 5 | 5 | **5** (no Gateway remains; only the request format is reused) | 5 | 5 | 3 |
| Storage per operator | 0 extra | Tens of GB (est.) | **~10-50 GB (est.)** | — | — | — |
| Monthly cost per operator | ~0 on top of the node | ~0 | **~€0-20 on top of the node** | — | — | — |
| Client changes required | Many | Many | **None to get it working** | Many | Many | Per dApp |
| Available before December 2026? | Partly | No | **Prototype of the critical subset: possible** | No | No | No |
| Development effort (est.) | 15-25 person-months | 6-10 pm (node) + clients | **4-7 pm** | — | Years | Per dApp |
| Technical risk | Medium | Medium-high (node core) | **Medium** | High | Very high | Low |
| Valid for Xi'an / Hyperscale | No | No | Partly (interface reusable) | — | Yes | Yes |

**Reading the table:**

- **E1** fails "every use case".
- **E2** is the ideal end state, but it is slow and touches the node core.
- **E3** gives full coverage at E1's cost, without waiting for wallet releases. Its index code is also what later moves into the node (E2).

---

[← Index](../../README.md) · [Previous: Section 3: what the node offers today and what it lacks](02-what-the-node-offers.md) · [Next: Section 6: recommended solution, storage, and the ideal long-term solution](04-recommended-solution.md)
