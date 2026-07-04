# ADR-002: Arquitectura general — Monolito modular en EC2 con Auto Scaling

## Status
ACCEPTED — 2026-07-03

## Context
El presupuesto y timeline son restrictivos: MVP en 4 meses, con el mandato explícito del CEO de "cloud estándar (AWS). No serverless, no Kubernetes al principio — empezar con EC2 + RDS y escalar después si hace falta". El sistema debe soportar 800 pedidos/día en temporada normal y 2.400-3.000 en Navidad (NFR-001, NFR-002), con 6 roles de usuario distintos (FR de Cliente, Almacén, Logística, Repartidor, Soporte, Administrador) que en el futuro podrían separarse en equipos distintos, pero hoy son desarrollados por un único equipo pequeño.

## Decision
Construir un **monolito modular** en Node.js/TypeScript, organizado internamente en módulos por dominio (Pedidos, Stock, Picking, Rutas, Incidencias, Facturación — ver [[05-Structural-specs]]), desplegado sobre un **Auto Scaling Group de instancias EC2** detrás de un Application Load Balancer. Separar en microservicios independientes solo si un módulo concreto demuestra necesitar escalado o ciclo de despliegue diferenciado (candidato más probable: Rutas/Tracking, por su naturaleza de tiempo real).

## Alternatives Considered
### Microservicios desde el inicio
- A favor: aislamiento de fallos por dominio, escalado independiente por servicio, alineado con el futuro roadmap de app móvil para repartidores.
- En contra: la complejidad operativa (orquestación, observabilidad distribuida, comunicación entre servicios) no es asumible en 4 meses con un equipo que además tiene mandato explícito de "no Kubernetes al principio".
- Motivo de descarte: sobreingeniería para el tamaño actual del equipo y el timeline del MVP; el coste de coordinación entre servicios retrasaría la entrega antes de Navidad.

### Serverless (Lambda + API Gateway)
- A favor: escalado automático nativo, coste variable ajustado al tráfico real (útil dado el x3 de Navidad).
- En contra: restricción explícita del CEO en la entrevista ("no serverless"); además, los flujos de picking offline y de rutas de larga duración encajan peor con el modelo de ejecución de Lambda (timeouts, cold starts).
- Motivo de descarte: contradice un requisito de negocio explícito, no una preferencia técnica discutible.

## Consequences
### Positivas
- Despliegue y observabilidad simples: un único pipeline de CI/CD (OPS-001), un único conjunto de dashboards.
- Cumple el mandato explícito del CEO sobre infraestructura.
- Auto Scaling Group cubre el pico de Navidad (NFR-002) escalando horizontalmente instancias idénticas, sin rediseño arquitectónico.

### Negativas (trade-offs aceptados)
- Un despliegue afecta a todos los módulos simultáneamente (menor aislamiento de cambios que microservicios).
- Escalado es uniforme: si solo el módulo de Rutas necesita más CPU en un momento dado, se escala toda la aplicación igualmente (ineficiencia de coste aceptada a cambio de simplicidad).

### Riesgos
- Riesgo: a medida que crezca el equipo o el dominio, el monolito puede volverse difícil de mantener. Mitigación: la separación en módulos con dependencias explícitas (ver interfaces en [[05-Structural-specs]]) permite extraer un módulo a microservicio de forma incremental sin reescritura completa.
- Riesgo: un bug en un módulo (ej. Picking) puede degradar el rendimiento de otro (ej. Checkout) al compartir proceso. Mitigación: límites de recursos por request y colas asíncronas para operaciones pesadas.

## Decision Makers
Marta Reyes (CEO, restricción de infraestructura), Roberto Sánchez (CTO, diseño técnico).
