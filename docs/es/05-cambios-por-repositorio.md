[← Índice](../../README.es.md) · [Anterior: Sección 6: solución recomendada, almacenamiento y solución ideal a largo plazo](04-solucion-recomendada.md) · [Siguiente: Sección 8: hoja de ruta](06-hoja-de-ruta.md)

# 7. Cambios por repositorio

Prioridad: **P0** imprescindible · **P1** necesario para todos los casos de uso · **P2** mejora.
Esfuerzo: **S** días · **M** 1-3 semanas · **L** más de un mes.

## 7.1 Nuevo repositorio: `radix-lite-indexer` (estrategia E3)

**Componentes:**

- **Ingesta (P0/M):**
  - `/core/stream/transactions` con recibos y eventos;
  - cursor persistente, tolerante a reinicios;
  - filtrar los cambios de ronda para los índices de entidad.
- **Índices (P0/M):**

  | Índice | Para qué |
  |---|---|
  | (a) `entity → [state_version]`, con cambios fungibles y de NFT, clase de manifiesto, emisores de eventos y recursos del manifiesto | Historial |
  | (b) `(resource, nf_id) → vault → entidad global` | Ubicación de NFT |
  | (c) `subintent_hash → state_version` | Pre-autorizaciones (o leerlo del nodo cuando exista N1) |
  | (d) `badge → entidades` | Reglas de rol, para MFA |
  | (e) `(entidad, clave de metadato) → last_updated_state_version` | Recuperación de cuentas |
  | (f) `locker → cuenta → último toque` | Lockers |
  | (g) emisiones por validador y época | Uptime |
  | (h) `resource → (entidad, saldo)` | Titulares de un recurso (exploradores) |
  | (i) agregados por hora y día: transacciones, comisiones, cuentas activas, TPS | Estadísticas (exploradores) |
  | (j) texto de `name`, `symbol` y `tags` → entidad, más `timestamp → state_version` | Búsqueda y navegación por fecha (exploradores) |

- **API de lectura en el formato actual de las apps (P0/L).** Hay que implementar estas rutas, con el mismo JSON que esperan hoy los clientes:
  - `status/gateway-status`, `status/network-configuration`;
  - `transaction/{construction, preview, preview-v2, submit, status, subintent-status, committed-details, account-deposit-pre-validation}`;
  - `state/entity/details` (con `aggregation_level`, `opt_ins` y `explicit_metadata`);
  - `state/entity/page/{fungibles, non-fungibles, non-fungible-vaults, non-fungible-vault/ids, fungible-vaults, metadata}`;
  - `state/non-fungible/{data, location, ids}`;
  - `state/key-value-store/{keys, data}`;
  - `state/validators/list`;
  - `state/account/page/{resource-preferences, authorized-depositors}`;
  - `state/account-lockers/touched-at`, `state/account-locker/page/vaults`;
  - `stream/transactions` (con los filtros que usan los wallets);
  - `extensions/entities-by-role-requirement/lookup`;
  - `extensions/resource-holders/page` (exploradores);
  - `statistics/validators/uptime`;
  - endpoints propios de estadísticas y búsqueda, para exploradores (índices i y j).

  Las lecturas de estado actual se resuelven en vivo contra el nodo. Los modelos de referencia están en `babylon-gateway/src/RadixDlt.NetworkGateway.GatewayApi`.
- **Reenvío de transacciones (P0/S).** Copiar `PendingTransactionResubmissionService` (`babylon-gateway`, `PendingTransactionResubmissionService.cs:123-307`; valores por defecto en `MempoolOptions.cs:74-99`).
- **Coherencia (P0/S).** Fijar la `state_version` de cada petición paginada y rechazar la petición si el nodo ha avanzado más allá del cursor.
- **Modo público (P0/S):**
  - CORS;
  - rate limit por IP;
  - límites de página;
  - TLS;
  - lista blanca de endpoints;
  - imagen Docker y plantilla para `babylon-nodecli`.
- **Snapshots (P2/M).** Exportar e importar índices firmados para no tener que sincronizar desde la génesis.
- **Conformidad (P0/M).** Comparar respuestas contra el Gateway público con un muestreo continuo, antes de que se apague.

## 7.2 `radixdlt/babylon-node` (estrategia E2)

