[← Índice](../../README.es.md) · [Anterior: Capítulos 4-5: las seis estrategias evaluadas y su comparativa](04-05-estrategias.md) · [Siguiente: Capítulo 7: cambios concretos en cada repositorio](07-cambios-por-repositorio.md)

<a id="s6"></a>

# 6. Solución recomendada y por qué

> [!IMPORTANT]
> - **¿Es posible?** Sí. Es la solución recomendada a corto plazo.
> - **Dificultad:** Media.
> - **Esfuerzo y plazo:** 7-12 personas-mes la solución mínima viable; meses (estimación).

## Infraestructura a corto plazo: sin Gateway

```mermaid
flowchart TB
    subgraph APPS["Usuarios y aplicaciones (versiones actuales; solo cambia la URL)"]
        W["Wallets iOS y Android"]
        EXT["Connector Extension"]
        DAPP["dApps con RDT"]
        ROLA["Backends con inicio de sesión ROLA"]
        EXP["Explorador de referencia<br/>web estática, código abierto"]
    end
    REG["Registro en la cadena<br/>metadatos api_endpoints"]
    subgraph READ["Nodos de lectura: muchos operadores independientes"]
        R1["Operador A<br/>nodo completo + indexador ligero"]
        R2["Operador B<br/>nodo completo + indexador ligero"]
        R3["Operador C<br/>nodo completo + indexador ligero"]
    end
    subgraph NET["Red Radix"]
        VAL[("Validadores<br/>perfil pequeño: R1-R4")]
    end
    subgraph SELF["Servicios auxiliares alojables por cualquiera"]
        SIG["Señalización y TURN propios"]
        RELAY["Relés de Radix Connect<br/>e iroh propios"]
        PRICE["Servicios de precios<br/>configurables"]
    end
    ARCH[("Archivo verificable de Olympia<br/>IPFS, BitTorrent, Arweave")]
    APPS -- "1. leen la lista" --> REG
    APPS -- "2. consultan con conmutación por error" --> READ
    READ --> VAL
    APPS -.-> SELF
    EXP -.-> ARCH
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class READ,REG,ARCH,SELF good
```

```mermaid
flowchart LR
    C["Wallets, dApps, ROLA y exploradores"]
    REG["Registro en la cadena<br/>metadatos api_endpoints<br/>de los validadores"]
    C -- "1. leen la lista de nodos de lectura" --> REG
    C -- "2. consultan con conmutación por error" --> A
    C --> B
    C --> Z
    subgraph A["Nodo de lectura A"]
        A1[("Nodo completo")] --- A2["Indexador ligero<br/>~10-50 GB"]
    end
    subgraph B["Nodo de lectura B"]
        B1[("Nodo completo")] --- B2["Indexador ligero"]
    end
    subgraph Z["Nodo de lectura ..."]
        Z1[("Nodo completo")] --- Z2["Indexador ligero"]
    end
    A1 --> R(("Red Radix"))
    B1 --> R
    Z1 --> R
```

## Cómo funciona un nodo de lectura

```mermaid
flowchart TB
    subgraph NODO["Nodo completo (sin cambios)"]
        CORE["Core API y Engine State API<br/>solo en 127.0.0.1"]
        LEDGER[("Ledger completo:<br/>transacciones, recibos, estado")]
        CORE --- LEDGER
    end
    subgraph IDX["Indexador ligero (proceso aparte)"]
        ING["Ingesta del stream<br/>desde la versión 1, génesis incluida"]
        IX[("Índices pequeños:<br/>historial por entidad, ubicación de NFT,<br/>titulares, subintents, reglas de rol,<br/>estadísticas, búsqueda")]
        API["API de lectura pública<br/>CORS, límite de peticiones, TLS"]
        RES["Reenvío de transacciones<br/>pendientes con espera exponencial"]
        ING --> IX
    end
    CORE -- "/stream/transactions" --> ING
    APP["Wallet, dApp o explorador"] --> API
    API -- "estado actual: saldos, NFT, metadatos" --> CORE
    API -- "historial, titulares, ubicación de NFT" --> IX
    API -- "enviar transacción" --> RES
    RES --> CORE
```

## Envío de una transacción sin Gateway

