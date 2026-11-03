[← Índice](../../README.es.md) · [Anterior: Sección 3: qué ofrece hoy el nodo y qué le falta](02-que-ofrece-el-nodo.md) · [Siguiente: Sección 6: solución recomendada, almacenamiento y solución ideal a largo plazo](04-solucion-recomendada.md)

# 4. Estrategias evaluadas

## E1. Clientes directos contra la Core API y la Engine State API, sin tocar el nodo

> [!IMPORTANT]
> - **¿Es posible?** Solo en parte: cubre ~80-85 % de los casos de uso (estimación).
> - **Dificultad:** Media-alta.
> - **Esfuerzo y plazo:** 15-25 persona-mes en clientes (estimación).

- **Cobertura:** ~80-85 %. Pierde N1-N5: historial completo, ubicación de NFT, búsqueda inversa por rol, snapshots y estado de pre-autorizaciones.
- **Cambios en clientes:** grandes. Todos los clientes necesitan un backend nuevo.
- **Coste para el operador:** nulo si ya tiene nodo. Necesita un proxy público.
- **Problema:** experiencia de usuario degradada y casos de uso que dejan de funcionar. **No cumple «todos los casos de uso».**

## E2. «Nodo completo de lectura»: índices y modo público dentro de `babylon-node`

> [!IMPORTANT]
> - **¿Es posible?** Sí.
> - **Dificultad:** Alta (toca el núcleo del nodo).
> - **Esfuerzo y plazo:** 6-10 persona-mes en el nodo + clientes (estimación); plazo largo.

- **Cobertura:** 100 % de lo que usan los clientes hoy, salvo la historia arbitraria de substates (ver E3).
- **Cambios en clientes:** muchos, como en E1, salvo que el nodo responda también en el formato de API que usan hoy los wallets; en ese caso no hacen falta para funcionar.
- **Coste por operador:** marginal (índices opcionales).
- **Descentralización:** máxima. Cualquier nodo, incluidos los 100 validadores activos, puede servir.
- **Problema:**
  - Requiere desarrollo en Java y Rust dentro del nodo y una publicación del nodo.
  - La Foundation está en mantenimiento, así que depende de desarrolladores de la comunidad y de la gobernanza.
  - Plazo largo.

## E3. Indexador ligero junto al nodo, que sustituye al Gateway

> [!IMPORTANT]
> - **¿Es posible?** Sí (sin historia arbitraria del estado).
> - **Dificultad:** Media.
> - **Esfuerzo y plazo:** 4-7 persona-mes (+1-2 para exploradores); meses (estimación).

Es un proceso aparte, en Rust y con RocksDB o SQLite. Funciona así:

- Consume `/core/stream/transactions` del nodo local.
- Mantiene **solo los índices que el nodo no tiene**:
  - marcadores entidad → transacción, con cambios fungibles y de NFT, clase de manifiesto y eventos;
  - NFT → vault → entidad global;
  - subintents;
  - índice inverso de reglas de rol;
  - `last_updated` de metadatos;
  - toques de lockers;
  - emisiones de validadores.
- **No contiene nada del software del Gateway:** ni DataAggregator, ni PostgreSQL, ni historia de substates. Solo conserva el **formato de la API, peticiones y respuestas** (el contrato OpenAPI), para que los wallets ya publicados funcionen sin actualizarse. Cuando los clientes adopten la API nativa del nodo (E2), esa capa de compatibilidad se puede quitar.
- Responde a las peticiones **con las mismas rutas y el mismo JSON que usan hoy los wallets y dApps** (los de la sección 2, unas 25 rutas). Así las apps ya publicadas funcionan cambiando solo la URL. Es una cuestión de formato: detrás no hay ningún Gateway. Las lecturas de estado actual las delega en vivo al nodo (Core y Engine State API); solo usa sus índices para lo que el nodo no sabe.

Evaluación:

- **Cobertura:** ~100 % de lo que usan wallets, RDT, ROLA, extensión y SDK.
  - **No incluye** `at_ledger_state` arbitrario. Hoy el único uso relevante es el de consultation; Sargon lo usa para dar coherencia a lecturas paginadas, lo que se resuelve fijando la versión.
- **Cambios en clientes:** **ninguno para funcionar.** Los wallets ya permiten añadir una URL personalizada (Ajustes → Gateways) que apunte al indexador, y la RDT tiene `gatewayBaseUrl`. Hacen falta cambios menores para elegir el nodo automáticamente.
- **Tamaño:** **~10-50 GB**, frente a ~2,5 TB. Base del cálculo: ~10-15 millones de transacciones de usuario × ~0,5-1 KB por marcador, más la tabla de ubicación de NFT. Se descartan los ~555 millones de cambios de ronda. *Es una estimación que hay que validar con un prototipo.*
- **Primera sincronización:** recorrer ~567 millones de versiones del stream. Al ritmo de ingesta de 2 000-5 000 versiones/s de un proceso sin Postgres, son ~1,5-3,5 días. Se puede acortar con snapshots firmados. *Estimación.*
- **Coste por operador:** marginal sobre un nodo existente (disco y algo de CPU).
- **Descentralización:** alta. Cualquier operador de nodo puede ejecutarlo.
- **Problema:**
  - Hay que mantener ese formato de peticiones (un contrato OpenAPI estable) mientras haya apps que lo usen. Cuando los clientes adopten la API nativa del nodo, se puede retirar.
  - Confianza: el cliente confía en el nodo, igual que hoy confía en el Gateway.

