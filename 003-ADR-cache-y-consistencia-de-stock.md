# ADR-003: Gestión de stock en tiempo real — Redis como caché de lectura + descuento transaccional en PostgreSQL

## Status
ACCEPTED — 2026-07-03

## Context
El problema de negocio más citado en las entrevistas es el desfase de stock: "a las 10:00 de la mañana, el stock que ves en la web es el de las 6:00" (Javier Ortiz, Responsable de Almacén). FR-019 exige descuento de stock en el momento exacto del escaneo, y NFR-005 exige que ese descuento se propague al catálogo visible en menos de 5 segundos. Al mismo tiempo, NFR-001/NFR-002 exigen que el catálogo responda con p95 <500ms incluso a 3.000 pedidos/día, y NFR-012 exige que no haya sobreventa cuando dos clientes compiten por la última unidad (condición de carrera explícitamente descrita por el CTO).

## Decision
El **stock disponible se lee desde Redis** (caché de lectura, invalidada activamente en cada evento de descuento) para servir el catálogo con baja latencia, mientras que el **descuento real y autoritativo de stock ocurre siempre en una transacción PostgreSQL** (tabla `Lote`/`ReservaStock`) con bloqueo a nivel de fila (`SELECT ... FOR UPDATE`) para resolver condiciones de carrera sobre la última unidad. Cada descuento exitoso en PostgreSQL publica un evento que invalida/actualiza la entrada correspondiente en Redis en el mismo request (no de forma diferida), cumpliendo el objetivo de <5s de NFR-005.

## Alternatives Considered
### Leer el stock directamente de PostgreSQL en cada consulta de catálogo
- A favor: una única fuente de verdad, sin riesgo de inconsistencia entre caché y base de datos.
- En contra: bajo el pico de Navidad, cada búsqueda de catálogo (alto volumen de lectura) competiría por las mismas filas que las transacciones de escritura de picking, degradando la latencia (riesgo directo para NFR-001).
- Motivo de descarte: no cumple el SLO de latencia de catálogo bajo carga de pico sin sobredimensionar la base de datos de forma desproporcionada.

### Consistencia eventual pura (actualizar Redis de forma asíncrona vía cola, sin invalidación síncrona)
- A favor: desacopla completamente lectura y escritura, máxima resiliencia de la caché ante picos de escritura.
- En contra: introduce una ventana de inconsistencia no acotada (podría superar los 5 segundos de NFR-005 bajo carga de la cola), reproduciendo exactamente el problema original que motivó el proyecto ("el stock no se actualiza en tiempo real").
- Motivo de descarte: viola directamente el requisito de negocio que dio origen a este ADR.

## Consequences
### Positivas
- Resuelve el problema de negocio original (stock desactualizado) de forma medible (NFR-005).
- El bloqueo a nivel de fila en PostgreSQL resuelve de forma determinista el escenario descrito por el CTO: "uno gana, uno pierde" en la última unidad (NFR-012).
- Redis también sirve como base para las colas de notificaciones y el export nocturno (reutilización de infraestructura).

### Negativas (trade-offs aceptados)
- Doble fuente de datos de stock (Redis + PostgreSQL) añade complejidad operativa: hay que vigilar que la invalidación de caché no falle silenciosamente.
- El bloqueo a nivel de fila en PostgreSQL puede convertirse en un cuello de botella si muchos operarios escanean el mismo SKU simultáneamente durante el pico de picking matutino; se mitiga con reintentos cortos y colas de picking por pasillo que reducen la contención real.

### Riesgos
- Riesgo: fallo de Redis deja el catálogo sin caché de lectura rápida. Mitigación: fallback a lectura directa de PostgreSQL con rate limiting temporal, aceptando latencia degradada en vez de caída total (ver patrón de resiliencia en [[09-Resiliance-spec]]).
- Riesgo: divergencia silenciosa entre Redis y PostgreSQL por un bug de invalidación. Mitigación: job de reconciliación periódico (cada 5 minutos) que compara y corrige discrepancias, con métrica de alerta si la tasa de discrepancia supera un umbral.

## Decision Makers
Roberto Sánchez (CTO), Javier Ortiz (Responsable de Almacén, validó que el flujo de escaneo cumple sus expectativas de tiempo real).
