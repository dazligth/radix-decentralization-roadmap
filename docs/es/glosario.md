[← Índice](../../README.es.md) · [Anterior: Fuentes, metodología y preguntas abiertas](fuentes-y-metodologia.md)

# Glosario

Términos usados en el informe, explicados de forma breve.

| Término | Significado |
|---|---|
| **Babylon** | La versión actual de la red Radix, en funcionamiento desde septiembre de 2023. |
| **Olympia** | La versión anterior de la red Radix (2021-2023). Su estado se migró a Babylon. |
| **Hyperscale** | El diseño de nueva generación de Radix: una red fragmentada (*sharded*) que escala añadiendo fragmentos. Hay una implementación independiente en Rust, `hyperscale-rs`, de código abierto (capítulo [11](11-evolucion-hyperscale.md#s11)). |
| **Xi'an** | Nombre que la hoja de ruta de Radix dio a la etapa fragmentada de la red, hoy planteada sobre Hyperscale. El capítulo [13](13-xian-con-scrypto.md#s13) describe una alternativa, «Xi'an con Scrypto», que fragmenta Babylon conservando Scrypto. |
| **FINAL A, B y C** | Las tres etapas finales posibles: seguir mejorando Babylon (A), migrar a Hyperscale (B, capítulo [11](11-evolucion-hyperscale.md#s11)) o fragmentar Babylon conservando Scrypto (C, capítulo [13](13-xian-con-scrypto.md#s13)). |
| **Gateway (Babylon Network Gateway)** | Servicio central que indexa la red y ofrece una API a wallets, dApps y exploradores. Es una base de datos PostgreSQL (~2,5 TB, estimación), un indexador (DataAggregator) y una API. |
| **Nodo completo** | Ordenador que mantiene una copia del ledger y valida todas las transacciones. Si además participa en el consenso, es un validador. |
| **Validador** | Nodo completo que además participa en el consenso y firma los bloques. Mainnet tiene unos 100 activos. |
| **Nodo de lectura** | Propuesta de este informe: nodo completo con un indexador ligero, que atiende las consultas de wallets, dApps y exploradores. |
| **Nodo de archivo** | Nodo que conserva todo el historial y genera snapshots verificables para otros nodos. |
| **Indexador ligero (`radix-lite-indexer`)** | Propuesta de este informe: proceso que sustituye al Gateway, guarda solo los índices que el nodo no tiene y añade ~10-50 GB (estimación) al nodo completo. |
| **Core API, LTS, Engine State API** | APIs que ofrece el propio nodo. La LTS es la parte de la Core API con compatibilidad a largo plazo. La Engine State API permite leer el estado interno de cualquier entidad. |
| **Ledger** | El registro completo de transacciones confirmadas de la red. |
| **Versión de estado (*state version*)** | Número de orden de cada transacción en el ledger. Mainnet supera los 567 millones. |
| **Cambio de ronda (*round update*)** | Transacción de sistema que se genera en cada ronda de consenso. Son ~97-98 % de las versiones de estado en todo el historial. |
| **Época** | Periodo (unos 5 minutos de media) tras el cual puede cambiar el conjunto de validadores. |
| **Recibo (*receipt*)** | Resultado de ejecutar una transacción: cambios de estado, eventos y comisiones. |
| **Subintent / pre-autorización** | Parte de una transacción firmada por adelantado por un usuario, que otra parte completa después. |
| **ROLA** | Protocolo con el que una dApp verifica que un usuario controla una cuenta o persona, usado para el inicio de sesión. |
| **RDT (Radix dApp Toolkit)** | Biblioteca que usan las dApps para conectarse al wallet. |
| **Sargon** | Biblioteca en Rust que comparten los wallets de iOS y Android. |
| **Radix Connect** | Canal entre el wallet móvil y el navegador o la dApp: relé, señalización WebRTC o iroh. |
| **Scrypto** | Lenguaje de Radix para escribir contratos (*blueprints*), basado en Rust. |
| **Radix Engine** | Máquina virtual que ejecuta las transacciones y los contratos Scrypto en Babylon. |
| **Manifiesto** | Descripción legible de lo que hace una transacción en Radix. |
| **SBOR** | Formato de codificación de datos de Radix. |
| **RocksDB** | Base de datos embebida que usa el nodo para guardar el ledger y el estado. |
| **Column family** | Cada una de las «tablas» internas de RocksDB (el nodo usa 26). |
| **JMT (Jellyfish Merkle Tree)** | Árbol de Merkle con el que el nodo calcula la raíz del estado. Permite demostrar que un dato pertenece al estado. |
| **Prueba de Merkle** | Prueba criptográfica corta de que un dato forma parte de un conjunto cuya raíz está firmada. |
| **Prueba del ledger** | Cabecera firmada por los validadores que incluye las raíces del estado, de las transacciones y de los recibos. |
| **Cliente ligero** | Programa que verifica las respuestas de un nodo con pruebas criptográficas, sin guardar el ledger y sin confiar en el nodo. |
| **Snapshot verificado** | Copia del estado en una versión concreta que el nodo comprueba contra la raíz firmada por los validadores antes de usarla. |
| **Poda (*pruning*)** | Borrar datos antiguos que el nodo ya no necesita. |
| **JVM** | Máquina virtual de Java. Ejecuta la parte del nodo escrita en Java. |
| **JNI** | Puente entre Java y el código nativo (Rust) dentro del mismo proceso. |
| **Fragmento (*shard*)** | Partición del estado con su propio consenso, en Hyperscale y en la etapa final C (capítulo [13](13-xian-con-scrypto.md#s13)). |
| **Lista de accesos** | Propuesta del capítulo [13](13-xian-con-scrypto.md#s13): lista de entidades globales, con modo de lectura o escritura, que una transacción declara antes de ejecutarse. Si accede a otra, se aborta de forma determinista. |
| **Segmento de historial** | Propuesta de la sección [6.5](06-solucion-recomendada.md#s6-5): fichero inmutable con las transacciones y recibos de un tramo fijo de épocas y las pruebas firmadas por los validadores. Todos los nodos honrados producen los mismos bytes para el mismo tramo. |
| **Cerberus** | Protocolo de consenso fragmentado de Radix, publicado en 2023 (*Journal of Systems Research*); base del plan original de Xi'an (capítulo [13](13-xian-con-scrypto.md#s13)). |
| **OLLP** | Técnica de Calvin (SIGMOD 2012): una consulta previa descubre qué datos usará una transacción y, si cambian, la transacción se repite de forma determinista. |
| **Pilotfish** | Motor de ejecución escalable de investigadores de Sui (prueba de concepto) que permite accesos dinámicos dentro de un objeto raíz declarado. |
| **Quint** | Lenguaje de especificación formal usado por Hyperscale para modelar sus protocolos. |
| **Vnodes** | En Hyperscale, varias identidades de validador ejecutadas por un mismo operador. |
| **Exportar antes de podar** | Regla por la que un nodo solo borra el historial antiguo cuando uno o varios archivos han confirmado que lo guardaron y verificaron. |
| **Compromiso atómico entre fragmentos** | Protocolo que garantiza que una transacción que afecta a varios fragmentos se confirma en todos o en ninguno. |
| **Cadena beacon (cadena coordinadora)** | En Hyperscale y en la etapa final C, la cadena lenta que gestiona los validadores, las épocas y la topología de fragmentos. |
| **BLS12-381** | Esquema de firmas que usa Hyperscale para los validadores, que permite agregar muchas firmas en una sola. |
| **WASM (WebAssembly)** | Formato de código ejecutable portable. Scrypto compila a WASM y la VM de Hyperscale también ejecuta WASM. |
| **RAC (Radix Accountability Council)** | Consejo elegido que hace de puente en la gobernanza mientras se constituye la DAO. |
| **DAO** | Organización comunitaria que gobierna la red tras la Foundation, con decisiones y financiación en la cadena. |
| **XRD** | Token nativo de la red Radix; se usa para pagar comisiones y para el *staking*. |
| **dApp** | Aplicación descentralizada que usa la red Radix. |
| **Wallet** | Aplicación del usuario que guarda sus claves y firma transacciones. |
| **NFT** | Token no fungible: cada unidad es única. |
| **Preview** | Simulación de una transacción sin enviarla, para ver su resultado y sus comisiones. |
| **`api_endpoints`** | Metadato propuesto en este informe para que cada validador publique sus puntos de acceso de lectura. |
| **Koios / Leios / Geyser** | Koios: API comunitaria de Cardano con varios operadores. Leios: mejora del consenso de Cardano para aumentar su capacidad. Geyser: interfaz de complementos del validador de Solana para enviar datos a indexadores. |
| **Mainnet / Stokenet** | Mainnet es la red principal de Radix. Stokenet es su red de pruebas pública. |
| **Radix Foundation** | Organización que desarrolló Radix y operó sus servicios centrales; desde 2026 está en modo de mantenimiento. |
| **Radix Dashboard** | Explorador oficial de la red, operado por la Foundation; su código no es público. |
| **Developer Console** | Herramienta web oficial para desplegar paquetes y enviar manifiestos; su código no es público. |
| **Connector Extension** | Extensión de navegador que conecta las dApps de escritorio con el wallet móvil. |
| **Mesh API (Rosetta)** | API estándar para exchanges que ofrecía el nodo; eliminada en la versión `v1.4.0.0`. |
| **DataAggregator** | Parte del Gateway que lee el ledger de los nodos y lo guarda en PostgreSQL. |
| **Capa 1 / capa 2** | La capa 1 es la propia red (protocolo y nodos). La capa 2 son soluciones que funcionan encima o al lado de ella, como los *rollups* o las cadenas asociadas. |
| **Cadena asociada (*partner chain*)** | Red independiente vinculada a otra, como Midnight respecto a Cardano. |
| **LSU (Liquid Stake Unit)** | Token que representa XRD depositados en *staking* con un validador concreto. |
| **Pool unit** | Token que representa una participación en un pool de liquidez. |
| **Badge** | Recurso que se usa como credencial para obtener permisos. |
| **Cuenta securificada** | Cuenta cuyo control pasa de una clave única a reglas de acceso más complejas (varios factores, recuperación). |
| **MFA / shields** | Configuración de seguridad multifactor de los wallets de Radix. |
| **Persona** | Identidad de inicio de sesión del usuario en Radix, separada de sus cuentas. |
| **RNS (Radix Name Service)** | Servicio de nombres legibles para direcciones de Radix. |
| **Account locker** | Componente nativo que guarda recursos para que una cuenta los reclame. |
| **KV-store** | Almacén clave-valor dentro de un componente. |
| **Blueprint / componente / paquete** | Un paquete contiene blueprints (plantillas de contratos); un componente es una instancia de un blueprint. |
| **Vault** | Contenedor de recursos (tokens o NFT) dentro de una cuenta o componente. |
| **Royalties** | Comisión que un desarrollador puede cobrar cuando se usa su paquete o componente. |
| **Intent hash** | Identificador de una transacción en Radix. |
| **Moratoria** | Mecanismo añadido en `v1.4.0.0` (commit `8ea6cf97`) para detener temporalmente las transacciones de usuario. |
| **Códigos del informe** | E1-E6: estrategias (capítulo [4](04-05-estrategias.md#s4)). N1-N9 y N5b: capacidades que faltan en el nodo (capítulo [3](03-que-ofrece-el-nodo.md#s3) y sección [7.2](07-cambios-por-repositorio.md#s7-2)); N10: exportación canónica del historial (secciones [6.5](06-solucion-recomendada.md#s6-5) y [7.2](07-cambios-por-repositorio.md#s7-2)). (D): descubrimiento en la cadena; (L): interfaz `LedgerBackend` (capítulo [6](06-solucion-recomendada.md#s6)). R1-R11: cambios para aligerar los nodos (capítulo [9](09-nodos-ligeros.md#s9)). M1-M3: estrategias de migración a Hyperscale (capítulo [11](11-evolucion-hyperscale.md#s11)). F0-F5: fases de la hoja de ruta (capítulo [8](08-hoja-de-ruta.md#s8)). C0-C6: etapas de Xi'an con Scrypto (capítulo [13](13-xian-con-scrypto.md#s13)). (a)-(j): índices del indexador ligero (sección [7.1](07-cambios-por-repositorio.md#s7-1)). |
| **Prioridad P0-P2 / esfuerzo S-XL** | P0 imprescindible, P1 necesario, P2 mejora. S días, M 1-3 semanas, L más de un mes, XL varios meses o más. |
| **Punto de acceso (*endpoint*)** | Dirección de red a la que una aplicación envía sus peticiones. |
| **OpenAPI** | Formato estándar para describir una API web (rutas, peticiones y respuestas). |
| **CORS** | Mecanismo que permite que una página web consulte una API alojada en otro dominio. |
| **TLS** | Cifrado de las conexiones web (HTTPS). |
| **Límite de peticiones (*rate limit*)** | Número máximo de peticiones que se aceptan de un mismo origen en un periodo. |
| **Conmutación por error (*failover*)** | Pasar automáticamente a otro servidor cuando el actual falla. |
| **Espera exponencial (*backoff*)** | Reintentar tras esperas que crecen en cada intento. |
| **Mempool** | Conjunto de transacciones recibidas por el nodo y pendientes de confirmación. |
| **Heap / RSS / swap** | Heap: memoria reservada para Java. RSS: memoria física que usa realmente un proceso. Swap: espacio en disco que el sistema usa como memoria adicional. |
| **SST / memtable** | Ficheros de datos en disco de RocksDB / tablas en memoria donde RocksDB acumula escrituras antes de pasarlas a disco. |
| **Vertex store** | Estructura del consenso que guarda los bloques propuestos aún no confirmados. |
| **HotStuff / BFT / finalidad** | HotStuff es el algoritmo de consenso de Babylon, de tipo BFT (tolerante a fallos bizantinos). Finalidad: una transacción confirmada no se revierte. |
| **QC (certificado de quórum)** | Conjunto de firmas de validadores que acredita un acuerdo del consenso. |
| **Acumulador** | Estructura de hashes que resume todas las transacciones o recibos confirmados. |
| **`state_tree_hash`** | Raíz del árbol de estado, incluida en las pruebas del ledger que firman los validadores. |
| **QUIC / iroh** | QUIC es un protocolo de transporte de red; iroh es una biblioteca que lo usa para conexiones directas entre dispositivos. |
| **WebRTC / señalización / TURN / STUN** | WebRTC conecta navegador y wallet; la señalización los pone en contacto; TURN y STUN ayudan a atravesar redes privadas. |
| **IPFS / BitTorrent / Arweave** | Sistemas de almacenamiento y distribución de ficheros descentralizados. |
| **ZK / zk-SNARK / Halo2** | Pruebas de conocimiento cero: demuestran algo sin revelar los datos. zk-SNARK es un tipo de prueba corta; Halo2 es un sistema concreto. |
| **HBOR / Blake3 / ML-DSA** | En Hyperscale: formato de codificación de datos, función hash y esquema de firma post-cuántica, respectivamente. |
| **Persona-mes** | Unidad de esfuerzo: el trabajo de una persona a tiempo completo durante un mes. |

---

[← Índice](../../README.es.md) · [Anterior: Fuentes, metodología y preguntas abiertas](fuentes-y-metodologia.md)