```mermaid
sequenceDiagram
    autonumber
    participant W as Wallet
    participant L as Nodo de lectura
    participant N as Nodo completo
    W->>L: preview de la transacción
    L->>N: /transaction/preview
    N-->>W: resultado y comisiones
    W->>W: el usuario firma
    W->>L: enviar transacción
    L->>N: /lts/transaction/submit
    loop mientras la época sea válida
        L->>N: /lts/transaction/status
        alt no confirmada
            L->>N: reenvío con espera exponencial
        end
    end
    W->>L: ¿estado?
    L-->>W: CommittedSuccess
```

**Estrategia: E3 primero, E2 después y E6 como complemento.** Hay que añadir dos piezas transversales:

- **(D) descubrimiento en la cadena:** que los validadores publiquen sus URLs de acceso en los metadatos de su componente, por ejemplo con la clave `api_endpoints`;
- **(L) abstracción `LedgerBackend`** en los clientes.

**Por qué:**

1. **Tiempo.** El Gateway está financiado solo hasta el 31-dic-2026. E3 es la única opción que puede **dar servicio a los wallets actuales sin publicar versiones nuevas**, porque ya permiten configurar la URL del Gateway.
2. **Coste.** Pasa de los ~2,5 TB y ~300-400 €/mes que requiere el Gateway a ~10-50 GB (estimación) añadidos a un nodo que ya existe. Los datos medidos lo explican: ~97-98 % de las versiones de estado son cambios de ronda que no hace falta indexar.
3. **Descentralización real.** Si cada validador o nodo completo puede servir la API con coste casi nulo y anunciarse en la cadena, desaparece el «proveedor por defecto». Es la condición para que la red dé servicio a cualquier usuario sin depender de un proveedor concreto.
4. **Camino de evolución.**
   - Los índices de E3 se integran luego en `babylon-node` (E2).
   - La interfaz `LedgerBackend` de los clientes permite pasar después a la API nativa del nodo, al cliente ligero (E5) y, si se adopta, a Hyperscale (sección [11](11-evolucion-hyperscale.md#s11)), sin rehacer la lógica de las aplicaciones.
5. **Riesgo controlado.** No toca el consenso ni el núcleo del nodo. Es un proceso aparte que se puede detener o reiniciar sin afectar al nodo.

**Lo que E3 no da, y cómo se cubre:**

- **Estado histórico arbitrario** (`at_ledger_state` antiguo):
  - E6, con captura hacia delante en consultation;
  - o una opción de «nodo de archivo» en E2, con historia larga y `ByTimestamp`.
- **Confianza en el nodo:** consultar a varios nodos descubiertos en la cadena y comparar `state_version` y el hash del acumulador (`/stream/proofs`). Con el tiempo, E5.

<a id="s6-1"></a>

## 6.1 Impacto en el almacenamiento de los validadores

**Respuesta corta:** no aumenta de forma considerable. Y los validadores **no tienen por qué ejecutarlo**.

1. **Es opcional y separado del validador.**
   - El indexador lo ejecuta solo quien quiera servir lecturas al público.
   - Es aconsejable que un validador **no** atienda tráfico público, para no exponerlo a ataques ni a carga que afecte al consenso. Las lecturas las sirven nodos completos (no validadores) operados por la comunidad, proveedores de wallets, dApps o los propios operadores de validadores en una máquina aparte.
   - Un validador que no lo ejecute **no ve ningún cambio** en su almacenamiento.
   - En E2, los índices dentro del nodo serían flags **desactivados por defecto**.
2. **Si un operador lo ejecuta en la misma máquina que su nodo:**

   | Concepto | Tamaño |
   |---|---|
   | Disco recomendado para un nodo (documentación oficial, «inicial») | 500 GB |
   | Indexador ligero (E3), estimado | ~10-50 GB |
   | Aumento relativo | **~2-10 %** del disco recomendado |
   | Crecimiento del indexador | ~0,1-0,3 GB/mes al ritmo actual (~4 400 transacciones de usuario/día × 0,5-1 KB, más índices auxiliares); ~1 GB/mes en el pico de marzo de 2025 (~35 000/día) |
   | Comparación: Gateway actual | ~2,5 TB, **+60 GB/mes** |

   **Comparación por punto de acceso público** (lo que necesita quien sirve datos a wallets, dApps y exploradores):

   | | Hoy, con el Gateway | Con el indexador ligero |
   |---|---|---|
   | Nodo completo (ledger completo: transacciones, recibos y estado; lo necesitan los dos modelos) | Sí, uno o varios: el DataAggregator lee de nodos completos | Sí, uno |
   | Además | Base de datos PostgreSQL de ~2,5 TB (+60 GB/mes) | Índices de ~10-50 GB (estimado) |
   | Total adicional sobre el nodo | **~2,5 TB** | **~10-50 GB** |

   Es decir: un nodo completo **ya guarda hoy todos los datos de la red**, porque lo necesita para sincronizarse y validar, y eso no cambia. El indexador ligero **solo añade sus índices sobre los datos del nodo**: un poco más que un nodo solo (~2-10 %) y mucho menos que un nodo con Gateway (entre 50 y 250 veces menos; estimación).

   El indexador ocupa poco porque **no duplica el estado ni las transacciones**, que ya están en el nodo. Solo guarda punteros (entidad → `state_version`) y pocas tablas pequeñas, y descarta los cambios de ronda, que son ~97-98 % de las versiones.
3. **Margen de error.**
   - Solo se conoce el tamaño **comprimido** del snapshot comunitario (274 GiB, sep-2026); no se ha publicado el tamaño en disco de un nodo de mainnet, y la documentación solo da los 500 GB «iniciales».
   - Los 10-50 GB del indexador son una **estimación** a partir de las mediciones de transacciones de usuario. El índice de titulares de recursos (h) y el de ubicación de NFT (b) dependen del número de pares recurso-titular y de NFT existentes, que no se ha medido.
   - El primer prototipo debe medir ambos antes de comprometer cifras.
4. **Lo que sí aumentaría mucho:** el «nodo de archivo» con historia completa de substates (N5), que es en esencia lo que hace que el Gateway ocupe 2,5 TB. Por eso el informe **no** lo propone para validadores. Sería una opción solo para quien necesite estado histórico arbitrario. Para gobernanza se sustituye por la captura del snapshot al abrir la votación (E6).

<a id="s6-2"></a>

## 6.2 La mejor solución con tiempo ilimitado: lecturas verificables desde cualquier nodo

> [!IMPORTANT]
> - **¿Es posible?** Sí, sin cambiar el consenso (los índices verificables sí requieren una actualización de protocolo).
> - **Dificultad:** Muy alta.
> - **Esfuerzo y plazo:** ~40-80 personas-mes adicionales; años (estimación).

La solución a corto plazo (E3) elimina el Gateway, pero el cliente **sigue teniendo que confiar** en el nodo que le responde, igual que hoy confía en el Gateway. Con tiempo ilimitado, la solución ideal elimina también esa confianza: **cualquier nodo, incluso uno desconocido, puede servir datos, y el wallet comprueba por sí mismo que son correctos.**

**Dato clave, comprobado en el código del nodo:** Babylon ya tiene casi todo lo necesario.

- Las pruebas del ledger que firman los validadores incluyen tres raíces de árboles de Merkle (`LedgerHashes`, `core-api-schema.yaml:1846-1870`):
  - `state_tree_hash` (todo el estado);
  - `transaction_tree_hash` (todas las transacciones);
  - `receipt_tree_hash` (todos los resultados).
- El nodo mantiene el árbol de estado (`state_hash_tree.*`).
- `/stream/proofs` ya entrega las pruebas firmadas, incluidos los cambios de época, que dicen qué validadores firman la época siguiente.
- **Lo que falta** es un endpoint que devuelva la **prueba de Merkle** de un dato concreto, y un cliente que la verifique.

## Infraestructura a largo plazo: solución ideal

```mermaid
flowchart TB
    subgraph APPS["Usuarios y aplicaciones con cliente ligero"]
        W["Wallets<br/>verifican cada respuesta"]
        DAPP["dApps, ROLA y exploradores<br/>cliente ligero en WebAssembly"]
    end
    subgraph NET["Red Radix: un solo binario en Rust"]
        V[("Validadores<br/>4 GB RAM, nodo podado")]
        N1[("Nodo con API de lectura nativa<br/>índices + pruebas de Merkle")]
        N2[("Otro nodo cualquiera<br/>incluso anónimo")]
        A[("Nodos de archivo<br/>historial completo")]
    end
    SNAP[("Snapshots de estado verificables<br/>IPFS, BitTorrent")]
    HIST[("Historial antiguo verificable<br/>Babylon y Olympia")]
    INC["Incentivos por servir lecturas<br/>decididos por gobernanza"]
    W -- "QUIC directo + pruebas" --> N1
    W -- "si uno falla, otro" --> N2
    DAPP -- "QUIC o HTTPS + pruebas" --> N1
    N1 --- V
    N2 --- V
    A -- "genera" --> SNAP
    SNAP -- "arranque en horas" --> N2
    A -- "publica" --> HIST
    INC -.-> N1
    INC -.-> N2
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class W,DAPP,V,N1,N2,A,SNAP,HIST good
```

### Cómo verifica el wallet una respuesta

```mermaid
sequenceDiagram
    autonumber
    participant W as Wallet con cliente ligero
    participant Q as Cualquier nodo, incluso desconocido
    participant V as Validadores
    V-->>Q: firman la prueba del ledger con state_tree_hash
    W->>Q: dame el saldo de mi cuenta
    Q-->>W: dato + prueba de Merkle + prueba del ledger firmada
    W->>W: ¿firmó más de 2/3 del stake de la época?
    W->>W: ¿la prueba de Merkle cuadra con state_tree_hash?
    alt las dos comprobaciones son correctas
        W->>W: muestra el saldo, sin confiar en el nodo
    else alguna falla
        W->>Q: descarta el nodo y pregunta a otro
    end
```

**Componentes de la solución ideal:**

| # | Componente | Qué hace |
|---|---|---|
| 1 | **Cada nodo es un nodo de lectura** (E2) | Los índices van dentro de `babylon-node` como opciones. Una **API de lectura nativa**, abierta y versionada, sustituye al formato heredado. No hay ningún proceso aparte. |
| 2 | **Lecturas con prueba** | Cada respuesta incluye el dato, su prueba de Merkle contra `state_tree_hash`, `transaction_tree_hash` o `receipt_tree_hash`, y la prueba del ledger firmada por los validadores. Sirve para saldos, NFT, metadatos, estado de componentes, transacciones y recibos. No cambia el consenso. |
| 3 | **Cliente ligero en los wallets y dApps** | En Sargon (Rust) y en WebAssembly para navegador (RDT, ROLA, exploradores). Sigue los cambios de época desde un punto de control incluido en la app y comprueba que más de 2/3 del stake firmó cada prueba. Con eso verifica cada respuesta. **El cliente deja de confiar en el nodo.** |
| 4 | **Índices verificables** | Una prueba de Merkle demuestra que un dato *está*, pero no que una lista (historial, titulares) esté *completa*. Para eso hace falta una actualización de protocolo: que el motor guarde en el estado, para cada entidad, un acumulador de su historial y de sus titulares. Así entran bajo `state_tree_hash` y también se pueden demostrar. Mientras tanto, se contrastan varios nodos independientes. |
| 5 | **Descubrimiento y transporte sin intermediarios** | Registro de nodos de lectura en la cadena (metadatos de validadores u otro componente) y conexión directa por QUIC/iroh entre wallet y nodo. Esto evita depender de DNS, de certificados TLS y de relés centrales; la misma tecnología ya se usa en Radix Connect. |
| 6 | **Incentivos para servir lecturas** | Una parte de las emisiones o comisiones de la red, o micropagos en XRD por consulta, para quien opere nodos de lectura. Lo decide la gobernanza. Sin incentivos, el servicio depende del compromiso voluntario de los operadores. |
| 7 | **Archivo histórico verificable** | La historia antigua (incluido Olympia) se guarda en almacenamiento distribuido (IPFS o Arweave). Su integridad se comprueba contra `transaction_tree_hash` y `receipt_tree_hash`. Ningún nodo necesita guardarla entera. |
| 8 | **Preparado para Hyperscale** | Hyperscale ya usa multipruebas JMT. El cliente ligero y la API nativa se pueden adaptar a esa red. La migración completa a Hyperscale se analiza en la sección [11](11-evolucion-hyperscale.md#s11). |

**Comparación con la solución a corto plazo:**

| | Corto plazo (E3 → E2) | Ideal ([6.2](#s6-2)) |
|---|---|---|
| Gateway | Eliminado | Eliminado |
| ¿El cliente confía en el nodo? | Sí, aunque puede comparar varios | **No, verifica cada respuesta** |
| ¿Quién puede servir datos? | Operadores de nodo conocidos | **Cualquier nodo, incluso anónimo** |
| Cambios de protocolo | Ninguno | Solo para los índices verificables (punto 4) |
| Plazo | Meses | **Años** |
| Esfuerzo (est.) | 7-12 personas-mes el mínimo viable; 21-35 el completo | **~40-80 personas-mes** adicionales, con perfiles expertos en el protocolo |

**Por qué es la mejor con tiempo ilimitado:** es la única opción en la que la red no depende de nadie, ni de un Gateway ni de la honestidad de un nodo. Es el mismo nivel de garantía que da la propia cadena, extendido hasta el wallet de cada usuario.

```mermaid
flowchart LR
    H["Hoy<br/>Gateway central"] --> S1["Etapa 1, meses<br/>E3: indexador ligero<br/>junto a cada nodo"]
    S1 --> S2["Etapa 2<br/>E2: índices dentro<br/>de babylon-node"]
    S2 --> S3["Etapa 3, años<br/>Lecturas verificables<br/>y cliente ligero"]
    classDef bad fill:#ffd6d6,stroke:#c00,color:#000
    classDef good fill:#d9f7d9,stroke:#080,color:#000
    class H bad
    class S3 good
```

**Cómo encaja con el corto plazo:** no son alternativas, son etapas. E3 resuelve el problema de 2026. E2 elimina el proceso aparte. Los puntos 2-8 añaden la verificación sobre lo que E2 ya sirve. La interfaz `LedgerBackend` de los clientes es la que permite pasar de una etapa a otra sin rehacer las aplicaciones.

<a id="s6-3"></a>

## 6.3 Riesgos de eliminar el Gateway y cómo mitigarlos

| Riesgo | Consecuencia | Mitigación |
|---|---|---|
| **Un nodo devuelve datos incorrectos**, por error o de forma malintencionada | El usuario ve saldos o estados falsos | Consultar varios puntos de acceso y comparar `state_version` y el hash del acumulador ([10.2](10-guia-de-implementacion.md#s10-2)); a largo plazo, lecturas verificables ([6.2](#s6-2)) |
| **Pocos operadores** de nodos de lectura | Vuelve a haber una dependencia central | Coste casi nulo por operador (E3); descubrimiento en la cadena; incentivos decididos por la gobernanza ([6.2](#s6-2), punto 6; capítulo [12](12-referencias-otras-redes.md#s12)) |
| **Ataques de denegación de servicio** contra los nodos de lectura | Servicio lento o caído | Modo público con límites por IP y lista blanca de rutas (N6); varios operadores; validadores separados de los nodos de lectura ([6.1](#s6-1)) |
| **El indexador no reproduce exactamente las respuestas** que esperan los wallets | Errores en las aplicaciones | Pruebas de conformidad contra el Gateway público **antes de que se apague** ([10.1](10-guia-de-implementacion.md#s10-1)) |
| **Pocos nodos de archivo** | Los nodos nuevos no pueden obtener el historial ni snapshots | Varios nodos de archivo en distintas manos; snapshots verificables en almacenamiento distribuido (R9) |
| **Se pierde una transacción** si el punto de acceso se reinicia | El usuario cree que envió algo que nunca se confirmó | Reenvío en el indexador y en los clientes ([10.1](10-guia-de-implementacion.md#s10-1), [10.4](10-guia-de-implementacion.md#s10-4)) |
| **Falta de financiación o de desarrolladores** | El plan se retrasa más allá del 31-dic-2026 | Priorizar el mínimo viable (7-12 personas-mes); decisión temprana de la gobernanza |
| **Errores en el software nuevo** (indexador, cambios en el nodo) | Fallos de servicio o, en el nodo, de consenso | Desplegar primero en Stokenet; opciones desactivadas por defecto; adopción gradual ([10.7](10-guia-de-implementacion.md#s10-7)) |

---

[← Índice](../../README.es.md) · [Anterior: Capítulos 4-5: las seis estrategias evaluadas y su comparativa](04-05-estrategias.md) · [Siguiente: Capítulo 7: cambios concretos en cada repositorio](07-cambios-por-repositorio.md)
