[← Index](../../README.md) · [Previous: Chapters 4-5: the six strategies evaluated and their comparison](04-05-strategies.md) · [Next: Chapter 7: concrete changes in each repository](07-per-repository-changes.md)

<a id="s6"></a>

# 6. Recommended solution and why

> [!IMPORTANT]
> - **Is it possible?** Yes. It is the recommended short-term solution.
> - **Difficulty:** Medium.
> - **Effort and timeline:** 7-12 person-months for the minimum viable solution; months (estimate).

## Short-term infrastructure: no Gateway

```mermaid
flowchart TB
    APPS["<b>Users and applications</b><br/>iOS and Android wallets, Connector Extension,<br/>dApps with RDT, backends with ROLA<br/>(current versions: only the URL changes)"]
    EXP["<b>Reference explorer</b><br/>static site, open source"]
    REG["<b>1. On-ledger registry</b><br/>api_endpoints metadata"]
    READ["<b>2. Read nodes</b><br/>many independent operators (A, B, C…)<br/>full node + light indexer"]
    SELF["<b>Auxiliary services</b> anyone can host<br/>signalling and TURN, Radix Connect and iroh relays,<br/>configurable prices"]
    ARCH[("Verifiable Olympia archive<br/>IPFS, BitTorrent, Arweave")]
    VAL[("Validators<br/>small profile: R1-R4")]
    APPS -->|"read the list"| REG
    APPS -->|"query with failover"| READ
    EXP --> READ
    APPS -.-> SELF
    EXP -.-> ARCH
    READ --> VAL
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class READ,REG,SELF,ARCH good
```

Detail: how applications find read nodes.

```mermaid
flowchart TB
    C["Wallets, dApps, ROLA and explorers"]
    REG["<b>1.</b> On-ledger registry<br/>api_endpoints metadata<br/>of the validators"]
    A["<b>Read node A</b><br/>full node +<br/>light indexer ~10-50 GB"]
    B["<b>Read node B</b><br/>full node +<br/>light indexer"]
    Z["<b>Read node …</b><br/>full node +<br/>light indexer"]
    R(("Radix network"))
    C -->|"read the list"| REG
    C --> A
    C -->|"<b>2.</b> query with<br/>failover"| B
    C --> Z
    A --> R
    B --> R
    Z --> R
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class REG,A,B,Z good
```

## How a read node works

```mermaid
flowchart TB
    subgraph NODE["Full node (unchanged)"]
        CORE["Core API and Engine State API<br/>127.0.0.1 only"]
        LEDGER[("Full ledger:<br/>transactions, receipts, state")]
        CORE --- LEDGER
    end
    subgraph IDX["Light indexer (separate process)"]
        ING["Stream ingestion<br/>from version 1, genesis included"]
        IX[("Small indexes:<br/>per-entity history, NFT location,<br/>holders, subintents, role rules,<br/>statistics, search")]
        API["Public read API<br/>CORS, rate limiting, TLS"]
        RES["Resubmission of pending<br/>transactions with backoff"]
        ING --> IX
    end
    CORE -- "/stream/transactions" --> ING
    APP["Wallet, dApp or explorer"] --> API
    API -- "current state: balances, NFTs, metadata" --> CORE
    API -- "history, holders, NFT location" --> IX
    API -- "submit transaction" --> RES
    RES --> CORE
```

## Submitting a transaction without the Gateway

```mermaid
sequenceDiagram
    autonumber
    participant W as Wallet
    participant L as Read node
    participant N as Full node
    W->>L: transaction preview
    L->>N: /transaction/preview
    N-->>L: result
    L-->>W: result and fees
    Note right of W: user signs
    W->>L: submit transaction
    L->>N: /lts/transaction/submit
    loop while the epoch window is valid
        L->>N: /lts/transaction/status
        alt not committed
            L->>N: resubmit with backoff
        end
    end
    W->>L: status?
    L-->>W: CommittedSuccess
```

**Strategy: E3 first, then E2, with E6 as a complement.** Two cross-cutting pieces are added:

- **(D) on-ledger discovery:** validators publish their access URLs in their component metadata, for example under the key `api_endpoints`;
- **(L) a `LedgerBackend` abstraction** in clients.

**Why:**

