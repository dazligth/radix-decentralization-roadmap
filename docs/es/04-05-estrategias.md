[← Índice](../../README.es.md) · [Anterior: Capítulo 3: qué ofrece hoy el nodo y qué le falta](03-que-ofrece-el-nodo.md) · [Siguiente: Capítulo 6: solución recomendada, almacenamiento, solución ideal y riesgos](06-solucion-recomendada.md)

<a id="s4"></a>

# 4. Estrategias evaluadas

## E1. Clientes directos contra la Core API y la Engine State API, sin tocar el nodo

> [!IMPORTANT]
> - **¿Es posible?** Solo en parte: cubre 9 de las 19 capacidades de la tabla 3.2 por completo y 2 en parte (las más habituales), pero no el resto.
> - **Dificultad:** Media-alta.
> - **Esfuerzo y plazo:** 15-25 personas-mes en clientes (estimación).

- **Cobertura:** parcial (9 de 19 capacidades completas y 2 en parte, sección [3.2](03-que-ofrece-el-nodo.md#s3-2)). Pierde N1-N5: estado de pre-autorizaciones (N1), ubicación de NFT (N2), historial completo (N3), búsqueda inversa por rol (N4) y estado histórico (N5).
- **Cambios en clientes:** grandes. Todos los clientes necesitan un backend nuevo.
- **Coste para el operador:** nulo si ya tiene nodo. Necesita un proxy público.
- **Problema:** experiencia de usuario degradada y casos de uso que dejan de funcionar. **No cumple «todos los casos de uso».**

## E2. «Nodo completo de lectura»: índices y modo público dentro de `babylon-node`

> [!IMPORTANT]
> - **¿Es posible?** Sí.
> - **Dificultad:** Alta (toca el núcleo del nodo).
> - **Esfuerzo y plazo:** 6-10 personas-mes en el nodo + ~4-6 en clientes (estimación); plazo largo.

- **Cobertura:** 100 % de lo que usan los clientes hoy, salvo la historia arbitraria de substates (ver E3).
- **Cambios en clientes:** muchos, como en E1, salvo que el nodo responda también en el formato de API que usan hoy los wallets; en ese caso no hacen falta para funcionar.
- **Coste por operador:** marginal (índices opcionales).
- **Descentralización:** máxima. Cualquier nodo completo puede servir lecturas (se desaconseja en los validadores, sección [6.1](06-solucion-recomendada.md#s6-1)).
- **Problema:**
  - Requiere desarrollo en Java y Rust dentro del nodo y una publicación del nodo.
  - La Foundation está en mantenimiento, así que depende de desarrolladores de la comunidad y de la gobernanza.
  - Plazo largo.

## E3. Indexador ligero junto al nodo, que sustituye al Gateway

> [!IMPORTANT]
> - **¿Es posible?** Sí (sin historia arbitraria del estado).
> - **Dificultad:** Media.
> - **Esfuerzo y plazo:** 4-7 personas-mes (+1-2 para exploradores); meses (estimación).

Es un proceso aparte, en Rust y con RocksDB o SQLite. Funciona así:

- Consume `/core/stream/transactions` del nodo local.
- Mantiene **solo los índices que el nodo no tiene** (detallados en la sección [7.1](07-cambios-por-repositorio.md#s7-1)):
  - (a) marcadores entidad → transacción, con cambios fungibles y de NFT, clase de manifiesto y eventos;
  - (b) NFT → vault → entidad global;
  - (c) subintents;
  - (d) índice inverso de reglas de rol;
  - (e) `last_updated` de metadatos;
  - (f) toques de lockers;
  - (g) emisiones de validadores;
  - (h), (i) y (j) para exploradores: titulares, estadísticas y búsqueda.
- **No contiene nada del software del Gateway:** ni DataAggregator, ni PostgreSQL, ni historia de substates. Solo conserva el **formato de la API, peticiones y respuestas** (el contrato OpenAPI), para que los wallets ya publicados funcionen sin actualizarse. Cuando los clientes adopten la API nativa del nodo (E2), esa capa de compatibilidad se puede quitar.
- Responde a las peticiones **con las mismas rutas y el mismo JSON que usan hoy los wallets y dApps** (los del capítulo [2](01-02-situacion-actual.md#s2), unas 25 rutas). Así las apps ya publicadas funcionan cambiando solo la URL. Es una cuestión de formato: detrás no hay ningún Gateway. Las lecturas de estado actual las delega en vivo al nodo (Core y Engine State API); solo usa sus índices para lo que el nodo no sabe.

Evaluación:

- **Cobertura:** ~100 % de lo que usan wallets, RDT, ROLA, extensión y SDK.
  - **No incluye** `at_ledger_state` arbitrario. Hoy el único uso relevante es el de consultation; Sargon lo usa para dar coherencia a lecturas paginadas, lo que se resuelve fijando la versión.
- **Cambios en clientes:** **ninguno para funcionar.** Los wallets ya permiten añadir una URL personalizada (Ajustes → Gateways) que apunte al indexador, y la RDT tiene `gatewayBaseUrl`. Hacen falta cambios menores para elegir el nodo automáticamente.
- **Tamaño:** **~10-50 GB**, frente a ~2,5 TB. Base del cálculo: ~10-15 millones de transacciones de usuario × ~0,5-1 KB por marcador (supuesto, a validar con un prototipo) dan ~5-15 GB; el resto del margen cubre las tablas de ubicación de NFT, titulares, búsqueda y auxiliares. Se descartan los ~555 millones de cambios de ronda. *Es una estimación que hay que validar con un prototipo.*
- **Primera sincronización:** recorrer ~567 millones de versiones del stream. Suponiendo un ritmo de ingesta de 2 000-5 000 versiones/s para un proceso sin Postgres (supuesto, a validar), son ~1,3-3,3 días. Se puede acortar con snapshots firmados. *Estimación.*
- **Coste por operador:** marginal sobre un nodo existente (disco y algo de CPU).
- **Descentralización:** alta. Cualquier operador de nodo puede ejecutarlo.
- **Problema:**
  - Hay que mantener ese formato de peticiones (un contrato OpenAPI estable) mientras haya apps que lo usen. Cuando los clientes adopten la API nativa del nodo, se puede retirar.
  - Confianza: el cliente confía en el nodo, igual que hoy confía en el Gateway.

## E4. Indexación en cada cliente

> [!IMPORTANT]
> - **¿Es posible?** No para el historial; solo como complemento para vigilar transacciones.
> - **Dificultad:** —
> - **Esfuerzo y plazo:** — (no es una solución por sí misma).

Cada wallet recorre el stream para construir su historial.

- **Datos:** 3,6-4,4 versiones/s continuas y ~567 millones de versiones de historia.
- **Viabilidad:** inviable en móvil para la historia; solo sirve para vigilar hacia delante durante una sesión.
- **Uso razonable:** únicamente como complemento. Por ejemplo, vigilar una transacción o un subintent desde la versión en que se envió.

## E5. Cliente ligero con pruebas criptográficas

> [!IMPORTANT]
> - **¿Es posible?** Sí, a largo plazo.
> - **Dificultad:** Muy alta.
> - **Esfuerzo y plazo:** ~40-80 personas-mes adicionales; varios años (estimación; ver la sección [6.2](06-solucion-recomendada.md#s6-2)).

- **Babylon:** hoy no hay un endpoint que dé pruebas de un dato concreto; `/stream/proofs` solo da las pruebas firmadas del ledger. Pero esas pruebas ya incluyen las raíces de Merkle del estado, las transacciones y los recibos, así que se puede construir sin cambiar el consenso (sección [6.2](06-solucion-recomendada.md#s6-2)).
- **Hyperscale:** tiene multipruebas JMT y cabeceras certificadas por QC, pero solo entre validadores, y no es compatible con Babylon.
- **Valor:** a largo plazo es la forma de **no tener que confiar** en el nodo, y forma parte de la solución ideal ([6.2](06-solucion-recomendada.md#s6-2)). No resuelve el problema de 2026.

## E6. Rediseño de datos en la cadena

> [!IMPORTANT]
> - **¿Es posible?** Solo en parte, aplicación por aplicación.
> - **Dificultad:** Baja-media por aplicación.
> - **Esfuerzo y plazo:** depende de cada aplicación.

Cambiar los blueprints para que la información esté en la cadena y no haya que indexarla. Ejemplos: registro de titulares, recuentos en el propio componente, eventos completos.

- **Alcance:** parcial. Solo para aplicaciones concretas, como consultation, y sin cambiar recursos ni cuentas nativas.
- **Valor:** útil como complemento, pero no es general.

<a id="s5"></a>

# 5. Comparativa con datos

Escala de puntuación: 1 = malo, 5 = muy bueno.

| Criterio | E1 Nodo directo | E2 Nodo de lectura | **E3 Indexador ligero** | E4 Cliente | E5 Cliente ligero | E6 En cadena |
|---|---|---|---|---|---|---|
| Cobertura de casos de uso | 2 (parcial: 9 de 19 completas y 2 en parte) | 5 | **5 (~100 %; sin historia arbitraria)** | 1 | 2 | 1-2 |
| Exploradores de transacciones completos | 1 (sin historial, titulares ni estadísticas) | 4 (si se añaden h, i, j) | **5 (con h, i, j)** | 1 | 1 | 1 |
| Elimina el Gateway como servicio central | 5 | 5 | **5** (no queda ningún Gateway; solo se reutiliza el formato de las peticiones) | 5 | 5 | 3 |
| Almacenamiento por operador | 0 extra | Decenas de GB (est.) | **~10-50 GB (est.)** | — | — | — |
| Coste mensual por operador | ~0 sobre el nodo | ~0 | **~0-20 € sobre el nodo (est.)** | — | — | — |
| Cambios necesarios en clientes | Muchos | Muchos (ninguno si el nodo sirve también el formato actual) | **Ninguno para funcionar** | Muchos | Muchos | Por dApp |
| ¿Disponible antes de diciembre de 2026? | Parcial | No | **Prototipo con el subconjunto crítico: posible** | No | No | No |
| Trabajo de desarrollo (est.) | 15-25 personas-mes | 6-10 personas-mes (nodo) + ~4-6 (clientes) | **4-7 personas-mes** (+1-2 para índices de exploradores) | — | ~40-80 personas-mes; años | Por dApp |
| Riesgo técnico | Medio | Medio-alto (núcleo del nodo) | **Medio** | Alto | Muy alto | Bajo |
| Válido con Xi'an / Hyperscale | No | No | Parcial (interfaz reutilizable) | — | Parcial (habría que adaptarlo) | No (habría que reescribirlos) |

**Lectura de la tabla:**

- **E1** no cumple «todos los casos de uso».
- **E2** es el destino a medio plazo, pero es lento de desarrollar y toca el núcleo del nodo.
- **E3** da cobertura completa con el coste de operación de E1, sin esperar a que los wallets publiquen versiones nuevas. Además, su código de índices es el que después se integra en el nodo (E2).

---

[← Índice](../../README.es.md) · [Anterior: Capítulo 3: qué ofrece hoy el nodo y qué le falta](03-que-ofrece-el-nodo.md) · [Siguiente: Capítulo 6: solución recomendada, almacenamiento, solución ideal y riesgos](06-solucion-recomendada.md)
