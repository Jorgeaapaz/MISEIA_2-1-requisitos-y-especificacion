# ADR-001: Base de datos principal — PostgreSQL en RDS Multi-AZ

## Status
ACCEPTED — 2026-07-03

## Context
El sistema actual usa MySQL en una única instancia sin réplica, que se cae 1-2 veces al mes durante 15-20 minutos (Entrevista CTO). NFR-003 exige ≤1h de downtime/mes en temporada normal y NFR-004 exige cero downtime no planificado en temporada alta (nov-ene). Además, el modelo de datos tiene relaciones transaccionales fuertes (pedido → líneas → lotes → facturas, con invariantes de IVA que deben ser atómicos, ver FR-011, REG-007) y requiere soporte para consultas analíticas de trazabilidad de lote en <4h (REG-001). El CTO ha decidido explícitamente el stack: TypeScript/Node.js, React, PostgreSQL, Redis, todo en AWS, sin serverless ni Kubernetes en el MVP.

## Decision
Usar **PostgreSQL 16 en Amazon RDS con despliegue Multi-AZ** como base de datos transaccional principal. Failover automático a la réplica síncrona ante fallo de la instancia primaria. Backups automáticos con WAL continuo (RPO de 5 minutos, ver [[07-Operative-specs]]).

## Alternatives Considered
### MySQL (mantener y añadir réplica)
- A favor: el equipo ya conoce el esquema actual; migración incremental posible.
- En contra: el código PHP+MySQL de 6 años "nadie quiere tocarlo"; el rediseño del modelo de datos es total de todas formas al cambiar de PHP a Node/TS.
- Motivo de descarte: no hay ahorro real de esfuerzo frente a partir de cero con PostgreSQL, y PostgreSQL ofrece mejores garantías transaccionales (constraints, tipos JSONB para flexibilidad futura) para el nuevo modelo.

### DynamoDB (NoSQL gestionado)
- A favor: escalado automático sin gestión de instancias, encaja con "no queremos gestionar servidores".
- En contra: el dominio tiene relaciones transaccionales complejas (stock, lotes, FEFO, IVA desglosado) que requieren transacciones ACID multi-entidad; modelarlo en DynamoDB añade complejidad de acceso significativa.
- Motivo de descarte: el modelo relacional del negocio (pedidos-líneas-lotes-facturas con invariantes fiscales) encaja mal con un modelo de acceso por clave-valor; el riesgo de inconsistencia en estos requisitos regulatorios es inaceptable.

## Consequences
### Positivas
- Multi-AZ elimina el punto único de fallo que causó las caídas históricas.
- Transacciones ACID nativas simplifican la implementación de los contratos de `confirmarPedido` y `escanearProducto` (ver [[08-Contracts]]).
- RDS gestiona parcheo, backups y failover sin equipo dedicado de DBA.

### Negativas (trade-offs aceptados)
- Escalado vertical con límites: para el pico de Navidad (x3 tráfico) se requiere dimensionar la instancia con margen adicional o introducir réplicas de lectura, lo cual es un coste recurrente aceptado.
- Mayor coste que una instancia única sin Multi-AZ (aprox. 2x el coste de cómputo de base de datos).

### Riesgos
- Riesgo: el failover de RDS Multi-AZ tiene una ventana de interrupción de 60-120 segundos. Mitigación: el ALB y los reintentos con backoff del backend absorben esta ventana sin impacto visible al usuario en la mayoría de los casos; se documenta como excepción aceptada dentro del SLO de NFR-004.
- Riesgo: picos de escritura durante el cutoff de las 22:00 (muchos pedidos confirmándose a la vez). Mitigación: uso de colas (Redis/BullMQ) para desacoplar operaciones no críticas del camino síncrono de escritura (ver [[003-ADR-cache-y-consistencia-de-stock]]).

## Decision Makers
Roberto Sánchez (CTO, decisión técnica final), equipo de Ingeniería (validación de modelo de datos).
