[← Índice](../../README.es.md) · [Anterior: Capítulo 11: evolución final hacia Hyperscale](11-evolucion-hyperscale.md) · [Siguiente: Fuentes, metodología y preguntas abiertas](fuentes-y-metodologia.md)

<a id="s12"></a>

# 12. Referencias de otras redes descentralizadas

Varias propuestas de este informe no tienen precedente en Radix, pero sí en otras redes que han publicado su diseño y su experiencia. Este capítulo recoge esos precedentes, con sus fuentes, consultadas el 3-nov-2026. **Un precedente no garantiza que la misma solución funcione igual en Radix**: cada una debe validarse aquí.

Como el objetivo de Radix es resolverlo todo en la **capa 1**, cada referencia indica si es un mecanismo de capa 1 (protocolo o software del nodo), un servicio externo de lectura, o una solución de capa 2 o de cadena asociada, que se cita solo como contraste.

<a id="s12-1"></a>

## 12.1 Resumen por propuesta

| Propuesta de este informe | Precedentes | Qué muestran |
|---|---|---|
| **R9.** Arranque desde un snapshot verificado | **Sui:** snapshots «formales» del estado al final de cada época, verificados automáticamente contra un compromiso firmado por los validadores. **Polkadot:** *warp sync*, que descarga y verifica pruebas de finalidad y después el estado. **CometBFT/Cosmos:** *state sync*, verificado con un cliente ligero. **Cardano:** Mithril, snapshots certificados por firmas de umbral ponderadas por stake. **Kaspa:** prueba del punto de poda más el conjunto de UTXO. **Ethereum:** *checkpoint sync*. | Es una técnica estándar en casi todas las redes modernas, en capa 1. Solana es el contraejemplo: la confianza en el snapshot se basa en elegir validadores conocidos. |
| **R8.** Nodos podados y nodos de archivo | **Polkadot:** poda por defecto (últimos 256 bloques). **NEAR:** recolección de basura (5 épocas por defecto) y nodos de archivo con almacenamiento en dos niveles. **Sui** y **Aptos:** poda configurable y el historial en un archivo o en el indexador. **Kaspa:** poda por diseño, con modo de archivo opcional. **Cosmos SDK:** opciones `default`, `nothing`, `everything` y `custom`. **Ethereum:** expiración parcial del historial (EIP-4444) en todos sus clientes de ejecución desde julio de 2025, con un ahorro de 300-500 GB por nodo. | Podar por defecto y dejar el historial completo a unos pocos nodos de archivo es la práctica habitual. |
| **9.5.** Nodos con poca RAM | **Cardano:** al llevar el ledger a disco (UTxO-HD con LSM), la memoria residente bajó de ~15-16 GB a ~5,0-5,8 GB (~65 %) en la versión beta del nodo (informe de IOG, T2-2026); reproducir la cadena requirió solo 4 GB en pruebas de mainnet (T3-2025). **Ethereum:** el cliente de consenso Nimbus funciona con 4 GB. | Mover datos de la memoria al disco reduce mucho la RAM. Ver en 12.9 por qué Radix lo tiene más fácil. |
| **E2.** API de lectura dentro del nodo | **Kaspa:** API wRPC integrada en el nodo. **Sui:** índice gRPC en el nodo completo. **Cardano:** Acropolis (IOG), un nodo modular en Rust con una API compatible con Blockfrost dentro del nodo, que sincronizaba en ~1 hora en lugar de 4 días (proyecto cerrado en T2-2026; el código sigue publicado). | Integrar la lectura en el propio nodo, como propone E2, ya se ha hecho. |
| **E3.** Indexador junto al nodo | **Solana:** interfaz Geyser, donde complementos cargados en el validador envían cuentas, transacciones y bloques a bases de datos externas. **Aptos:** servicio de flujo de transacciones (gRPC) para indexadores. **Cardano:** Kupo, indexador ligero. | El patrón de alimentar indexadores ligeros desde el nodo es común. |
| **10.2.** Descubrimiento de puntos de acceso | **Cardano: Koios**, una API comunitaria «sin instancia central», con operadores independientes que se añaden a un fichero de topología en GitHub y un monitor que solo marca como activos los que están al día. **Kaspa:** red de nodos públicos con un servicio *Resolver* que dirige a los clientes al nodo menos cargado. **Polkadot:** los clientes ligeros encuentran nodos por P2P, sin depender de servidores RPC. | Hay redes de lectura federadas, pero **ninguna de las analizadas publica sus puntos de acceso en un registro en la propia capa 1**. El descubrimiento en la cadena que propone este informe sería una aportación nueva. |
| **6.2.** Lecturas verificables y cliente ligero | **Polkadot:** smoldot, cliente ligero que funciona en el navegador y «no confía en absoluto» en los nodos completos. **Sui:** cliente ligero que sigue los checkpoints de fin de época firmados por los validadores. **Ethereum:** `eth_getProof` (EIP-1186), comités de sincronización y Helios. | El cliente ligero en el navegador o en el wallet existe en producción. |
| **6.2, punto 6.** Incentivos para servir lecturas | **Pocket Network** y **The Graph** pagan a quien sirve lecturas o índices, con pruebas del trabajo hecho. | Son modelos útiles, pero funcionan como redes aparte. En Radix habría que implementarlos en la propia capa 1. |
| **R1.** Límites de memoria de RocksDB | **Documentación oficial de RocksDB:** por defecto, cada column family tiene su propia caché de bloques (32 MB desde la versión 8.2; 8 MB antes); los bloques de índice y de filtro no cuentan dentro de esa caché; `max_open_files = -1` precarga todos los índices y filtros; el *Write Buffer Manager* limita la memoria de las memtables entre column families y puede contabilizarla en la caché de bloques. | Confirma las causas del consumo de memoria descritas en 9.2 y la solución de R1. |
| **R11.** Reescritura del nodo en Rust | **Kaspa:** `rusty-kaspa` sustituyó al nodo en Go como «reemplazo directo», conviviendo en la red P2P, con adopción gradual y retirada posterior del nodo en Go. **Solana:** Frankendancer, cliente híbrido. **Cardano:** nuevos nodos (Amaru en Rust, Dingo en Go) que entran primero como relés y después producen bloques. **Ethereum:** diversidad de clientes. | Reescribir el nodo por fases, con convivencia en la red, es un camino ya recorrido. Kaspa es el precedente más cercano a R11. |
| **M1.** Migración con reclamación | **Midnight (Glacier Drop):** instantánea de saldos de ocho redes; el usuario firma un mensaje con la clave de la dirección de la foto e indica una dirección de destino nueva, con un periodo posterior para quien no reclamó a tiempo. **Cosmos Hub:** migración del estado a una nueva génesis. **Uniswap:** contrato `MerkleDistributor`. | Las reclamaciones firmadas y la migración por génesis son técnicas probadas. |
| **Capítulo [11](11-evolucion-hyperscale.md#s11).** Evolución de la capa 1 | **Cardano: Ouroboros Leios**, que aumenta la capacidad de la capa 1 mediante una bifurcación de protocolo **en la misma cadena**, con una red de pruebas dedicada de larga duración. | Muestra la alternativa a migrar a una red nueva. Ver 12.9. |

<a id="s12-2"></a>

## 12.2 Cardano

| Mecanismo | Tipo | Qué hace | Estado |
|---|---|---|---|
| **Mithril** | Capa 1 (firmado por los operadores de stake pools, fuera del consenso) | Certifica snapshots con firmas de umbral ponderadas por stake; el cliente verifica la cadena de certificados y restaura un nodo de mainnet «en menos de 20 minutos», según su documentación | En producción (informe de IOG, T2-2026). La última cifra de participación encontrada es de 2024 (~22 % del stake). |
| **UTxO-HD / LSM** | Capa 1 | Lleva el ledger de la memoria al disco | Beta del nodo 10.7.1/11.0.1: RAM de ~15-16 GB a ~5,0-5,8 GB. Requisitos oficiales: 24 GB en memoria; 8 GB con el ledger en disco («pendiente de confirmación»); 250-350 GB de disco. |
| **Koios** | Servicio de lectura federado | API comunitaria con varios operadores, sin instancia central | En funcionamiento; descubrimiento por fichero en GitHub, sin incentivos en la cadena |
| **cardano-db-sync / Blockfrost** | Servicio de lectura | Indexador completo en PostgreSQL (~438 GB, 64 GB de RAM recomendados); Blockfrost publica su backend para que cualquiera lo aloje | En funcionamiento. Es el equivalente al Gateway de Radix, y muestra su coste. |
| **Amaru (Rust), Dingo (Go)** | Capa 1 | Nuevas implementaciones del nodo, que entran primero como relés | Amaru funciona como relé en mainnet; la fecha de producción de bloques no está verificada |
| **Ouroboros Leios** | Capa 1 (bifurcación de protocolo) | Añade bloques de respaldo (*endorser blocks*) votados por un comité, para aumentar la capacidad sin cambiar la cadena principal (CIP-164, propuesta) | Red de pruebas desde el 23-jun-2026; código completo previsto para el T4-2026; fecha de mainnet sin fijar. Requiere más disco y más recursos en los relés de quien tenga mucho stake. |

<a id="s12-3"></a>

## 12.3 Sui

| Mecanismo | Tipo | Qué hace |
|---|---|---|
| **Poda configurable** | Capa 1 | Épocas a conservar de objetos y de checkpoints; sin poda, un nodo puede superar 7 TB |
| **Snapshots formales** | Capa 1 | Estado vivo al final de cada época, verificado contra un compromiso firmado por los validadores; según la documentación de Sui, en mainnet se conservan los de unas 60 épocas |
| **Cliente ligero** | Capa 1 | Sigue los checkpoints de fin de época, con firmas BLS agregadas de los validadores, desde la génesis |
| **Indexador y GraphQL** | Servicio de lectura | Indexador en PostgreSQL y API GraphQL, que pueden alojar los propios usuarios o proveedores; JSON-RPC marcado como obsoleto |
| **Hardware de un nodo completo** | — | 128 GB de RAM y 4 TB NVMe |

<a id="s12-4"></a>

## 12.4 Aptos

- **Sincronización rápida** (capa 1): omite el historial y se basa en las firmas de los validadores; suele completarse en pocas horas.
- **Poda** (capa 1): por defecto conserva los últimos 150 millones de transacciones; los nodos de archivo están obsoletos en favor del indexador.
- **Indexador** (servicio de lectura): flujo de transacciones por gRPC y API GraphQL, alojable por terceros.
- **Hardware de un validador:** 128 GB de RAM y 3 TB NVMe (estimación oficial para 2024).

<a id="s12-5"></a>

## 12.5 Solana

- **Geyser** (capa 1): complementos dentro del validador que envían datos a indexadores externos. Es el precedente más cercano al indexador ligero de E3.
- **Snapshots** (capa 1): se recomienda descargarlos solo de validadores conocidos, para evitar estados maliciosos; la confianza es social, no criptográfica.
- **Historial** (servicio externo): durante años solo estaba en Google Bigtable; el proyecto Old Faithful lo empaqueta en ficheros que se pueden alojar en IPFS o Filecoin.
- **Hardware:** nodo RPC con 256 GB de RAM. Es un contraejemplo de nodo ligero.

<a id="s12-6"></a>

## 12.6 NEAR y Polkadot

- **NEAR** (capa 1): nodos RPC con 5 épocas por defecto y nodos de archivo con almacenamiento caliente (3 TB SSD) y frío (115 TB HDD). Su servicio de datos NEAR Lake, de un solo operador, dejó de indexar el 24-mar-2026. Es un ejemplo del riesgo de depender de un servicio central de lectura.
- **Polkadot** (capa 1):
  - nodos podados por defecto;
  - *warp sync* (pruebas de finalidad + estado);
  - cliente ligero **smoldot** en el navegador, que la documentación describe como alternativa «de confianza mínima» frente a los servidores RPC públicos, «cómodos pero centralizados».
  - Validador: 32 GB de RAM y 2 TB NVMe.

<a id="s12-7"></a>

## 12.7 Kaspa

| Mecanismo | Tipo | Qué hace |
|---|---|---|
| **rusty-kaspa** | Capa 1 | Reescritura en Rust del nodo en Go, como «reemplazo directo». Primero convivió con los nodos en Go en la red P2P, con una pequeña parte de la potencia de minado; después pasó a ser el nodo recomendado y el nodo en Go quedó obsoleto. La bifurcación Crescendo (5-may-2025) llevó la red de 1 a 10 bloques por segundo ya con el nodo en Rust. |
| **Poda por diseño** | Capa 1 | Un nodo nuevo descarga la prueba del punto de poda, las cabeceras, el conjunto de UTXO de ese punto y después los bloques. Las cabeceras incluyen un compromiso del conjunto de UTXO. Hay opciones de retención y de archivo. |
| **API y nodos públicos** | Lectura | API wRPC en el nodo; red de nodos públicos aportados por la comunidad y un *Resolver* que reparte a los clientes |
| **Hardware** | — | Mínimo 16 GB de RAM y 256 GB SSD; preferible 32 GB y 512 GB |

Kaspa es el precedente más directo de dos propuestas de este informe: **R11** (reescritura del nodo en Rust por fases) y **R8 + R9** (poda por defecto con arranque verificable).

<a id="s12-8"></a>

## 12.8 Midnight: cómo implementa la privacidad (contraste)

**Tipo:** red aparte, **cadena asociada** a Cardano, construida con el kit Partner Chains (basado en Substrate); sus productores de bloques deben ser operadores de stake pools de Cardano. **No es un modelo para Radix**, cuyo objetivo es la capa 1. Se analiza por su forma de implementar la privacidad.

| Elemento | Cómo funciona |
|---|---|
| **Pruebas de conocimiento cero** | zk-SNARK con el sistema Halo2 (sustituyó a Plonk en 2024), adaptado para recursión eficiente (notas de versión de 2024; la documentación posterior describe una pila basada en PLONK/KZG sobre BLS12-381). |
| **Estado público y privado** | El estado público está en la cadena; el privado lo guarda cada usuario localmente y nunca se expone. |
| **Tokens** | NIGHT (público, modelo UTXO, firmas Schnorr); tokens protegidos (*shielded*, con pruebas ZK); DUST, un recurso protegido e intransferible que se genera al poseer NIGHT y que se consume para pagar las transacciones. |
| **Lenguaje Compact** | Lenguaje parecido a TypeScript que compila a circuitos ZK. Los datos privados entran como *witnesses* desde el cliente; cualquier dato privado que vaya a hacerse público debe marcarse con `disclose()`, o el compilador rechaza el programa. |
| **Generación de pruebas** | En un «servidor de pruebas» local del usuario; la documentación aconseja ejecutarlo en una máquina propia, porque recibe los datos privados. |
| **Divulgación selectiva** | Claves de visualización que permiten a un indexador autorizado leer el historial protegido de un usuario. |
| **Distribución de NIGHT (Glacier Drop)** | Foto de saldos de ocho redes (11-jun-2025); el usuario firma un mensaje con la clave de la dirección de la foto e indica una dirección nueva; después, periodos adicionales para quien no reclamó. Solo admitía cuentas de clave única, no multifirma ni contratos. |
| **Lanzamiento** | NIGHT se emitió como activo de Cardano en diciembre de 2025; la red principal estaba prevista para finales de marzo de 2026 (la fecha exacta no se ha verificado). |

**Qué podría aplicarse en la capa 1 de Radix** (requiere trabajo profundo en el motor y una actualización de protocolo):

1. **Verificación de pruebas ZK como función nativa** del Radix Engine, invocable desde Scrypto.
2. **Recursos protegidos** junto a los recursos públicos (compromisos y anuladores), con comisiones pagadas mediante un recurso generado e intransferible, al estilo de DUST.
3. **Disciplina de divulgación explícita** de los datos privados, comprobada por el compilador, en una extensión de Scrypto.
4. **Generación de pruebas en el wallet**, de forma local.
5. **Reclamaciones firmadas con plazo largo** para cualquier migración de tokens (relevante para M1). La limitación a cuentas de clave única muestra que Radix debe prever las cuentas securificadas.

Lo que **no** se trasladaría: la cadena separada, los operadores federados, la dependencia de Cardano y el puente entre redes.

<a id="s12-9"></a>

## 12.9 Lecciones para Radix

1. **El arranque desde un snapshot verificado (R9) es estándar** en Sui, Polkadot, Cosmos, Cardano, Kaspa y Ethereum. Refuerza su prioridad.
2. **Podar por defecto y dejar el historial a nodos de archivo (R8)** es la práctica habitual.
3. **Los 4 GB de RAM: Radix lo tiene más fácil que otras redes, pero hay que medirlo.**
   - Ninguna de las redes analizadas funciona hoy con 4 GB en un nodo completo que haga consenso y ejecución. Los requisitos publicados van de 8 GB (Cardano con el ledger en disco, pendiente de confirmación) a 128-256 GB (Sui, Aptos, Solana), pasando por 16 GB (Kaspa, Geth). El precedente más cercano es **Cardano**, con ~5,0-5,8 GB tras llevar el ledger a disco, en su versión beta.
   - Sin embargo, **Radix funciona de otra forma** en tres aspectos que reducen la memoria necesaria:
     - **su estado ya está en disco** (RocksDB, column family `substates`). La gran reducción de Cardano consistió precisamente en llevar a disco un ledger que tenía en memoria; Radix no necesita ese cambio estructural, y su consumo actual se debe sobre todo a la configuración (12 GB reservados para Java, RocksDB sin límites y una mempool de ~1,2 GB), que se puede ajustar (R1-R2);
     - **su conjunto de validadores es pequeño** (unos 100), así que el estado del consenso ocupa poco (el vertex store está limitado a 150 MB);
     - **su carga actual es baja** (~4 400 transacciones de usuario al día, sección [1.2](01-02-situacion-actual.md#s1-2)), muy lejos de redes como Solana o Sui, cuyos requisitos responden a volúmenes mucho mayores.
   - **Valoración:** con R1-R2, un nodo de **8 GB como máximo parece alcanzable**, y **4 GB es plausible**, sobre todo tras la migración a Rust (R11). Ambas cifras son estimaciones que deben confirmarse midiendo un nodo real, también en periodos de carga alta como el pico de marzo de 2025 (~35 000 transacciones de usuario al día).
4. **La lectura integrada en el nodo (E2) y el indexador junto al nodo (E3)** tienen precedentes directos (Kaspa, Sui, Acropolis; Geyser, Aptos).
5. **Las redes de lectura federadas existen** (Koios, nodos públicos de Kaspa), pero se coordinan fuera de la cadena. **Publicar los puntos de acceso en la propia capa 1**, como propone este informe, sería una aportación nueva de Radix.
6. **Depender de un único servicio de lectura es un riesgo real:** NEAR Lake dejó de indexar en marzo de 2026.
7. **La reescritura en Rust por fases (R11)** tiene un precedente muy similar en **Kaspa**, que pasó de Go a Rust conviviendo en la red, con adopción gradual.
8. **Evolucionar la capa 1 sin cambiar de red es posible** cuando el nuevo diseño es compatible con el anterior, como hace Cardano con Leios. Hyperscale, en cambio, no es compatible con Babylon (sección [11.2](11-evolucion-hyperscale.md#s11-2)). Por eso exige una migración, y por eso conviene que la comunidad compare también la opción de mejorar Babylon en su propia cadena antes de decidir.
9. **La privacidad** se puede plantear en la capa 1 con verificación de pruebas ZK en el motor y recursos protegidos. Es un trabajo de gran envergadura, que Midnight resuelve con una red aparte.

**Qué no se ha podido verificar:**

- la participación de stake en Mithril en 2025-2026;
- la fecha de producción de bloques de Amaru;
- la fecha exacta de cierre del JSON-RPC de Sui;
- cómo verifica Solana los snapshots al cargarlos;
- la ventana de poda exacta de Kaspa;
- la fecha exacta del lanzamiento de la red principal de Midnight;
- el esquema de firma por red del Glacier Drop;
- el estado de la Portal Network de Ethereum en 2026;
- la fecha en que Firedancer completo entró en mainnet.

Ninguno de estos puntos se usa como argumento en el informe.

---

[← Índice](../../README.es.md) · [Anterior: Capítulo 11: evolución final hacia Hyperscale](11-evolucion-hyperscale.md) · [Siguiente: Fuentes, metodología y preguntas abiertas](fuentes-y-metodologia.md)
