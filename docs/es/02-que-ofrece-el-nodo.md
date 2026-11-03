[← Índice](../../README.es.md) · [Anterior: Secciones 1-2: situación actual, quién depende del Gateway, exploradores y Olympia](01-situacion-actual.md) · [Siguiente: Secciones 4-5: las seis estrategias evaluadas y su comparativa con datos](03-estrategias.md)

# 3. Qué ofrece el nodo hoy (comprobado en `babylon-node`: rama `develop` y versión `v1.4.0.0`)

## 3.1 Las APIs del nodo

- **Core API (puerto 3333):**
  - `/lts/transaction/{construction, submit, status}`
  - `/lts/state/{account-all-fungible-resource-balances, account-fungible-resource-balance, account-deposit-behaviour}`
  - `/lts/stream/{transaction-outcomes, account-transaction-outcomes}`
  - `/status/{network-configuration, network-status}`
  - `/mempool/*`
  - `/transaction/{parse, submit, status, receipt, call-preview, preview, preview-v2}`
  - `/stream/{transactions, proofs}`
  - `/state/{consensus-manager, account, component, validator, access-controller, resource, non-fungible, package}`
- **Engine State API (puerto 3336):**
  - `/entity/info`, `/blueprint/info`
  - `/object/field`, `/object/collection/{iterator, entry}`
  - `/kv-store/{iterator, entry}`
  - `/object/attached-modules/{metadata/iterator, metadata/entry, role-assignment, royalty}`
  - `/extra/entity-search` (filtros `Blueprint`, `SystemType`, `EntityType`)
- **Mesh API (Rosetta):** estaba desactivada por defecto y **la versión `v1.4.0.0` (sep-2026) la elimina**. Servía a exchanges, no a dApps.
- **Exposición:** todas escuchan en `127.0.0.1` y no tienen CORS (`RadixNodeModule.java:127-131`).
- **Flags:**

  | Flag | Valor por defecto | Para qué sirve |
  |---|---|---|
  | `db.account_change_index.enable` | `true` | Historial por cuenta, solo fungibles |
  | `db.historical_substate_values.enable` | `false` | Lecturas históricas |
  | `state_hash_tree.state_version_history_length` | `60000` | Tamaño de la ventana histórica |
  | `db.entity_listing_indices.enable` | — | `/extra/entity-search` |
  | `api.core.flags.enable_unbounded_endpoints` | `true` | Endpoints de coste no acotado |

## 3.2 Correspondencia con el Gateway

| Capacidad | Con el nodo actual |
|---|---|
| Época, red, construcción de transacciones | ✅ `/status/network-status`, `/lts/transaction/construction` |
| Preview v1/v2 (el Gateway solo lo reenvía) | ✅ `/transaction/preview`, `/transaction/preview-v2` |
| Envío y estado | ✅ `/lts/transaction/{submit, status}`, con traducción de estados. Los estados LTS son `CommittedSuccess`, `CommittedFailure`, `PermanentRejection`, `InMempool`, `NotSeen`, `FateUncertain` y `FateUncertainButLikelyRejection`. |
| Saldos fungibles | ✅ `/lts/state/account-all-fungible-resource-balances` |
| NFT de una cuenta y sus datos | ✅ con 2-3 llamadas: colección `resource_vaults` de la cuenta, contenido del vault y `/state/non-fungible` |
| Estado de componentes, KV-stores, metadatos, roles | ✅ Engine State API |
| Validadores, LSU, pools | ✅ `/state/validator`, `/state/consensus-manager`, `/state/component`. El valor de canje se calcula en el cliente. |
| Prevalidación de depósito | ✅ `/lts/state/account-deposit-behaviour` |
| Resultado de una transacción | ✅ `/transaction/receipt` |
| **N1** Estado de un subintent | ❌ No hay endpoint, pero el nodo **ya lo indexa**: column family `finalized_subintent_hashes`, en `state-manager/src/store/column_families.rs:154-162` y `rocks_db.rs:705-708` |
| **N2** Ubicación de un NFT (qué vault o cuenta tiene el id X) | ❌ |
| **N3** Historial filtrado por entidad, con cambios de NFT, clase de manifiesto y eventos | ⚠️ Solo `/lts/stream/account-transaction-outcomes`, que da cambios fungibles de cuentas |
| **N4** Entidades que usan un badge en sus reglas | ❌ |
| **N5** Estado histórico de cualquier antigüedad | ⚠️ Solo ≈ 4-5 h, y con un flag desactivado por defecto |
| `last_updated_at_state_version` de los metadatos | ❌ Lo usa la recuperación de cuentas |
| Lecturas por lotes (20 entidades por llamada) | ❌ Hay que hacer una llamada por entidad |
| Uptime de validadores, titulares de un recurso, `touched-at` de lockers | ❌ |
| **N6** Acceso público seguro (CORS, solo lectura, rate limit, TLS) | ❌ |
| Reenvío de transacciones pendientes | ❌ Lo hace el Gateway. Con el nodo pasa a ser cosa del cliente: guardar el payload y reenviar con backoff mientras la época sea válida. |

---

[← Índice](../../README.es.md) · [Anterior: Secciones 1-2: situación actual, quién depende del Gateway, exploradores y Olympia](01-situacion-actual.md) · [Siguiente: Secciones 4-5: las seis estrategias evaluadas y su comparativa con datos](03-estrategias.md)
