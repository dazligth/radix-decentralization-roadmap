[← Index](../../README.md) · [Previous: Section 12: references from other decentralized networks](10-references-other-networks.md)

# Sources

- Code: `babylon-node` (`develop` and release `v1.4.0.0`), `babylon-gateway` (`main`), `sargon` (`main`), `babylon-wallet-ios` (`main`), `rola`, `babylon-nodecli` (cloned 3 Nov 2026), and the local repositories. `file:line` references refer to those versions.
- Measurements: public API `mainnet.radixdlt.com` (`/status/gateway-status`, `/stream/transactions`), 3 Nov 2026. Daily rates come from 100-transaction samples per date and are orders of magnitude.
- [RFPs for Gateway and Relay Services (3 Feb 2026)](https://www.radixdlt.com/blog/the-next-phase-of-decentralization-rfps-for-gateway-and-relay-services)
- [The Foundation Operational Stack: Mapping the 2026 Transition](https://www.radixdlt.com/blog/the-foundation-operational-stack-mapping-the-2026-transition)
- [Foundation Update: Moving to Maintenance Mode (28 Apr 2026)](https://www.radixdlt.com/blog/foundation-update-moving-to-maintenance-mode)
- [2026 Strategy: The Next Chapter of Radix](https://www.radixdlt.com/blog/2026-strategy-the-next-chapter-of-radix)
- [Interim Hyperscale: Closing the Chapter](https://www.radixdlt.com/blog/interim-hyperscale-closing-the-chapter)
- [RadixTalk — Foundation RFP: Babylon Gateway (posts 15, 22, 25, 26, 29, 38)](https://radixtalk.com/t/foundation-rfp-babylon-gateway/2202)
- [RadixTalk — Action on P3 Foundation Services](https://radixtalk.com/t/action-on-p3-foundation-services/2238)
- [Radix docs — Node setup (node hardware requirements)](https://docs.radixdlt.com/docs/node-setup)
- [Radix docs — Network Gateway](https://docs.radixdlt.com/docs/network-gateway)
- [Radix docs — Optimizing Performance](https://docs.radixdlt.com/docs/node-optimizing-performance)
- [Community ledger snapshots (radix.live), size history](https://snapshots.radix.live/Validators-Latest/)
- [PublicNode — snapshots](https://www.publicnode.com/snapshots)
- [Radix Labs Roadmap — to Hyperscale and beyond (30 Nov 2024)](https://www.radixdlt.com/blog/radix-labs-roadmap---to-hyperscale-and-beyond)
- [radix.wiki — Xi'an](https://radix.wiki/contents/tech/releases/radix-mainnet-xian)
- [RadixTalk — Babylon node betatesting (betanet RAM measurements, 2023)](https://radixtalk.com/t/babylon-node-betatesting-first-impressions/1792)
- [hyperscale-rs (GitHub)](https://github.com/hyperscalers/hyperscale-rs) and [hyperscale-vm (GitHub)](https://github.com/hyperscalers/hyperscale-vm), reviewed in Nov 2026 (commit 6204357 and `vm` submodule at ddb17014)
- Node code: `radixdlt/babylon-node` branch `develop` (12919a01) and tag `v1.4.0.0` (7400951e); `radixdlt/babylon-nodecli`.
- [CometBFT — State sync (ABCI++ specification)](https://github.com/cometbft/cometbft/blob/v0.38.x/spec/abci/abci++_app_requirements.md) and [Cosmos Hub — node sync](https://docs.cosmos.network/hub/latest/node/sync.md)
- [Cosmos SDK — pruning](https://github.com/cosmos/cosmos-sdk/blob/main/store/pruning/README.md), [migrations and genesis export](https://docs.cosmos.network/evm/latest/documentation/concepts/migrations.md) and [Cosmos Hub upgrade history](https://docs.cosmos.network/hub/latest/upgrade/history.md)
- [EIP-4444](https://eips.ethereum.org/EIPS/eip-4444) and [Ethereum Foundation — partial history expiry (8 Jul 2025)](https://blog.ethereum.org/2025/07/08/partial-history-exp)
- [Ethereum — nodes and clients (checkpoint sync)](https://ethereum.org/en/developers/docs/nodes-and-clients/), [weak subjectivity](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/weak-subjectivity/), [light clients](https://ethereum.org/en/developers/docs/nodes-and-clients/light-clients/) and [client diversity](https://ethereum.org/en/developers/docs/nodes-and-clients/client-diversity/)
- [EIP-1186 (`eth_getProof`)](https://eips.ethereum.org/EIPS/eip-1186), [Altair specification](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/beacon-chain.md) and [Helios](https://github.com/a16z/helios)
- [Nimbus — hardware](https://nimbus.guide/hardware.html), [Nimbus on Raspberry Pi](https://nimbus.guide/pi-guide.html) and [Geth — hardware requirements](https://geth.ethereum.org/docs/getting-started/hardware-requirements)
- [Uniswap MerkleDistributor](https://github.com/Uniswap/merkle-distributor)
- [Firedancer / Frankendancer](https://github.com/firedancer-io/firedancer)
- [Pocket Network — rewards](https://docs.pocket.network/node-operators/rewards-economics/) and [The Graph — indexing](https://thegraph.com/docs/en/indexing/overview/)
- [RocksDB — memory usage](https://github.com/facebook/rocksdb/wiki/Memory-usage-in-RocksDB), [block cache](https://github.com/facebook/rocksdb/wiki/Block-Cache) and [Write Buffer Manager](https://github.com/facebook/rocksdb/wiki/Write-Buffer-Manager)
- [rust-rocksdb v0.24.0](https://github.com/rust-rocksdb/rust-rocksdb/tree/v0.24.0)
- Cardano: [Mithril](https://mithril.network/doc/), [bootstrapping a node with Mithril](https://mithril.network/doc/manual/getting-started/bootstrap-cardano-node), [IOG Q2 2026 report](https://services.iog.io/hubfs/Branded_Q2%202026%20delivery%20report.pdf), [IOG Q3 2025 report](https://www.iog.io/news/strengthening-cardanos-foundations-q3-2025-progress-report), [hardware requirements](https://developers.cardano.org/docs/operators/basics/hardware-requirements/), [cardano-db-sync](https://github.com/IntersectMBO/cardano-db-sync), [Koios](https://koios.rest/), [CIP-164 (Leios)](https://cips.cardano.org/cip/CIP-164) y [Dijkstra planning](https://product.cardano.intersectmbo.org/hardfork-planning/dijkstra/)
- Sui: [full node](https://docs.sui.io/guides/operator/sui-full-node), [data management and pruning](https://docs.sui.io/guides/operator/data-management), [snapshots](https://docs.sui.io/guides/operator/snapshots), [checkpoint verification](https://docs.sui.io/concepts/cryptography/system/checkpoint-verification) e [indexer and GraphQL](https://docs.sui.io/concepts/data-access/graphql-indexer)
- Aptos: [state sync](https://aptos.dev/network/nodes/configure/state-sync), [pruning](https://aptos.dev/network/nodes/configure/data-pruning), [requirements](https://aptos.dev/network/nodes/validator-node/node-requirements) e [indexer](https://aptos.dev/build/indexer)
- Solana: [Geyser](https://docs.anza.xyz/validator/geyser), [requirements](https://docs.anza.xyz/operations/requirements), [validator start](https://docs.anza.xyz/operations/guides/validator-start) y [Old Faithful](https://docs.old-faithful.net/)
- NEAR: [RPC node hardware](https://near-nodes.io/rpc/hardware-rpc), [archival nodes](https://near-nodes.io/archival/hardware-archival) y [NEAR Lake](https://docs.near.org/data-infrastructure/lake-framework/near-lake)
- Polkadot: [full node](https://docs.polkadot.com/node-infrastructure/run-a-node/relay-chain/full-node/), [warp sync](https://github.com/paritytech/substrate/pull/9227), [smoldot](https://github.com/smol-dot/smoldot), [light clients](https://docs.polkadot.com/reference/tools/light-clients/) y [validator requirements](https://docs.polkadot.com/node-infrastructure/run-a-validator/requirements/)
- Kaspa: [rusty-kaspa](https://github.com/kaspanet/rusty-kaspa), [kaspad v0.12.19](https://github.com/kaspanet/kaspad/releases/tag/v0.12.19), [Crescendo guide](https://raw.githubusercontent.com/kaspanet/rusty-kaspa/master/docs/crescendo-guide.md), [API references](https://docs.kaspa.org/references) y [Resolver](https://kaspa.aspectron.org/rpc/kaspa-resolver.html)
- Midnight: [how it works](https://docs.midnight.network/learn/how-midnight-works), [2024 release notes (Halo2)](https://docs.midnight.network/relnotes/2024-09_testnet_release_notes), [tokens](https://docs.midnight.network/tokens/overview), [explicit disclosure in Compact](https://docs.midnight.network/compact/explicit_disclosure), [local proving](https://docs.midnight.network/guides/local-proving), [Glacier Drop](https://midnight.network/blog/glacier-drop-claim-portal-now-open) y [Kachina (paper)](https://eprint.iacr.org/2020/543)

---

[← Index](../../README.md) · [Previous: Section 12: references from other decentralized networks](10-references-other-networks.md)