1. **Time.** The Gateway is funded only until 31 Dec 2026. E3 is the only option that can **serve today's wallets without new releases**, because they already accept a custom Gateway URL.
2. **Cost.** It goes from the Gateway's ~2.5 TB, and the ~€300-400/month a community proposal estimated for running it (section [1.1](01-02-current-situation.md#s1-1)), to ~10-50 GB (estimate) added to a node that already exists. The measured data explains why: ~97-98 % of state versions are round updates that do not need indexing.
3. **Real decentralisation.** If every validator or full node can serve the API at near-zero cost and announce itself on ledger, the "default provider" disappears. That is the condition for the network to serve any user without depending on a particular provider.
4. **Upgrade path.**
   - E3's indexes later move into `babylon-node` (E2).
   - The clients' `LedgerBackend` interface lets them switch to the node's native API later, to the light client (E5) and, if adopted, to Hyperscale (chapter [11](11-hyperscale-evolution.md#s11)), without rewriting application logic.
5. **Contained risk.** It does not touch consensus or the node core. It is a separate process that can be stopped or restarted without affecting the node.

**What E3 does not provide, and how it is covered:**

- **Arbitrary historical state** (old `at_ledger_state`):
  - E6, with forward capture in consultation;
  - or an "archive node" option in E2, with long history and `ByTimestamp`.
- **Trust in the node:** query several nodes discovered on ledger and compare `state_version` and the accumulator hash (`/stream/proofs`). Eventually, E5.

<a id="s6-1"></a>

## 6.1 Impact on validator storage

**Short answer:** it does not increase significantly. And validators **do not have to run it**.

1. **It is optional and separate from the validator.**
   - Only those who want to serve public reads run the indexer.
   - It is advisable that a validator does **not** serve public traffic, so it is not exposed to attacks or to load that affects consensus. Reads are served by full nodes (not validators) run by the community, wallet providers, dApps, or validator operators on a separate machine.
   - A validator that does not run it sees **no change** in storage.
   - In E2, the in-node indexes would be flags that are **off by default**.
2. **If an operator runs it on the same machine as their node:**

   | Item | Size |
   |---|---|
   | Recommended node disk (official docs, "initially") | 500 GB |
   | Light indexer (E3), estimated | ~10-50 GB |
   | Relative increase | **~2-10 %** of the recommended disk |
   | Indexer growth | ~0.1-0.3 GB/month at today's rate (~4,400 user transactions/day × 0.5-1 KB, plus auxiliary indexes); ~1 GB/month at the March 2025 peak (~35,000/day) |
   | For comparison: today's Gateway | ~2.5 TB, **+60 GB/month** |

   **Comparison per public access point** (what someone serving wallets, dApps and explorers needs):

   | | Today, with the Gateway | With the light indexer |
   |---|---|---|
   | Full node (complete ledger: transactions, receipts and state; both models need it) | Yes, one or more: the DataAggregator reads from full nodes | Yes, one |
   | Plus | ~2.5 TB PostgreSQL database (+60 GB/month) | ~10-50 GB of indexes (estimated) |
   | Total extra on top of the node | **~2.5 TB** | **~10-50 GB** |

   In other words: a full node **already stores all network data today**, because it needs it to sync and validate, and that does not change. The light indexer **only adds its indexes on top of the node's data**: slightly more than a bare node (~2-10 %) and far less than a node plus Gateway (50 to 250 times less; estimate).

   The indexer stays small because it **does not duplicate state or transactions**, which already live in the node. It only stores pointers (entity → `state_version`) and a few small tables, and it skips round updates, which are ~97-98 % of versions.
3. **Margin of error.**
   - Only the **compressed** size of the community snapshot is known (274 GiB, Sep 2026); the on-disk size of a mainnet node has not been published, and the docs only give the 500 GB "initial" figure.
   - The 10-50 GB for the indexer is an **estimate** based on the user-transaction measurements. The resource-holder index (h) and the NFT-location index (b) depend on the number of resource-holder pairs and existing NFTs, which were not measured.
   - The first prototype must measure both before committing to figures.
4. **What would grow a lot:** an "archive node" with full substate history (N5), which is essentially what makes the Gateway 2.5 TB. That is why this report does **not** propose it for validators. It would be an option only for those who need arbitrary historical state. For governance it is replaced by capturing the snapshot when the vote opens (E6).

<a id="s6-2"></a>

## 6.2 The best solution with unlimited time: verifiable reads from any node

> [!IMPORTANT]
> - **Is it possible?** Yes, without changing consensus (verifiable indexes do need a protocol update).
> - **Difficulty:** Very high.
> - **Effort and timeline:** ~40-80 more person-months; years (estimate).

The short-term solution (E3) removes the Gateway, but the client **still has to trust** the node that answers it, just as it trusts the Gateway today. With unlimited time, the ideal solution removes that trust too: **any node, even an unknown one, can serve data, and the wallet checks for itself that the data is correct.**

**Key fact, verified in the node code:** Babylon already has almost everything needed.

- The ledger proofs signed by validators include three Merkle tree roots (`LedgerHashes`, `core-api-schema.yaml:1846-1870`):
  - `state_tree_hash` (all state);
  - `transaction_tree_hash` (all transactions);
  - `receipt_tree_hash` (all results).
- The node maintains the state tree (`state_hash_tree.*`).
- `/stream/proofs` already delivers the signed proofs, including epoch changes, which say which validators sign the next epoch.
- **What is missing** is an endpoint that returns the **Merkle proof** for a specific piece of data, and a client that verifies it.

### Long-term infrastructure: the ideal solution

```mermaid
flowchart TB
    subgraph APPS["Users and applications with a light client"]
        W["Wallets<br/>verify every response"]
        DAPP["dApps, ROLA and explorers<br/>light client in WebAssembly"]
    end
    subgraph NET["Radix network: a single Rust binary"]
        N2[("Any other node<br/>even anonymous")]
        N1[("Node with native read API<br/>indexes + Merkle proofs")]
        V[("Validators<br/>4 GB RAM, pruned node")]
        A[("Archive nodes<br/>full history")]
        N1 --- V
        N2 --- V
        V --- A
    end
    SNAP[("Verifiable state snapshots<br/>IPFS, BitTorrent<br/>a new node bootstraps in hours")]
    HIST[("Verifiable old history<br/>Babylon and Olympia")]
    INC["Incentives for serving reads<br/>decided by governance"]
    W -->|"QUIC + proofs"| N1
    W -->|"if one fails, another"| N2
    DAPP -->|"QUIC or HTTPS<br/>+ proofs"| N1
    INC -.-> N1
    A -->|"produces"| SNAP
    A -->|"publishes"| HIST
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class W,DAPP,V,N1,N2,A,SNAP,HIST good
```

#### How the wallet verifies a response

```mermaid
sequenceDiagram
    autonumber
    participant W as Wallet<br/>with light client
    participant Q as Any node<br/>even unknown
    participant V as Validators
    V-->>Q: sign the ledger proof with state_tree_hash
    W->>Q: give me my account balance
    Q-->>W: data + Merkle proof + signed ledger proof
    Note right of W: did more than 2/3 of the epoch stake sign?
    Note right of W: does the Merkle proof match state_tree_hash?
    alt both checks pass
        Note right of W: show the balance, without trusting the node
    else a check fails
        W->>Q: drop the node and ask another
    end
```

**Components of the ideal solution:**

| # | Component | What it does |
|---|---|---|
| 1 | **Every node is a read node** (E2) | Indexes live inside `babylon-node` as options. An open, versioned **native read API** replaces the legacy format. There is no separate process. |
| 2 | **Reads with proofs** | Each response includes the data, its Merkle proof against `state_tree_hash`, `transaction_tree_hash` or `receipt_tree_hash`, and the validator-signed ledger proof. This covers balances, NFTs, metadata, component state, transactions and receipts. Consensus does not change. |
| 3 | **Light client in wallets and dApps** | In Sargon (Rust), and in WebAssembly for the browser (RDT, ROLA, explorers). It follows epoch changes from a checkpoint shipped with the app and checks that more than 2/3 of stake signed each proof. With that, it verifies every response. **The client stops trusting the node.** |
| 4 | **Verifiable indexes** | A Merkle proof shows that a piece of data *is there*, but not that a list (history, holders) is *complete*. That needs a protocol update: the engine keeps, in state, an accumulator of each entity's history and holders, so they fall under `state_tree_hash` and can be proven too. Until then, several independent nodes are cross-checked. |
| 5 | **Discovery and transport without intermediaries** | An on-ledger registry of read nodes (validator metadata or another component), and a direct QUIC/iroh connection between wallet and node. This reduces the dependency on DNS and TLS certificates, and relays can be self-hosted or many; the same technology is already used in Radix Connect. |
| 6 | **Incentives to serve reads** | A share of network emissions or fees, or per-query XRD micropayments, for those who run read nodes. Governance decides. Without incentives, the service depends on operators' voluntary commitment. |
| 7 | **Verifiable historical archive** | Old history (Olympia included) lives in distributed storage (IPFS or Arweave). Its integrity is checked against `transaction_tree_hash` and `receipt_tree_hash`. No node has to keep all of it. |
| 8 | **Ready for Hyperscale** | Hyperscale already uses JMT multiproofs. The light client and the native API can be adapted to that network. The full migration to Hyperscale is analysed in chapter [11](11-hyperscale-evolution.md#s11). |

**Comparison with the short-term solution:**

| | Short term (E3 → E2) | Ideal |
|---|---|---|
| Gateway | Removed | Removed |
| Does the client trust the node? | Yes, though it can compare several | **No, it verifies every response** |
| Who can serve data? | Known node operators | **Any node, even anonymous** |
| Protocol changes | None | Only for verifiable indexes (item 4) |
| Lead time | Months | **Years** |
| Effort (est.) | 7-12 person-months minimum viable; 21-35 full | **~40-80 person-months** more, with protocol experts |

**Why it is the best with unlimited time:** it is the only option in which the network depends on no one, neither a Gateway nor the honesty of a node. It gives the same level of guarantee as the chain itself, extended to every user's wallet.

```mermaid
%%{init: {"flowchart": {"rankSpacing": 80, "nodeSpacing": 30}}}%%
flowchart TB
    subgraph ROW1[" "]
        direction LR
        subgraph S0["<b>TODAY</b>"]
            direction LR
            S0A["<b>Gateway</b><br/>central, single operator"]
            S0B["<b>Network nodes</b><br/>16 GB of RAM"]
        end
        S0 --> S1["<b>STAGE 1 · months</b><br/>light indexer (E3)<br/>no Gateway · R1-R4"] --> S2["<b>STAGE 2 · 2027</b><br/>indexes in the node (E2)<br/>4-8 GB nodes · R9"]
    end
    subgraph ROW2[" "]
        direction LR
        S3["<b>STAGE 3 · years</b><br/>verifiable reads<br/>and light client"] --> D{"<b>Governance</b><br/>adopt<br/>Hyperscale?"}
    end
    subgraph ROW3[" "]
        direction TB
        FA["<b>FINAL A</b><br/>if not, or later<br/>keep improving Babylon<br/>node in Rust (R11)"]
        FB["<b>FINAL B</b><br/>if adopted<br/>Radix on Hyperscale<br/>(chapter 11)"]
    end
    ROW1 --> ROW2 --> ROW3
    classDef today fill:#ffd6d6,stroke:#c00,color:#000
    classDef step fill:#dbe9ff,stroke:#2b5fb4,color:#000
    classDef decision fill:#fff3c4,stroke:#b58900,color:#000
    classDef final fill:#d9f7d9,stroke:#080,color:#000
    classDef row fill:none,stroke:none
    class S0A,S0B today
    style S0 fill:#fff5f5,stroke:#c00,color:#000
    class S1,S2,S3 step
    class D decision
    class FA,FB final
    class ROW1,ROW2,ROW3 row
```

**How it fits with the short term:** these are stages, not alternatives. E3 solves the 2026 problem. E2 removes the separate process. Items 2-8 add verification on top of what E2 already serves. The clients' `LedgerBackend` interface is what lets them move from one stage to the next without rewriting applications.

<a id="s6-3"></a>

## 6.3 Risks of removing the Gateway and how to mitigate them

| Risk | Consequence | Mitigation |
|---|---|---|
| **A node returns incorrect data**, by mistake or maliciously | The user sees false balances or state | Query several access points and compare `state_version` and the accumulator hash ([10.2](10-implementation-guide.md#s10-2)); in the long term, verifiable reads ([6.2](#s6-2)) |
| **Few read-node operators** | A central dependency reappears | Near-zero cost per operator (E3); on-ledger discovery; incentives decided by governance ([6.2](#s6-2), item 6; chapter [12](12-references-other-networks.md#s12)) |
| **Denial-of-service attacks** on read nodes | Slow or unavailable service | Public mode with per-IP limits and a route allow-list (N6); several operators; validators kept separate from read nodes ([6.1](#s6-1)) |
| **The indexer does not exactly reproduce the responses** wallets expect | Application errors | Conformance tests against the public Gateway **before it is switched off** ([10.1](10-implementation-guide.md#s10-1)) |
| **Few archive nodes** | New nodes cannot obtain history or snapshots | Several archive nodes in different hands; verifiable snapshots on distributed storage (R9) |
| **A transaction is lost** if the access point restarts | The user believes something was sent that never committed | Resubmission in the indexer and in clients ([10.1](10-implementation-guide.md#s10-1), [10.4](10-implementation-guide.md#s10-4)) |
| **Lack of funding or developers** | The plan slips beyond 31 Dec 2026 | Prioritise the minimum viable solution (7-12 person-months); an early governance decision |
| **Bugs in new software** (indexer, node changes) | Service failures or, in the node, consensus failures | Deploy on Stokenet first; options off by default; gradual adoption ([10.7](10-implementation-guide.md#s10-7)) |

---

[← Index](../../README.md) · [Previous: Chapters 4-5: the six strategies evaluated and their comparison](04-05-strategies.md) · [Next: Chapter 7: concrete changes in each repository](07-per-repository-changes.md)