| # | Cambio | P / Esfuerzo |
|---|---|---|
| N1 | `/core/transaction/subintent-status` (y versión LTS), leyendo `finalized_subintent_hashes`, que ya existe | P0 / S |
| N6 | **Modo de lectura pública:** `api.public.*` con lista blanca, CORS, rate limit, límites de página, `enable_unbounded_endpoints=false` forzado | P0 / M |
| N5b | `last_updated_at_state_version` en `/object/attached-modules/metadata/{entry, iterator}` | P1 / S |
| N2 | Índice opcional `db.non_fungible_location_index.enable` y `/core/state/non-fungible-location` | P1 / M |
| N3 | Índice de cambios por entidad global (no solo cuentas), con cambios de NFT, clase de manifiesto y filtros | P1 / L |
| N4 | Índice inverso de reglas de rol y `/core/extra/entities-by-role-requirement` | P1 / M |
| N5 | Nodo de archivo: `state_version_history_length` sin límite práctico, `LedgerStateSelector::ByTimestamp` (ya está previsto en el esquema, `engine-state-api-schema.yaml:592`) y `at_ledger_state` en la LTS | P1 / M |
| N7 | Endpoints por lotes (metadatos y estado de varias entidades a la vez) | P2 / S |
| N8-N9 | Toques de lockers y emisiones de validadores | P2 / S |
| D | Convención de metadatos del validador `api_endpoints` (lista de URLs con capacidades) y documentación para operadores | P0 / S |

Todos los índices deben ser **flags opcionales**, como los que ya existen, para no penalizar a los validadores que no quieran servir lecturas.

## 7.3 `radixdlt/babylon-nodecli` (despliegue de nodos)

- **P0/S:** añadir un perfil «nodo de lectura pública» con:
  - Engine State API activada (hoy el asistente propone `N`, `node-runner-cli/utils/Prompts.py:252-257`);
  - `db.historical_substate_values` y `db.entity_listing_indices` según el perfil;
  - el sidecar `radix-lite-indexer`;
  - proxy TLS.
- **P1/S:** asistente para publicar `api_endpoints` en los metadatos del validador.

## 7.4 `radixdlt/babylon-gateway`

- **Sin cambios para el objetivo.** Se usa como **especificación** de la API, de la lógica de reenvío y de la clasificación de errores al enviar (`TransactionSubmitter.cs:115-230`).
- **P2:** separar el contrato OpenAPI en un paquete independiente, versionado y compartido con `radix-lite-indexer`, para garantizar la compatibilidad.

## 7.5 `radixdlt/sargon` (núcleo de los wallets)

1. **P0/M — Trait `LedgerBackend`.**
   - `GatewayClient` (`crates/gateway/client-and-api/src/client/gateway_client.rs:11-40`) es una struct concreta. Hay que extraer un trait con los métodos de `methods/state_methods.rs` y `methods/transaction_methods.rs`, y devolverlo desde `SargonOS::gateway_client()` (`system/os/os/src/sargon_os_gateway_client.rs:8-17`).
   - Hay cinco sitios que crean su propio cliente y deben pasar a usar el trait:
     - `transaction_intent_builder.rs:58`
     - `xrd_balances_fetcher.rs:46`
     - `access_controller_state_repository_client.rs:27`
     - `radix-name-service/src/service.rs:30`
     - `app/home-cards/src/manager.rs:45`
2. **P0/S — Varios endpoints con failover y descubrimiento en la cadena.**
   - `SavedGateways` (`crates/profile/models/gateway/src/saved_gateways.rs:16-24`) admite una lista, pero solo usa el actual.
   - Hay que añadir failover y comprobación de red.
   - Añadir el descubrimiento (D): leer `api_endpoints` de los validadores activos con `/state/consensus-manager` y sus metadatos.
3. **P0/M — Reenvío local.** Hoy `sargon_os_transaction_status.rs:38-105` solo consulta el estado. Hay que guardar el payload notarizado y reenviarlo con backoff hasta que se confirme o caduque su época. Así la transacción no se pierde si el punto de acceso cambia o se reinicia.
4. **P1/M — `NodeBackend` nativo** (Core y Engine State API, con DTOs generados de los YAML del nodo), para cuando exista E2.
   - Implica un cambio de esquema del Profile: `Gateway { network, url }` (`gateway.rs:21-27`) pasa a llevar `kind: GatewayApi | NodeApi`.
   - Hay que coordinarlo con iOS y Android y prever una migración.
