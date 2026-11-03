[← Índice](../../README.es.md)

# Glosario

Términos usados en el informe, explicados de forma breve.

| Término | Significado |
|---|---|
| **Babylon** | La versión actual de la red Radix, en funcionamiento desde septiembre de 2023. |
| **Olympia** | La versión anterior de la red Radix (2021-2023). Su estado se migró a Babylon. |
| **Hyperscale** | El diseño de nueva generación de Radix: una red fragmentada (*sharded*) que escala añadiendo fragmentos. Su implementación en Rust, `hyperscale-rs`, es de código abierto. |
| **Xi'an** | Nombre que la hoja de ruta de Radix dio a la etapa de la red basada en Hyperscale. |
| **Gateway (Babylon Network Gateway)** | Servicio central que indexa la red y ofrece una API a wallets, dApps y exploradores. Es una base de datos PostgreSQL (~2,5 TB), un indexador (DataAggregator) y una API. |
| **Nodo completo** | Ordenador que mantiene una copia del ledger y valida todas las transacciones, sin participar en el consenso. |
| **Validador** | Nodo completo que además participa en el consenso y firma los bloques. Mainnet tiene unos 100 activos. |
| **Nodo de lectura** | Propuesta de este informe: nodo completo con un indexador ligero, que atiende las consultas de wallets, dApps y exploradores. |
| **Nodo de archivo** | Nodo que conserva todo el historial y genera snapshots verificables para otros nodos. |
| **Indexador ligero (`radix-lite-indexer`)** | Propuesta de este informe: proceso que sustituye al Gateway, guarda solo los índices que el nodo no tiene y ocupa ~10-50 GB (estimado). |
| **Core API, LTS, Engine State API** | APIs que ofrece el propio nodo. La LTS es la parte de la Core API con compatibilidad a largo plazo. La Engine State API permite leer el estado interno de cualquier entidad. |
| **Ledger** | El registro completo de transacciones confirmadas de la red. |
| **Versión de estado (*state version*)** | Número de orden de cada transacción en el ledger. Mainnet supera los 567 millones. |
| **Cambio de ronda (*round update*)** | Transacción de sistema que se genera en cada ronda de consenso. Son más del 99 % de las versiones de estado. |
| **Época** | Periodo (~5 minutos) tras el cual puede cambiar el conjunto de validadores. |
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
| **Fragmento (*shard*)** | En Hyperscale, cada partición del estado con su propio consenso. |
| **Cadena beacon** | En Hyperscale, la cadena lenta que gestiona los validadores y la topología de fragmentos. |
| **BLS12-381** | Esquema de firmas que usa Hyperscale para los validadores, que permite agregar muchas firmas en una sola. |
| **WASM** | Formato de código ejecutable portable. Scrypto compila a WASM y la VM de Hyperscale también ejecuta WASM. |
| **RAC / DAO** | Radix Accountability Council y la organización comunitaria que gobierna la red tras la Foundation. |
| **Persona-mes** | Unidad de esfuerzo: el trabajo de una persona a tiempo completo durante un mes. |
