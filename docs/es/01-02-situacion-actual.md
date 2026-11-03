[← Índice](../../README.es.md) · [Siguiente: Capítulo 3: qué ofrece hoy el nodo y qué le falta](03-que-ofrece-el-nodo.md)

<a id="alcance"></a>

# Alcance

**Objetivo.** Que la red Babylon de Radix sirva **a cualquier usuario y en cualquier caso de uso** sin depender del Babylon Network Gateway: el servicio REST + DataAggregator + PostgreSQL que hoy publica `mainnet.radixdlt.com`. La lista de casos de uso es la siguiente:

- wallets;
- dApps;
- inicio de sesión con ROLA;
- exploradores;
- gobernanza;
- agentes;
- herramientas de desarrollo.

**Principio de diseño: todo en la capa 1.** El objetivo de Radix es escalar y ofrecer todas sus funciones en la capa 1, sin recurrir a soluciones de capa 2 (rollups, cadenas laterales o cadenas asociadas). Todas las propuestas de este informe respetan ese principio:

- El indexador ligero, los nodos de lectura y el descubrimiento de puntos de acceso son infraestructura para **leer** el estado de la capa 1. No ejecutan transacciones ni mantienen un estado propio en el que la red tenga que confiar.
- La poda, los snapshots verificados, la migración a Rust y las lecturas verificables son cambios en el software de los nodos de la capa 1.
- Los incentivos para servir lecturas se pagarían en la propia capa 1 (por ejemplo, mediante un componente en el ledger), según decida la gobernanza.
- Hyperscale escala **en la propia capa 1**, mediante fragmentación con composición atómica, lo que es coherente con ese objetivo.
- El puente de la estrategia M2 (capítulo [11](11-evolucion-hyperscale.md#s11)) solo conectaría dos redes de capa 1 durante la transición, y es opcional.

Las referencias a mecanismos de capa 2 o de cadenas asociadas en otras redes (capítulo [12](12-referencias-otras-redes.md#s12)) se incluyen solo como contraste, no como modelo.

**Repositorios analizados.**

- **En local:**
  - `babylon-wallet-android` y sus forks;
  - `radix-dapp-toolkit`;
  - `connector-extension` y `connector-extension-iroh`;
  - `radix-connect-iroh` y `radix-connect-webrtc`;
  - `radixdlt-rust-sdk`;
  - `consultation_v2` y `consultation_v3`;
  - `governance-framework`;
  - `radixdlt-scrypto` (rama `develop`, commit `a405d7ae48`, ene-2026): motor y transacciones, para el capítulo [13](13-xian-con-scrypto.md#s13);
  - `wallet-agent-ai`;
  - `Radix-Desktop-Tool`;
  - `olympia-wallet`;
  - `wallet-compatible-derivation`;
  - `reference/hyperscale-rs`.
- **Clonados de GitHub** (no estaban en el equipo):
  - `radixdlt/babylon-node`;
  - `radixdlt/babylon-gateway`;
  - `radixdlt/sargon`;
  - `radixdlt/babylon-wallet-ios`;
  - `radixdlt/rola`;
  - `radixdlt/babylon-nodecli`;
  - `radixdlt/explorer`, el explorador de Olympia (ver [2.2](#s2-2)).
- **No públicos:** el código del Radix Dashboard y de la Developer Console no está en la organización `radixdlt` de GitHub. Se tratan como clientes de caja negra.

## Infraestructura actual

```mermaid
flowchart TB
    subgraph APPS["Usuarios y aplicaciones"]
        W["Wallets iOS y Android (Sargon)<br/>y Connector Extension"]
        DAPP["dApps con RDT y backends<br/>con inicio de sesión ROLA"]
        EXP["Radix Dashboard, exploradores<br/>y Developer Console"]
        OLYW["olympia-wallet"]
    end
    subgraph CENTRAL["Servicios centrales: un solo operador"]
        AUX["Servicios de la wallet<br/>señalización WebRTC, TURN,<br/>Radix Connect Relay, precios,<br/>imágenes, directorio de dApps"]
        GW["<b>Babylon Gateway</b><br/>Gateway API + DataAggregator<br/>(una sola instancia)<br/>PostgreSQL ~2,5 TB, +60 GB/mes"]
        OGW["Gateway de Olympia<br/>historial 2021-2023"]
    end
    subgraph NET["Red Radix"]
        FN[("Nodos completos<br/>Java + Rust, 16 GB RAM")]
        VAL[("~100 validadores<br/>Java + Rust, 16 GB RAM")]
    end
    W -.-> AUX
    DAPP -.-> AUX
    W --> GW
    DAPP --> GW
    EXP --> GW
    OLYW --> OGW
    GW <-->|"stream de transacciones,<br/>preview y envío"| FN
    FN --- VAL
    classDef central fill:#ffd6d6,stroke:#c00,color:#000
    class GW,AUX,OGW central
```

En rojo, los servicios que dependen de un solo operador.

<a id="s1"></a>

# 1. Situación actual: datos

<a id="s1-1"></a>

## 1.1 El Gateway y su financiación

| Dato | Valor | Fuente |
|---|---|---|
| Tamaño de la base de datos del Gateway (feb. 2026) | **~2 TB, crecimiento ~60 GB/mes** | RFP de la Foundation, 3-feb-2026 |
| Tamaño estimado hoy, nov. 2026 | ~2,5 TB (2 TB + 9 meses × 60 GB) | cálculo |
| Requisitos del RFP | 99,9 % de disponibilidad, <1 s de latencia, despliegue blue/green | RFP |
| DataAggregator | **Una sola instancia**: las transacciones se procesan en orden | Foundation, radixtalk 2202, posts 15 y 22 |
| Operador actual | El antiguo equipo DevOps de la Foundation, como operación independiente | Blog, 28-abr-2026 |
| Financiación | **Hasta finales de diciembre de 2026** | Blog, 28-abr-2026 |
| Coste para un tercero | ~300-400 €/mes solo el Gateway; ~500 €/mes con relé y señalización | Propuesta comunitaria, radixtalk 2202, post 26 |
| Carga estimada | ~250 000 peticiones/hora (~69 por segundo) | Estimación comunitaria, radixtalk 2202, post 29 |
| Xi'an / Hyperscale | «requeriría construir un Gateway completamente nuevo»; el actual no es reutilizable | Foundation, radixtalk 2202, post 15 |
| Señalización WebRTC | ~6 000 conexiones simultáneas en pico | RFP |

<a id="s1-2"></a>

## 1.2 Mediciones propias de mainnet (3-nov-2026, con la API pública)

| Medida | Valor |
|---|---|
| `state_version` actual | 567 005 616 (época 355 117) |
| Ritmo de versiones de estado | **~3,6/s de media** desde junio de 2026; 4,4/s en la muestra de hoy. Cada ronda de consenso es una versión. |
| Transacciones de usuario/día (muestra de 100 transacciones en cada fecha) | nov-2023: 1,5 mil · mar-2024: 5,5 mil · jul-2024: 11,6 mil · nov-2024: 20,7 mil · **mar-2025: 34,7 mil (pico)** · jul-2025: 8,4 mil · nov-2025: 5,9 mil · mar-2026: 3,4 mil · jul-2026: 2,2 mil · oct-2026: 4,4 mil |
| Total estimado de transacciones de usuario desde la génesis | **~10-15 millones** (integración aproximada, ±50 %) |
| Peso de las transacciones de usuario | **~2-3 % de las versiones de estado** en todo el historial (~10-15 millones de ~567 millones); hoy, menos del 1 %. El resto (~97-98 %) son cambios de ronda y de época. |
| Ventana histórica del nodo | `state_version_history_length = 60000` versiones por defecto, **≈ 4-5 horas** a la media de 3,6 versiones/s (≈ 3,8 h al ritmo medido hoy) |

**Consecuencia clave.** Probablemente, casi todo el volumen del Gateway viene de guardar la historia completa de cada substate, incluidos los ~555 millones de cambios de ronda. (Es una deducción: no se ha publicado la composición de su base de datos.) Un índice que solo registre lo que necesitan los clientes trabaja con ~10-15 millones de transacciones de usuario. Es un problema **unas 40-55 veces más pequeño**.

<a id="s1-3"></a>

## 1.3 Qué muestran consultas reales al Gateway (3-nov-2026)

Para entender mejor qué hace el Gateway y qué habría que sustituir, se hicieron consultas a su API pública:

| Consulta | Resultado | Qué implica |
|---|---|---|
| Estado del recurso XRD en la versión 10 000 (`at_ledger_state`, 28-sep-2023) | Responde: suministro total de 12 660 434 555 XRD entonces, frente a 13 565 348 962 hoy | El Gateway sirve **historia completa desde la génesis**. El nodo solo conserva unas horas (sección [3.1](03-que-ofrece-el-nodo.md#s3-1)). Esa historia arbitraria es lo que más ocupa y lo que el indexador ligero no replica; quien la necesite tendría un nodo de archivo (secciones [6.2](06-solucion-recomendada.md#s6-2) y [9.6](09-nodos-ligeros.md#s9-6)). |
| Titulares de XRD (`/extensions/resource-holders/page`) | `total_count`: **435 605** | Da el orden de magnitud del índice de titulares (h). Con ~100 bytes por fila (supuesto), XRD ocuparía unos 44 MB: una fracción mínima del total estimado para el indexador ligero (10-50 GB). |
| Cabeceras de respuesta | `server: cloudflare`, `cf-cache-status: DYNAMIC` | El Gateway público está detrás de Cloudflare y sus respuestas no se sirven desde caché. Es otra dependencia central; los nodos de lectura necesitan su propia protección frente a ataques (N6). |
| `/status/gateway-status` y `/status/network-configuration` | Versión 1.10.6 (imagen v1.10.7); red `mainnet`, identificador 1 y direcciones conocidas | El indexador ligero puede obtener los mismos datos del nodo (`/status/network-configuration`). |

**Conclusión:** las funciones del Gateway se dividen en tres grupos, y los tres tienen sustituto:

1. estado actual → el propio nodo;
2. índices (historial por entidad, titulares, ubicación de NFT, búsqueda) → el indexador ligero;
3. historia arbitraria del estado → nodos de archivo, solo para quien la necesite.

Por eso el Gateway **se puede eliminar por completo**.

<a id="s2"></a>

# 2. Quién depende del Gateway y para qué

| Cliente | Endpoints del Gateway que usa | Para qué |
|---|---|---|
| **Sargon** (Rust, compartido por iOS y Android) | `transaction/{construction, preview, preview-v2, submit, status, subintent-status}`, `status/gateway-status`, `state/entity/details`, `state/entity/page/{fungibles, non-fungibles, non-fungible-vaults, non-fungible-vault/ids}`, `state/account/page/{resource-preferences, authorized-depositors}`, `state/non-fungible/{location, data}`, `extensions/entities-by-role-requirement/lookup` | Revisión de transacciones y comisiones, envío y seguimiento, pre-autorizaciones, cuentas securificadas o borradas, MFA y shields, borrado de cuenta, RNS, precio de NFT |
| **Wallet Android** (Kotlin, además de Sargon) | `state/entity/details`, `state/entity/page/{fungibles, non-fungibles, non-fungible-vault/ids, metadata}`, `state/non-fungible/data`, `state/account-lockers/touched-at`, `state/account-locker/page/vaults`, `transaction/{construction, account-deposit-pre-validation}`, `stream/transactions`, `status/gateway-status` | Saldos, NFT, pools, validadores, LSU, reclamaciones de stake, metadatos de dApps, recuperación de cuentas, lockers, prevalidación de depósitos, **historial** |
| **Wallet iOS** (Swift, además de Sargon) | Los mismos, más `state/entity/page/fungible-vaults` | Igual |
| **Radix dApp Toolkit** | `transaction/status`, `transaction/subintent-status`, `state/entity/page/metadata` | Seguir la transacción que envía el wallet; nombre de la dApp en el botón |
| **ROLA** (`radixdlt/rola`) | `state/entity/details` (metadato `owner_keys`) | **Inicio de sesión en cualquier dApp.** Se consulta en cada inicio de sesión (`typescript/src/rola.ts:49-87`, `gateway.ts:14-38`). Por defecto usa la URL pública. |
| **Connector Extension** | `transaction/status` (a través del `GatewayModule` de la RDT) | Notificación de transacción confirmada |
| **Exploradores de transacciones** (Radix Dashboard, código no público; exploradores comunitarios) | No verificable en el Dashboard; por sus funciones, prácticamente toda la API, incluidas las extensiones de **titulares de recursos**, ubicación de NFT, historial filtrado de **cualquier** entidad, uptime de validadores, KV-stores, paquetes, esquemas y `at_ledger_state` | Ver la sección [2.1](#s2-1) |
| **Developer Console** (código no público) | No verificable; por sus funciones, al menos preview, envío, estado y detalles de entidad | Despliegue de paquetes, manifiestos |
| **Consultation** (gobernanza) | `state/entity/details`, `state/key-value-store/{keys, data}`, `stream/transactions`, `transaction/committed-details`, `status/gateway-status`, saldos y NFT **con `at_ledger_state` histórico** | Propuestas, votos, recuento con snapshot |
| **radixdlt-rust-sdk** | `status/gateway-status`, `state/entity/page/fungibles`, `transaction/{status, committed-details, submit, construction, preview}`, `state/entity/details` | SDK genérico para firmar y enviar |
| **wallet-agent-ai** | ~25 llamadas: detalles de entidad, preview, envío, estado, stream, validadores, uptime | Agente con wallet propio |
| **Exchanges e integradores** | Según la especificación de la Core API, se recomienda que usen los endpoints `/lts/` del propio nodo (`core-api-schema.yaml:83-89`, v1.4.0.0) | Depósitos, retiradas y seguimiento de transacciones. Quien siga esa recomendación **no depende del Gateway**. Quien use el Gateway o la Mesh API (eliminada en v1.4.0.0) debe pasar a la LTS. |
| **Radix-Desktop-Tool** | Toda la API, mediante `@atlantis-l/radix-tool` | Herramientas de escritorio, historial |

<a id="s2-1"></a>

## 2.1 Exploradores de transacciones: requisito explícito

> [!IMPORTANT]
> - **¿Es posible?** Sí. Los exploradores pueden funcionar sin el Gateway.
> - **Dificultad:** Media.
> - **Esfuerzo y plazo:** ~1-2 personas-mes de índices + ~2-3 de explorador de referencia (estimación).

Un explorador es el cliente **más exigente**: necesita índices globales que ni el nodo ni los wallets necesitan. Cada función de un explorador tiene una fuente distinta: el propio nodo o un índice nuevo.

| Función del explorador | Cómo se sirve sin el Gateway |
|---|---|
| Buscar una transacción por su hash (intent hash) y ver su recibo, eventos, cambios de estado y comisiones | Sí, en el nodo: `/core/transaction/receipt` (por `intent_hash`) y `/core/lts/transaction/status`. Los cambios de saldo se calculan a partir del recibo. |
| Últimas transacciones y transacciones por época o ronda | Sí, en el nodo: `/core/stream/transactions` (orden ascendente). El orden descendente o desde una fecha necesita el índice `timestamp → state_version`. |
| Página de una cuenta, componente o recurso (estado actual, saldos, NFT, metadatos, roles, royalties) | Sí, en el nodo: Core y Engine State API |
| Paquetes: blueprints, código, esquemas | Sí, en el nodo: `/core/state/package`, `/blueprint/info`, `/entity/schema/entry` |
| KV-stores de un componente | Sí, en el nodo: `/kv-store/iterator` |
| **Historial de transacciones de cualquier entidad**, con filtros por tipo, recurso o evento | No: índice (a) de E3, o N3 |
| **Titulares de un recurso**, ordenados por saldo | No: índice nuevo (h): `resource → (entidad, saldo)` |
| **Dónde está un NFT** | No: índice (b) de E3, o N2 |
| Validadores: uptime, historial de stake, APY | No: índice (g) de E3, o N9. El stake actual sale del nodo. |
| Estado de la cuenta o del recurso en una fecha pasada | No: nodo de archivo (N5), o índice de cambios de saldo (a) + (h) |
| Estadísticas: TPS, transacciones/día, cuentas activas, comisiones quemadas | No: índice nuevo (i): agregados por hora y día |
| Buscar por nombre o símbolo (metadatos) | No: índice nuevo (j): texto `name`, `symbol`, `tags` → entidad |
| Buscar por dirección | Sí: la dirección indica su tipo; basta con `/entity/info` |

**Consecuencias para la estrategia:**

- Los exploradores **descartan E1**: con el nodo solo, se pierden las páginas de historial, de titulares y de estadísticas.
- **E3** tiene que incluir los índices (h), (i) y (j). Siguen siendo pequeños:
  - (h) depende del número de pares recurso-titular, no de las versiones de estado;
  - (i) son unas pocas filas por hora;
  - (j) solo cubre entidades con metadatos `name` o `symbol`.
- **El explorador oficial (Radix Dashboard) no es de código abierto.** Para que la red tenga un explorador que no dependa de nadie, hace falta además un **explorador de referencia de código abierto** que funcione contra `radix-lite-indexer` o contra cualquier punto de acceso descubierto en la cadena (sección [7.16](07-cambios-por-repositorio.md#s7-16)).

<a id="s2-2"></a>

## 2.2 La herencia de Olympia

> [!IMPORTANT]
> - **¿Es posible?** Sí. Olympia no impide eliminar el Gateway. Archivar su historial también es posible.
> - **Dificultad:** Baja.
> - **Esfuerzo y plazo:** semanas para el archivo (estimación); es urgente, porque debe hacerse antes de que se apague el Gateway de Olympia.

```mermaid
flowchart LR
    O[("Ledger de Olympia<br/>congelado el 28-sep-2023")] -- "estado migrado:<br/>saldos, stakes, tokens" --> G["Génesis de Babylon<br/>versiones 1 a 647"]
    G --> B[("Ledger de Babylon<br/>lo guarda cada nodo")]
    O -- "historial de transacciones<br/>(no está en Babylon)" --> OG["Gateway de Olympia<br/>un solo servicio"]
    O -. "propuesta: exportar una vez" .-> AR["Archivo verificable<br/>IPFS, BitTorrent o Arweave"]
    classDef risk fill:#ffd6d6,stroke:#c00,color:#000
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class OG risk
    class AR good
```

Radix tuvo una red anterior, **Olympia**, que se migró a Babylon el 28-sep-2023. Se ha comprobado en el ledger real y en el código: **no impide la solución**, pero añade cuatro consideraciones, tres de las cuales requieren trabajo (puntos 1, 3 y 4).

1. **El estado de Olympia está dentro de la génesis de Babylon.**
   - Las versiones de estado 1 a 647 de mainnet son las transacciones de génesis (las 648-669 son ya cambios de ronda). La primera transacción de usuario es la versión 670 (2023-09-28 07:26 UTC).
   - Esas transacciones cargan, en bloques (`GenesisDataChunk`: `Validators`, `Stakes`, `Resources`, `ResourceBalances`, `XrdBalances`; `radix-engine/.../genesis_helper/src/lib.rs:34-46`), los validadores, stakes, tokens y saldos que había en Olympia. Las cuentas se crean a partir de las claves secp256k1 de Olympia.
   - **Consecuencia para el indexador (E3/E2):**
     - la primera sincronización tiene que empezar en la versión 1 y procesar la génesis, que tiene recibos muy grandes;
     - si no lo hace, faltarían en los índices todas las cuentas migradas que no se han movido desde 2023: titulares de XRD, primer movimiento de la cuenta, stakes migrados.
   - Es un trabajo de una sola vez y se puede incluir en los snapshots firmados.
2. **Las cuentas de Olympia siguen funcionando sin el Gateway.**
   - Importar cuentas de Olympia al wallet de Babylon se hace **sin red**: exportación con `olympia-wallet` y `MigrateOlympiaAccountsUseCase` en el wallet Android.
   - ROLA acepta la derivación de la dirección para las cuentas sin `owner_keys`, también las secp256k1 de Olympia, aunque siempre consulta el ledger para comprobarlo (sección [7.9](07-cambios-por-repositorio.md#s7-9)).
   - El **escaneo de recuperación** con frase semilla de Olympia (ruta BIP44, `ResolveDerivationPathsForRecoveryScanUseCase.kt:42-79`) usa las mismas consultas al ledger que el resto. Con el nodo funciona: las cuentas creadas en la génesis existen y se detectan con `/entity/info`, aunque no exista `last_updated_at_state_version`.
3. **El historial de transacciones de Olympia (antes del 28-sep-2023) no está en el ledger de Babylon.**
   - Solo lo sirve el **Gateway de Olympia**, `olympia-gateway.radixdlt.com`. Sigue activo hoy, con Gateway API 1.1.14, congelado en la versión 337 759 839 del 2023-09-28T06:33:34Z.
   - Lo usan `olympia-wallet` (`src/helpers/network.ts:14-39`) y el explorador de Olympia (`radixdlt/explorer`).
   - Es **otro servicio central**, y no aparece en las listas de servicios de la transición publicadas por la Foundation que se han revisado.
   - Si se apaga, el historial público de Olympia (auditorías, impuestos, investigación) **se pierde**, salvo que alguien conserve una copia de la base de datos de un nodo o del Gateway de Olympia.
   - **Ventaja:** está congelado y ya no cambia, así que no necesita indexador. Basta un **archivo único y verificable**:
     - exportar el ledger de Olympia (base de datos del nodo o volcado del Gateway de Olympia);
     - publicarlo con su hash en almacenamiento distribuido (IPFS, BitTorrent o Arweave);
     - añadir un índice estático por dirección `rdx1…` que cualquier explorador pueda servir como ficheros.
   - **Prioridad P0, por urgencia:** hay que hacerlo antes de que se apague la infraestructura de la Foundation.
4. **Los exploradores deben entender las dos épocas.**
   - Una dirección de Olympia (`rdx1…`) se convierte de forma determinista en su cuenta de Babylon (`account_rdx…`), porque ambas salen de la misma clave. Sargon ya hace esa conversión, así que no hace falta índice.
   - El explorador de referencia ([7.16](07-cambios-por-repositorio.md#s7-16)) debe aceptar ambas direcciones y mostrar el historial de Olympia desde el archivo estático y el de Babylon desde los nodos.

---

[← Índice](../../README.es.md) · [Siguiente: Capítulo 3: qué ofrece hoy el nodo y qué le falta](03-que-ofrece-el-nodo.md)