5. **P1/S — Servicios fijos.** Hacer configurable `nft-pricing-service.radixdlt.com` (`non_fungibles_prices_service/remote_fetcher.rs:4-5`).

## 7.6 `babylon-wallet-android` (y los forks `-iroh`, `wallet-pr-*`)

- **P0/S.** `BaseUrlInterceptor` (`di/HttpModule.kt:128-144`) solo reescribe el esquema y el host, y **pierde el puerto y la ruta**. Un punto de acceso como `https://nodo:8443/gw` deja de funcionar al cambiar de Gateway.
- **P0/S.** `HomeCardsManager` usa `Gateway.mainnet` fijo (`di/ApplicationModule.kt:74-87`); debe usar el punto de acceso actual.
- **P0/S.** En Ajustes → Gateways (`GatewaysViewModel.kt:105-133`), mostrar la lista descubierta en la cadena y probar varios puntos de acceso.
- **P1/M.** Cuando Sargon tenga `LedgerBackend`, crear la implementación de nodo de los repositorios de `di/DataModule.kt:48-137`:
  - `StateRepository`
  - `StreamRepository`
  - `TransactionRepository`
  - `NetworkInfoRepository`
  - `AccountLockersRepository`
  - `ResolveAccountsLedgerStateRepository`

  La paginación y la agregación están en `data/gateway/extensions/StateEntityDetailsApiHelper.kt`.
- **P1/S.** Hacer configurables con fallback:
  - `image-service.radixdlt.com` (`app/build.gradle:70,112`, `Thumbnail.kt:523`), cargando la imagen original si no está disponible;
  - `dapps-list.radixdlt.com` (`DAppDirectoryRepository.kt:81-87`);
  - los enlaces al dashboard (`NetworkIdExtensions.kt:50-56`).

## 7.7 `radixdlt/babylon-wallet-ios`

Es simétrico al de Android. Los clientes de DI (`DependencyKey`) permiten sustituir la `liveValue`:

- **P0/S:**
  - failover y descubrimiento en `GatewaysClient` y `NetworkSwitchingClient` (`NetworkSwitchingClient+Live.swift:30`).
- **P1/M:**
  - `GatewayAPIClient` (`Clients/GatewayAPI/GatewayAPIClient/GatewayAPIClient+Interface.swift:5`, con los endpoints en `+Live.swift:122-217`);
  - `OnLedgerEntitiesClient`;
  - `TransactionHistoryClient`;
  - `AccountLockersClient`;
  - `TransactionClient`.
- **P1/S:**
  - `image-service` (`URLFormatterClient+Live.swift:20-22`);
  - `dapps-list` (`DAppsDirectoryClient+Live.swift:4`);
  - dashboard (`Radix+Dashboard.swift:48-102`).

## 7.8 `radix-dapp-toolkit`

- **P0/S.** Aceptar **una lista** de puntos de acceso con failover en `gatewayBaseUrl`, o descubrimiento en la cadena. La URL por defecto está fija en `modules/gateway/types.ts:60-97`.
- **P1/S.** Proveedor `NodeGatewayApiService` para la API nativa del nodo (E2). Se inyecta por `GatewayModule({ providers: { gatewayApiService } })` (`gateway.module.ts:16-26`) y solo necesita tres funciones:
  - `getTransactionStatus`
  - `getSubintentStatus`
  - `getEntityMetadataPage`
- **P1/S.** El relé de Radix Connect está fijo en `modules/wallet-request/wallet-request.ts:159`; hay que añadir la opción `relayUrl`.
- **P2/S.** Enlaces fijos (`connect-button.module.ts:305,307`, `packages/connect-button/src/config.ts:3`).

## 7.9 `radixdlt/rola` (login de todas las dApps)

- **P0/S.** `GatewayService` (`typescript/src/gateway.ts:14-38`) usa por defecto `RadixNetworkConfigById[networkId].gatewayUrl`. Hay que:
  - aceptar una lista o descubrimiento;
  - añadir la opción `ownerKeysReader` para leer `owner_keys` con la Engine State API (`/object/attached-modules/metadata/entry`).