## E4. Indexación en cada cliente

> [!IMPORTANT]
> - **¿Es posible?** No para el historial; solo como complemento para vigilar transacciones.
> - **Dificultad:** No aplicable como solución.

Cada wallet recorre el stream para construir su historial.

- **Datos:** 3,6-4,4 versiones/s continuas y ~567 millones de versiones de historia.
- **Viabilidad:** inviable en móvil para la historia; solo sirve para vigilar hacia delante durante una sesión.
- **Uso razonable:** únicamente como complemento. Por ejemplo, vigilar una transacción o un subintent desde la versión en que se envió.

## E5. Cliente ligero con pruebas criptográficas

> [!IMPORTANT]
> - **¿Es posible?** Sí, a largo plazo.
> - **Dificultad:** Muy alta.
> - **Esfuerzo y plazo:** Varios años.

- **Babylon:** hoy no hay un endpoint que dé pruebas de un dato concreto; `/stream/proofs` solo da las pruebas firmadas del ledger. Pero esas pruebas ya incluyen las raíces de Merkle del estado, las transacciones y los recibos, así que se puede construir sin cambiar el consenso (sección 6.2).
- **Hyperscale:** tiene multipruebas JMT y cabeceras certificadas por QC, pero solo entre validadores, y no es compatible con Babylon.
- **Valor:** a largo plazo es la forma de **no tener que confiar** en el nodo, y forma parte de la solución ideal (6.2). No resuelve el problema de 2026.

## E6. Rediseño de datos en la cadena

> [!IMPORTANT]
> - **¿Es posible?** Solo en parte, aplicación por aplicación.
> - **Dificultad:** Baja-media por aplicación.

Cambiar los blueprints para que la información esté en la cadena y no haya que indexarla. Ejemplos: registro de titulares, recuentos en el propio componente, eventos completos.

- **Alcance:** parcial. Solo para aplicaciones concretas, como consultation, y sin cambiar recursos ni cuentas nativas.
- **Valor:** útil como complemento, pero no es general.

# 5. Comparativa con datos

Escala de puntuación: 1 = malo, 5 = muy bueno.

| Criterio | E1 Nodo directo | E2 Nodo de lectura | **E3 Indexador ligero** | E4 Cliente | E5 Cliente ligero | E6 En cadena |
|---|---|---|---|---|---|---|
| Cobertura de casos de uso | 2 (~80-85 %) | 5 | **5 (~100 %; sin historia arbitraria)** | 1 | 2 | 1-2 |
| Exploradores de transacciones completos | 1 (sin historial, titulares ni estadísticas) | 4 (si se añaden h, i, j) | **5 (con h, i, j)** | 1 | 1 | 1 |
| Elimina el Gateway como servicio central | 5 | 5 | **5** (no queda ningún Gateway; solo se reutiliza el formato de las peticiones) | 5 | 5 | 3 |
| Almacenamiento por operador | 0 extra | Decenas de GB (est.) | **~10-50 GB (est.)** | — | — | — |
| Coste mensual por operador | ~0 sobre el nodo | ~0 | **~0-20 € sobre el nodo (est.)** | — | — | — |
| Cambios necesarios en clientes | Muchos | Muchos (ninguno si el nodo sirve también el formato actual) | **Ninguno para funcionar** | Muchos | Muchos | Por dApp |
| ¿Disponible antes de diciembre de 2026? | Parcial | No | **Prototipo con el subconjunto crítico: posible** | No | No | No |
| Trabajo de desarrollo (est.) | 15-25 persona-mes | 6-10 pm (nodo) + clientes | **4-7 pm** (+1-2 para índices de exploradores) | — | Años | Por dApp |
| Riesgo técnico | Medio | Medio-alto (núcleo del nodo) | **Medio** | Alto | Muy alto | Bajo |
| Válido con Xi'an / Hyperscale | No | No | Parcial (interfaz reutilizable) | — | Sí | Sí |

**Lectura de la tabla:**

- **E1** no cumple «todos los casos de uso».
- **E2** es el destino a medio plazo, pero es lento de desarrollar y toca el núcleo del nodo.
- **E3** da cobertura completa con el coste de operación de E1, sin esperar a que los wallets publiquen versiones nuevas. Además, su código de índices es el que después se integra en el nodo (E2).

---

[← Índice](../../README.es.md) · [Anterior: Sección 3: qué ofrece hoy el nodo y qué le falta](02-que-ofrece-el-nodo.md) · [Siguiente: Sección 6: solución recomendada, almacenamiento y solución ideal a largo plazo](04-solucion-recomendada.md)
