# ADR-005: Motor de optimización de rutas y picking — Google Maps API con fallback determinista propio

## Status
ACCEPTED — 2026-07-03

## Context
Pablo Ruiz (Coordinador de Logística) invierte hoy 30-45 minutos cada mañana asignando rutas manualmente, sin optimizar el orden de entrega (FR-025..FR-028). El dominio impone restricciones no estándar de un problema clásico de ruteo: la cadena de frío obliga a que los congelados se entreguen primero (máx. 45 min fuera de congelador) y el almacén tiene tres zonas de picking con recogida secuencial (congelados al final, FR-023). El propio Pablo advirtió: "Lo que no puede pasar es que se queden parados esperando" si Google Maps falla.

## Decision
Usar la **API de Google Maps (Directions + Distance Matrix)** como motor primario de geocodificación y optimización de rutas, combinada con una **capa de reglas de negocio propia** que reordena el resultado bruto de Maps para priorizar entregas con productos congelados al inicio de cada ruta. Ante caída de Google Maps, activar un **fallback determinista propio**: ordenar las entregas por código postal y luego alfabéticamente por calle, sin optimización de distancia (ver NFR-009 y comportamiento de fallo en [[04-Behavioral-specs]]).

## Alternatives Considered
### Construir un motor de optimización de rutas propio (VRP solver interno)
- A favor: control total sobre las restricciones específicas del dominio (temperatura, franjas, peso máximo de 25kg).
- En contra: un solver de Vehicle Routing Problem de calidad de producción es un proyecto de varias semanas por sí solo; el timeline del MVP es de 4 meses para todo el sistema.
- Motivo de descarte: no justifica el esfuerzo frente al pequeño tamaño de cada ruta (30-40 paradas), donde Google Maps + post-procesado de reglas de negocio da resultados suficientemente buenos.

### Depender únicamente de Google Maps sin fallback propio
- A favor: menor complejidad de implementación.
- En contra: contradice directamente la advertencia explícita de Pablo Ruiz sobre repartidores parados esperando; Google Maps no es infalible y el negocio no puede detener el reparto matutino por ello.
- Motivo de descarte: el requisito de resiliencia (NFR-009) es explícito y no negociable para el flujo crítico de reparto.

## Consequences
### Positivas
- Reduce de 30-45 minutos manuales a un proceso automático con revisión rápida del Coordinador (FR-027).
- El fallback determinista garantiza que el reparto matutino nunca se detiene por completo, incluso en el peor caso.
- Reutiliza la misma API de Google Maps para el tracking del cliente (FR-030), evitando integrar un segundo proveedor de geolocalización.

### Negativas (trade-offs aceptados)
- El fallback por código postal no está optimizado por distancia real; en un día de caída de Google Maps, las rutas serán menos eficientes (más kilómetros, más tiempo), lo cual se acepta como degradación controlada frente a la alternativa de detener el reparto.
- Coste variable de la API de Google Maps que crece linealmente con el volumen de pedidos (especialmente en el pico de Navidad), a vigilar en el presupuesto de infraestructura.

### Riesgos
- Riesgo: la capa de reglas de negocio (priorización de congelados) podría generar rutas subóptimas en distancia total si se aplica de forma demasiado agresiva. Mitigación: el Coordinador de Logística conserva la capacidad de ajuste manual antes de aprobar (FR-027) como salvaguarda humana.
- Riesgo: coste de la API de Google Maps en el pico de Navidad (x3 volumen). Mitigación: cachear resultados de geocodificación de direcciones recurrentes (clientes habituales) para reducir llamadas repetidas.

## Decision Makers
Pablo Ruiz (Coordinador de Logística, validó las restricciones operativas), Roberto Sánchez (CTO, decisión de proveedor).