- **P0/S.** Igual en el paquete `python/`.
- **P1/S.** Si no se puede consultar el ledger y la cuenta es virtual (sin `owner_keys`), la derivación de la dirección ya basta (`rola.ts:85`). Hay que documentar que ese caso no necesita red.

## 7.10 `connector-extension` y `connector-extension-iroh`

- **P1/S.** `src/chrome/background/create-gateway-module.ts:1-13` usa la URL fija de `RadixNetworkConfigById`. Hay que añadir la URL o la lista de puntos de acceso a la página de opciones (`src/options/index.ts:20-23`).
- **P1/S.** Con un servidor de señalización propio, la extensión sigue usando los TURN de RDX con credenciales fijas (`src/config.ts:44-53, 97-101`). Hay que permitir configurar el TURN.
- **P2/S.** Explorador configurable (`src/utils/explorer.ts:6`).

## 7.11 Transporte de Radix Connect: `radix-connect-iroh`, `radix-connect-webrtc` y los repos oficiales `radix-connect-relay` y `signaling-server`

No usan el Gateway, pero son infraestructura P1 que también depende de un único operador:

- **`radix-connect-iroh`:** ya admite relé propio (`Relays::Custom`, `crates/radix-connect-iroh/src/config.rs:15-18`). P2: varios relés propios y pkarr propio.
- **`radix-connect-webrtc`:** P2/S. Permitir **sustituir** los STUN de Google (`src/connector/connector-client.ts:115-131`), no solo añadir otros.
- **`radix-connect-relay` y `signaling-server`:** P1/S. Publicar imágenes y guías para alojarlos uno mismo y anunciarlos con el mismo mecanismo de descubrimiento (D).

## 7.12 `radixdlt-rust-sdk`

- **P0/M.** Crear un crate ligero `ledger-api` con el trait `LedgerBackend` y dos implementaciones: API del Gateway y API del nodo.
- **P0/S.** Varios cambios en `gateway-tx`:
  - `Gateway` (`crates/gateway-tx/src/lib.rs:145`) implementa el trait.
  - **Fallo:** `fungible_balance` (`:213-228`) solo lee la primera página, así que un saldo en la página 2 aparece como 0.
  - Las URLs fijas (`:38-40`) pasan a ser una lista con descubrimiento.
- **P1/S.** `connect-iroh`: `Wallet.gateway` (`crates/connect-iroh/src/protocol.rs:249`) pasa a ser `Box<dyn LedgerBackend>`.
- **P1/S.** `connector-mcp/src/gateway.rs:10-15` tiene las URLs fijas.
- **P1/S.** `rola`: implementar la lectura de `owner_keys` (`crates/rola/src/lib.rs:8-9`).
- **P1/S, seguridad.** `connect-webrtc/src/connector.rs:56-70` tiene credenciales TURN en el código.

## 7.13 `consultation_v3` (idéntico a `consultation_v2`)

- **P0/S.** Configurar el punto de acceso:
  - `packages/shared/src/gateway.ts:10-42`;
  - `envVars.ts:8-15`, que no tiene `GATEWAY_URL`, así que el frontend usa siempre el Gateway público;
  - la RDT de `apps/consultation/src/lib/dappToolkit.ts:46-50`.
- **P0/M.** **Recuento sin historia arbitraria (E6):**
  - Manejar `TemperatureCheckCreatedEvent` y `ProposalCreatedEvent`, que hoy se ignoran (`governanceEvents.ts:66-69`).
  - En ese momento, guardar en la base de datos el snapshot completo: saldos de XRD, LSU, LSULP y pool units de los participantes, tasas de canje y estado de los pools.
  - Para los fungibles también vale el «delta inverso»: saldo actual menos `fungible_entity_balance_changes` de `/lts/stream/account-transaction-outcomes`.
  - Hoy `voteCalculation.ts:178-216` y `scripts/tally.ts` dependen de `at_ledger_state` antiguo.
- **P1/S, fallo actual.** `accountBalanceState.ts:110-124` lee los validadores en el estado actual, no en el del snapshot.
- **P1/M.** Si se usa la API nativa del nodo, `poll.ts:30-45` pierde `affected_global_entities_filter`: hay que leer todo el stream y filtrar por emisor.
- **P2.** En el blueprint (`scrypto/`), emitir en `Created` los parámetros del snapshot y valorar recuentos en la propia cadena. Esto afecta a las reglas de `governance-framework` sobre «snapshot al abrir» y repeticiones.

