[← Índice](../../README.es.md) · [Anterior: Sección 7: cambios concretos en cada repositorio](05-cambios-por-repositorio.md) · [Siguiente: Sección 9: nodos más ligeros, 4 GB de RAM y 500 GB de disco, migración de Java a Rust](07-nodos-ligeros.md)

# 8. Hoja de ruta

```mermaid
gantt
    title Hoja de ruta orientativa
    dateFormat YYYY-MM-DD
    axisFormat %b %Y
    section Urgente
    Archivo de Olympia                       :crit, 2026-11-05, 45d
    Prototipo del indexador ligero           :crit, 2026-11-05, 55d
    Medir disco y RAM; R1-R2 RocksDB y perfil 4 GB :crit, 2026-11-10, 50d
    Fin de la financiación del Gateway       :milestone, 2026-12-31, 0d
    section Paridad
    Índices completos y exploradores         :2027-01-01, 60d
    Clientes con failover y descubrimiento   :2027-02-01, 90d
    section Nodo
    Índices dentro de babylon-node           :2027-03-01, 270d
    R3-R8 poda y nodo podado                 :2027-01-15, 150d
    R9 arranque desde snapshot verificado    :2027-04-01, 180d
    R10 actualización de protocolo           :2027-07-01, 120d
    R11 migración de Java a Rust             :2027-03-01, 600d
    section Ideal
    Lecturas verificables y cliente ligero   :2027-09-01, 540d
```

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

| Fase | Fechas orientativas | Contenido | Resultado |
|---|---|---|---|
| **F0** | nov.-dic. 2026 | **Archivo verificable del ledger de Olympia** (2.2), antes de que se apague. **Medir el tamaño por column family de un nodo de mainnet.** R1-R2 (RocksDB y perfil de 4 GB). Prototipo de `radix-lite-indexer` con el subconjunto crítico: estado, envío, preview, saldos, NFT, metadatos, subintents, validadores. Perfil en `babylon-nodecli`. Pruebas de conformidad contra el Gateway público. | Wallets, RDT y ROLA actuales funcionan cambiando solo la URL |
| **F1** | ene.-feb. 2027 | Índices de historial, ubicación de NFT y reglas de rol. Índices para exploradores (titulares, estadísticas, búsqueda). Reenvío. Modo público. Snapshots. Convención `api_endpoints` en los validadores. Primera versión de `radix-explorer`. | Paridad con lo que usan los clientes. Exploradores sin Gateway. Varios operadores de nodos sirviendo datos con el indexador. |
| **F2** | feb.-abr. 2027 | Clientes: listas con failover y descubrimiento (Sargon, wallets, RDT, ROLA, extensión, SDK). Reenvío en Sargon. Correcciones (`BaseUrlInterceptor`, mainnet fijo, servicios fijos). | Sin proveedor por defecto central |
| **F3** | 2027 | `babylon-node`: N1, N6, N5b, después N2-N4. R3-R8 (poda y nodo podado) y R9 (arranque desde snapshot verificado). Captura de snapshots en consultation. | Cada nodo es un punto de acceso; el indexador pasa a ser opcional |
| **F4** | 2027-2029 | Migración del núcleo a Rust (R11) por fases. Actualización de protocolo R10. Lecturas verificables y cliente ligero (6.2, 10.6). | Nodos de 4 GB con margen; acceso sin confiar en el nodo |

---

[← Índice](../../README.es.md) · [Anterior: Sección 7: cambios concretos en cada repositorio](05-cambios-por-repositorio.md) · [Siguiente: Sección 9: nodos más ligeros, 4 GB de RAM y 500 GB de disco, migración de Java a Rust](07-nodos-ligeros.md)
