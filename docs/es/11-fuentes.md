[← Índice](../../README.es.md) · [Anterior: Sección 12: referencias de otras redes descentralizadas](10-referencias-otras-redes.md)

# Fuentes

- Código: `babylon-node` (rama `develop` y versión `v1.4.0.0`), `babylon-gateway` (`main`), `sargon` (`main`), `babylon-wallet-ios` (`main`), `rola`, `babylon-nodecli` (clones del 3-nov-2026) y los repos locales. Las rutas `fichero:línea` se refieren a esas versiones.
- Mediciones: API pública `mainnet.radixdlt.com` (`/status/gateway-status`, `/stream/transactions`), 3-nov-2026. Las tasas diarias son muestras de 100 transacciones por fecha y sirven como orden de magnitud.
- [RFPs for Gateway and Relay Services (3-feb-2026)](https://www.radixdlt.com/blog/the-next-phase-of-decentralization-rfps-for-gateway-and-relay-services)
- [The Foundation Operational Stack: Mapping the 2026 Transition](https://www.radixdlt.com/blog/the-foundation-operational-stack-mapping-the-2026-transition)
- [Foundation Update: Moving to Maintenance Mode (28-abr-2026)](https://www.radixdlt.com/blog/foundation-update-moving-to-maintenance-mode)
- [2026 Strategy: The Next Chapter of Radix](https://www.radixdlt.com/blog/2026-strategy-the-next-chapter-of-radix)
- [Interim Hyperscale: Closing the Chapter](https://www.radixdlt.com/blog/interim-hyperscale-closing-the-chapter)
- [RadixTalk — Foundation RFP: Babylon Gateway (posts 15, 22, 25, 26, 29, 38)](https://radixtalk.com/t/foundation-rfp-babylon-gateway/2202)
- [RadixTalk — Action on P3 Foundation Services](https://radixtalk.com/t/action-on-p3-foundation-services/2238)
- [Radix docs — Node setup (requisitos de hardware del nodo)](https://docs.radixdlt.com/docs/node-setup)
- [Radix docs — Network Gateway](https://docs.radixdlt.com/docs/network-gateway)
- [Radix docs — Optimizing Performance](https://docs.radixdlt.com/docs/node-optimizing-performance)
- [Snapshots comunitarios del ledger (radix.live), histórico de tamaños](https://snapshots.radix.live/Validators-Latest/)
- [PublicNode — snapshots](https://www.publicnode.com/snapshots)
- [Radix Labs Roadmap — to Hyperscale and beyond (30-nov-2024)](https://www.radixdlt.com/blog/radix-labs-roadmap---to-hyperscale-and-beyond)
- [radix.wiki — Xi'an](https://radix.wiki/contents/tech/releases/radix-mainnet-xian)
- [RadixTalk — Babylon node betatesting (mediciones de RAM en betanet, 2023)](https://radixtalk.com/t/babylon-node-betatesting-first-impressions/1792)
- [hyperscale-rs (GitHub)](https://github.com/hyperscalers/hyperscale-rs) y [hyperscale-vm (GitHub)](https://github.com/hyperscalers/hyperscale-vm), revisados en nov-2026 (commit 6204357 y submódulo `vm` en ddb17014)
- Código del nodo: `radixdlt/babylon-node` rama `develop` (12919a01) y etiqueta `v1.4.0.0` (7400951e); `radixdlt/babylon-nodecli`.
- [CometBFT — State sync (especificación ABCI++)](https://github.com/cometbft/cometbft/blob/v0.38.x/spec/abci/abci++_app_requirements.md) y [Cosmos Hub — sincronización de nodos](https://docs.cosmos.network/hub/latest/node/sync.md)
- [Cosmos SDK — poda](https://github.com/cosmos/cosmos-sdk/blob/main/store/pruning/README.md), [migraciones y exportación de génesis](https://docs.cosmos.network/evm/latest/documentation/concepts/migrations.md) e [historial de actualizaciones de Cosmos Hub](https://docs.cosmos.network/hub/latest/upgrade/history.md)
- [EIP-4444](https://eips.ethereum.org/EIPS/eip-4444) y [Ethereum Foundation — expiración parcial del historial (8-jul-2025)](https://blog.ethereum.org/2025/07/08/partial-history-exp)
- [Ethereum — nodos y clientes (checkpoint sync)](https://ethereum.org/en/developers/docs/nodes-and-clients/), [weak subjectivity](https://ethereum.org/en/developers/docs/consensus-mechanisms/pos/weak-subjectivity/), [clientes ligeros](https://ethereum.org/en/developers/docs/nodes-and-clients/light-clients/) y [diversidad de clientes](https://ethereum.org/en/developers/docs/nodes-and-clients/client-diversity/)
- [EIP-1186 (`eth_getProof`)](https://eips.ethereum.org/EIPS/eip-1186), [especificación Altair](https://github.com/ethereum/consensus-specs/blob/master/specs/altair/beacon-chain.md) y [Helios](https://github.com/a16z/helios)
- [Nimbus — hardware](https://nimbus.guide/hardware.html), [Nimbus en Raspberry Pi](https://nimbus.guide/pi-guide.html) y [Geth — requisitos de hardware](https://geth.ethereum.org/docs/getting-started/hardware-requirements)
- [Uniswap MerkleDistributor](https://github.com/Uniswap/merkle-distributor)
- [Firedancer / Frankendancer](https://github.com/firedancer-io/firedancer)
- [Pocket Network — recompensas](https://docs.pocket.network/node-operators/rewards-economics/) y [The Graph — indexación](https://thegraph.com/docs/en/indexing/overview/)
- [RocksDB — uso de memoria](https://github.com/facebook/rocksdb/wiki/Memory-usage-in-RocksDB), [caché de bloques](https://github.com/facebook/rocksdb/wiki/Block-Cache) y [Write Buffer Manager](https://github.com/facebook/rocksdb/wiki/Write-Buffer-Manager)
- [rust-rocksdb v0.24.0](https://github.com/rust-rocksdb/rust-rocksdb/tree/v0.24.0)
- Cardano: [Mithril](https://mithril.network/doc/), [arranque de un nodo con Mithril](https://mithril.network/doc/manual/getting-started/bootstrap-cardano-node), [informe de IOG T2-2026](https://services.iog.io/hubfs/Branded_Q2%202026%20delivery%20report.pdf), [informe de IOG T3-2025](https://www.iog.io/news/strengthening-cardanos-foundations-q3-2025-progress-report), [requisitos de hardware](https://developers.cardano.org/docs/operators/basics/hardware-requirements/), [cardano-db-sync](https://github.com/IntersectMBO/cardano-db-sync), [Koios](https://koios.rest/), [CIP-164 (Leios)](https://cips.cardano.org/cip/CIP-164) y [planificación de Dijkstra](https://product.cardano.intersectmbo.org/hardfork-planning/dijkstra/)
- Sui: [nodo completo](https://docs.sui.io/guides/operator/sui-full-node), [gestión de datos y poda](https://docs.sui.io/guides/operator/data-management), [snapshots](https://docs.sui.io/guides/operator/snapshots), [verificación de checkpoints](https://docs.sui.io/concepts/cryptography/system/checkpoint-verification) e [indexador y GraphQL](https://docs.sui.io/concepts/data-access/graphql-indexer)
- Aptos: [state sync](https://aptos.dev/network/nodes/configure/state-sync), [poda](https://aptos.dev/network/nodes/configure/data-pruning), [requisitos](https://aptos.dev/network/nodes/validator-node/node-requirements) e [indexador](https://aptos.dev/build/indexer)
- Solana: [Geyser](https://docs.anza.xyz/validator/geyser), [requisitos](https://docs.anza.xyz/operations/requirements), [arranque del validador](https://docs.anza.xyz/operations/guides/validator-start) y [Old Faithful](https://docs.old-faithful.net/)
- NEAR: [hardware de nodos RPC](https://near-nodes.io/rpc/hardware-rpc), [nodos de archivo](https://near-nodes.io/archival/hardware-archival) y [NEAR Lake](https://docs.near.org/data-infrastructure/lake-framework/near-lake)
- Polkadot: [nodo completo](https://docs.polkadot.com/node-infrastructure/run-a-node/relay-chain/full-node/), [warp sync](https://github.com/paritytech/substrate/pull/9227), [smoldot](https://github.com/smol-dot/smoldot), [clientes ligeros](https://docs.polkadot.com/reference/tools/light-clients/) y [requisitos de validador](https://docs.polkadot.com/node-infrastructure/run-a-validator/requirements/)
- Kaspa: [rusty-kaspa](https://github.com/kaspanet/rusty-kaspa), [kaspad v0.12.19](https://github.com/kaspanet/kaspad/releases/tag/v0.12.19), [guía de Crescendo](https://raw.githubusercontent.com/kaspanet/rusty-kaspa/master/docs/crescendo-guide.md), [referencias de API](https://docs.kaspa.org/references) y [Resolver](https://kaspa.aspectron.org/rpc/kaspa-resolver.html)
- Midnight: [cómo funciona](https://docs.midnight.network/learn/how-midnight-works), [notas de versión de 2024 (Halo2)](https://docs.midnight.network/relnotes/2024-09_testnet_release_notes), [tokens](https://docs.midnight.network/tokens/overview), [divulgación explícita en Compact](https://docs.midnight.network/compact/explicit_disclosure), [generación local de pruebas](https://docs.midnight.network/guides/local-proving), [Glacier Drop](https://midnight.network/blog/glacier-drop-claim-portal-now-open) y [Kachina (artículo)](https://eprint.iacr.org/2020/543)

---

[← Índice](../../README.es.md) · [Anterior: Sección 12: referencias de otras redes descentralizadas](10-referencias-otras-redes.md)
