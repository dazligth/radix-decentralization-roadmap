[← Index](../../README.md) · [Previous: Section 7: concrete changes in each repository](05-per-repository-changes.md) · [Next: Sources and methodology](07-sources.md)

# 8. Roadmap

```mermaid
gantt
    title Indicative roadmap
    dateFormat YYYY-MM-DD
    axisFormat %b %Y
    section Urgent
    Olympia archive                     :crit, 2026-11-05, 45d
    Light indexer prototype             :crit, 2026-11-05, 55d
    End of Gateway funding              :milestone, 2026-12-31, 0d
    section Parity
    Full indexes and explorers          :2027-01-01, 60d
    Clients with failover and discovery :2027-02-01, 90d
    section Node
    Indexes inside babylon-node         :2027-03-01, 270d
    section Ideal
    Verifiable reads and light client   :2027-09-01, 540d
```

```mermaid
flowchart LR
    H["Today<br/>central Gateway"] --> S1["Stage 1, months<br/>E3: light indexer<br/>next to each node"]
    S1 --> S2["Stage 2<br/>E2: indexes inside<br/>babylon-node"]
    S2 --> S3["Stage 3, years<br/>Verifiable reads<br/>and light client"]
    classDef bad fill:#ffd6d6,stroke:#c00,color:#000
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class H bad
    class S3 good
```

| Phase | Indicative dates | Content | Result |
|---|---|---|---|
| **F0** | Nov-Dec 2026 | **Verifiable archive of the Olympia ledger** (2.2), before it is switched off. `radix-lite-indexer` prototype with the critical subset: status, submit, preview, balances, NFTs, metadata, subintents, validators. `babylon-nodecli` profile. Conformance tests against the public Gateway. | Today's wallets, RDT and ROLA work by changing only the URL |
| **F1** | Jan-Feb 2027 | History, NFT-location and role-rule indexes. Explorer indexes (holders, statistics, search). Resubmission. Public mode. Snapshots. Validator `api_endpoints` convention. First release of `radix-explorer`. | Parity with what clients use. Explorers without the Gateway. Several node operators serving data with the indexer. |
| **F2** | Feb-Apr 2027 | Clients: lists with failover and discovery (Sargon, wallets, RDT, ROLA, extension, SDK). Resubmission in Sargon. Fixes (`BaseUrlInterceptor`, hard-coded mainnet, hard-coded services). | No central default provider |
| **F3** | 2027 | `babylon-node`: N1, N6, N5b, then N2-N4. Snapshot capture in consultation. | Every node is an access point; the indexer becomes optional |
| **F4** | Long term | Light client with proofs (E5), on Babylon if substate proofs are added, or on Hyperscale | Access without trusting the node |

---

[← Index](../../README.md) · [Previous: Section 7: concrete changes in each repository](05-per-repository-changes.md) · [Next: Sources and methodology](07-sources.md)
