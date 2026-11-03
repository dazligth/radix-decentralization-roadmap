[← Índice](../../README.es.md) · [Anterior: Alcance y capítulos 1-2: situación actual, quién depende del Gateway, exploradores y Olympia](01-02-situacion-actual.md) · [Siguiente: Capítulos 4-5: las seis estrategias evaluadas y su comparativa](04-05-estrategias.md)

<a id="s3"></a>

# 3. Qué ofrece el nodo hoy (comprobado en `babylon-node`: rama `develop` y versión `v1.4.0.0`)

<a id="s3-1"></a>

## 3.1 Las APIs del nodo

- **Core API (puerto 3333):**
  - `/lts/transaction/{construction, submit, status}`
  - `/lts/state/{account-all-fungible-resource-balances, account-fungible-resource-balance, account-deposit-behaviour}`
  - `/lts/stream/{transaction-outcomes, account-transaction-outcomes}`
  - `/status/{network-configuration, network-status, scenarios}`
  - `/mempool/*`
  - `/transaction/{parse, submit, status, receipt, call-preview, preview, preview-v2}`
  - `/stream/{transactions, proofs}`
  - `/state/{consensus-manager, account, component, validator, access-controller, resource, non-fungible, package}`
- **Engine State API (puerto 3336):**
  - `/entity/info`, `/blueprint/info`, `/entity/schema/entry`
  - `/object/field`, `/object/collection/{iterator, entry}`
  - `/kv-store/{iterator, entry}`
  - `/object/attached-modules/{metadata/iterator, metadata/entry, role-assignment, royalty}`
  - `/extra/entity-search` (filtros `Blueprint`, `SystemType`, `EntityType`)
- **Mesh API (Rosetta):** estaba desactivada por defecto y **la versión `v1.4.0.0` (sep-2026) la elimina**. Servía a exchanges, no a dApps.
- **Exposición:** todas escuchan por defecto en `127.0.0.1` y no tienen CORS (`RadixNodeModule.java:127-131`); dentro del contenedor, `babylon-nodecli` expone la Engine State API en `0.0.0.0` (`CoreDockerConfig.py:29`).
- **Referencias de línea:** salvo que se indique, corresponden a la rama `develop`.
- **Flags:**

  | Flag | Valor por defecto | Para qué sirve |
  |---|---|---|
  | `db.account_change_index.enable` | `true` | Historial por cuenta, solo fungibles |
  | `db.historical_substate_values.enable` | `false` | Lecturas históricas |
  | `state_hash_tree.state_version_history_length` | `60000` | Tamaño de la ventana histórica |
  | `db.entity_listing_indices.enable` | `false` (en el nodo Java) | `/extra/entity-search` |
  | `api.core.flags.enable_unbounded_endpoints` | `true` | Endpoints de coste no acotado |

<a id="s3-2"></a>

## 3.2 Correspondencia con el Gateway

| Capacidad | Con el nodo actual |
|---|---|
| Época, red, construcción de transacciones | Sí: `/status/network-status`, `/lts/transaction/construction` |
| Preview v1/v2 (el Gateway solo lo reenvía) | Sí: `/transaction/preview`, `/transaction/preview-v2` |
| Envío y estado | Sí: `/lts/transaction/{submit, status}`, con traducción de estados. Los estados LTS son `CommittedSuccess`, `CommittedFailure`, `PermanentRejection`, `InMempool`, `NotSeen`, `FateUncertain` y `FateUncertainButLikelyRejection`. |
| Saldos fungibles | Sí: `/lts/state/account-all-fungible-resource-balances` |
| NFT de una cuenta y sus datos | Sí: con 2-3 llamadas: colección `resource_vaults` de la cuenta, contenido del vault y `/state/non-fungible` |
| Estado de componentes, KV-stores, metadatos, roles | Sí: Engine State API |
| Validadores, LSU, pools | Sí: `/state/validator`, `/state/consensus-manager`, `/state/component`. El valor de canje se calcula en el cliente. |
| Prevalidación de depósito | Sí: `/lts/state/account-deposit-behaviour` |
| Resultado de una transacción | Sí: `/transaction/receipt` |
| **N1** Estado de un subintent | No: no hay endpoint, pero el nodo **ya lo indexa**: column family `finalized_subintent_hashes`, en `state-manager/src/store/column_families.rs:154-162` y `rocks_db.rs:705-708` |
| **N2** Ubicación de un NFT (qué vault o cuenta tiene el id X) | No |
| **N3** Historial filtrado por entidad, con cambios de NFT, clase de manifiesto y eventos | En parte: solo `/lts/stream/account-transaction-outcomes`, que da cambios fungibles de cuentas |
| **N4** Entidades que usan un badge en sus reglas | No |
| **N5** Estado histórico de cualquier antigüedad | En parte: solo ≈ 4-5 h, y con un flag desactivado por defecto |
| **N5b** `last_updated_at_state_version` de los metadatos | No: lo usa la recuperación de cuentas |
| **N7** Lecturas por lotes (20 entidades por llamada) | No: hay que hacer una llamada por entidad |
| `touched-at` de lockers y uptime de validadores (**N8-N9**); titulares de un recurso (índice h del indexador ligero) | No |
| **N6** Acceso público seguro (CORS, solo lectura, límite de peticiones, TLS) | No |
| Reenvío de transacciones pendientes | No: lo hace el Gateway. Con el nodo pasa a ser responsabilidad del cliente: guardar la transacción notarizada y reenviar con espera exponencial mientras la época sea válida. |

---

[← Índice](../../README.es.md) · [Anterior: Alcance y capítulos 1-2: situación actual, quién depende del Gateway, exploradores y Olympia](01-02-situacion-actual.md) · [Siguiente: Capítulos 4-5: las seis estrategias evaluadas y su comparativa](04-05-estrategias.md)