## 7.14 `wallet-agent-ai`

- **P0/S.** URLs fijas en `src/network-config.ts:14-22`; la red se deduce de la URL (`:33-36`, `weft.ts:211-225`).
- **P0/S, fallo actual.** Comprueba `"Rejected"` / `"PermanentlyRejected"` (`src/core/RadixClient.ts:122`).
- **P1/M.** Crear la interfaz `LedgerReader` para las ~25 llamadas `fetch`.
- **P1/S.** `waitForOwnerApproval` (`AgentWallet.ts:697-722`) debe vigilar el estado del componente en vez de filtrar el stream.
- **P2.** La API de Astrolescent debe admitir una instancia propia.

## 7.15 `Radix-Desktop-Tool`

- La URL ya se puede cambiar en la interfaz (`src/views/LocalSettings.vue`). Con E3 funciona sin cambios.
- **P2/M.** Para la API nativa del nodo hay que hacer un fork de `@atlantis-l/radix-tool`.
- **P1.** El historial (`HistoryCheck.vue:999`) depende del índice (a) de E3, o de N3.

## 7.16 Exploradores de transacciones: Radix Dashboard, exploradores comunitarios y un nuevo explorador de referencia

- **Radix Dashboard (código no público).** Usa la API del Gateway, así que funciona con E3 si se configura su URL de API y E3 incluye los índices (h), (i) y (j). Hay que confirmarlo con quien lo opere. **P0:** pedir que se publique su código o que haya un despliegue comunitario.
- **Exploradores comunitarios.** No he podido verificar su backend. Si usan la API del Gateway, les sirve E3 cambiando la URL. Si tienen su propio indexador, ya no dependen del Gateway.
- **P0/L — Nuevo repositorio `radix-explorer` (explorador de referencia, código abierto, sitio estático).** Así ninguna entidad controla el único explorador.
  - **Fuente de datos:** la API de lectura de `radix-lite-indexer` y de los nodos, con la lista de puntos de acceso descubierta en la cadena (metadatos `api_endpoints` de los validadores) y failover.
  - **Páginas mínimas:**
    - transacción (recibo, manifiesto, eventos, cambios de saldo, comisión);
    - cuenta, componente, recurso y paquete;
    - NFT, con su ubicación;
    - titulares de un recurso;
    - validadores;
    - últimas transacciones;
    - estadísticas;
    - búsqueda.
  - **Despliegue:** sitio estático publicable en cualquier hosting o en IPFS, sin servidor propio.
  - **Verificación:** mostrar de qué nodo viene cada dato y, opcionalmente, comparar dos puntos de acceso (`state_version` y hash del acumulador).
- **Developer Console (código no público).** Solo necesita preview, envío, estado y detalles de entidad, así que funciona con E3 o con el nodo. **P1:** publicar su código o tener un sustituto comunitario.

## 7.17 Repos que no necesitan cambios

- **`olympia-wallet`:** su uso actual (exportar cuentas) es fuera de línea. **P2/S:** cuando exista el archivo de Olympia (2.2), apuntar a él en vez de a `olympia-gateway.radixdlt.com` (`src/helpers/network.ts:14-39`).
- **`radixdlt/explorer`** (explorador de Olympia): **P1/S:** servir el historial desde el archivo estático de Olympia en vez de desde el Gateway de Olympia; o integrarlo en el explorador de referencia (7.16).
- **`wallet-compatible-derivation`:** derivación de claves fuera de línea, sin red.
- **`governance-framework`:** solo lo dicho en 7.13.
- **`reference/hyperscale-rs`:**
  - Otra red, no compatible con Babylon.
  - Su RPC (`crates/production/src/rpc/routes.rs:13-35`) no tiene lecturas de estado.
  - Según la Foundation, con Xi'an habría que construir un acceso nuevo de todos modos. La interfaz `LedgerBackend` de los clientes es donde se añadiría esa tercera implementación.

---

[← Índice](../../README.es.md) · [Anterior: Sección 6: solución recomendada, almacenamiento y solución ideal a largo plazo](04-solucion-recomendada.md) · [Siguiente: Sección 8: hoja de ruta](06-hoja-de-ruta.md)
