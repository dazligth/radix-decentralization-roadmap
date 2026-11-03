# Cómo contribuir

Español · [English](CONTRIBUTING.md)

Este repositorio es de código abierto y está pensado para que la comunidad de Radix lo mejore. Cualquier persona puede proponer cambios mediante *issues* y *pull requests* (PR).

## Qué se puede aportar

| Tipo de aportación | Ejemplos | Dónde |
|---|---|---|
| **Correcciones** | Un dato erróneo, una referencia `fichero:línea` desactualizada, un enlace roto, una errata, una traducción imprecisa | PR sobre el documento afectado en `docs/es/` y `docs/en/` |
| **Datos y mediciones** | Tamaño real del ledger por *column family* en un nodo de mainnet, consumo de RAM medido, tiempo de sincronización, tamaño de los índices de un prototipo | PR que actualice la cifra y cite la medición (método, fecha, versión del nodo) |
| **Nuevas estrategias** | Una alternativa al indexador ligero, otra forma de migrar a Hyperscale, otro modelo de incentivos | Primero un *issue* con la plantilla «Nueva estrategia»; después, una nota en `community/` o un PR al capítulo correspondiente |
| **Ideas y notas de la comunidad** | Opiniones, propuestas, experiencias de operadores, dudas razonadas | Un fichero en `community/` (ver [community/README.md](community/README.md)) |
| **Implementación** | Enlaces a prototipos, repositorios o PR en `babylon-node`, Sargon, la RDT, etc. que avancen alguna de las propuestas | PR que añada el enlace en la sección correspondiente y en la hoja de ruta |
| **Diagramas** | Mejoras o diagramas nuevos en Mermaid | PR sobre el documento, comprobando que GitHub los muestra correctamente |

## Cómo hacer un pull request

1. Hacer un *fork* del repositorio y crear una rama con un nombre descriptivo (por ejemplo, `correccion/tamano-ledger`).
2. Editar los ficheros:
   - **Contenido del informe:** `docs/es/` y `docs/en/`. Los resúmenes y la conclusión están en `README.es.md` y `README.md`.
   - **Notas de la comunidad:** `community/`.
3. Mantener **los dos idiomas** sincronizados. Quien solo pueda escribir en uno debe indicarlo en el PR con la etiqueta «necesita traducción» y otra persona lo completará.
4. Abrir el PR con la plantilla y explicar qué cambia y por qué.
5. Los cambios de contenido se revisan antes de incorporarse. Se pueden pedir fuentes o ajustes.

## Reglas de calidad

El valor de este informe depende de que todo lo que dice sea comprobable. Por eso:

- **No se inventa nada.** Cada dato debe tener una fuente: `fichero:línea` y versión del código, una URL o una medición descrita (método, fecha, versión).
- **Las estimaciones se marcan como tales** («estimación», «est.») y se explica de dónde salen.
- **Las valoraciones usan el mismo formato** al principio de cada apartado:

  ```markdown
  > [!IMPORTANT]
  > - **¿Es posible?** Sí / Sí, con condiciones / No / Solo en parte.
  > - **Dificultad:** Baja / Baja-media / Media / Media-alta / Alta / Muy alta.
  > - **Esfuerzo y plazo:** ... (estimación).
  ```

- **Registro:** claro y neutral, comprensible para personas no técnicas, sin jerga innecesaria ni tono coloquial. Redacción impersonal.
- **Sin emojis** en el texto.
- **Diagramas en Mermaid**, para que se puedan editar como texto.
- Si un cambio afecta a otras secciones (por ejemplo, un esfuerzo que aparece también en la conclusión), se actualizan todas.

## Discrepancias y debate

Quien no esté de acuerdo con una conclusión o una estrategia puede abrir un *issue* explicando el motivo y aportando datos. Las posiciones distintas razonadas son bienvenidas. Si no hay consenso, se puede recoger como nota de la comunidad en `community/` y enlazarla desde la sección correspondiente.

## Licencia

Las aportaciones se publican bajo la licencia del repositorio (ver `LICENSE`).
