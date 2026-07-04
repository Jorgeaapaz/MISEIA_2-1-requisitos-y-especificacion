# ADR-004: Pasarela de pago — Stripe con idempotency keys

## Status
ACCEPTED — 2026-07-03

## Context
El CTO exige explícitamente el uso de Stripe (ya usado en la web actual, soporta tarjeta y Bizum). El escenario de fallo más citado por Operaciones y CTO es: "¿Qué pasa si Stripe se cae justo cuando un cliente está pagando?" con la exigencia explícita de Roberto Sánchez de "nada de cobrar dos veces". Esto está codificado como NFR-011 y como precondición/postcondición del contrato `confirmarPedido` (ver [[08-Contracts]]). REG-003 (PCI DSS) exige además no almacenar datos de tarjeta en los sistemas propios.

## Decision
Integrar **Stripe** como único proveedor de pago del MVP, usando **Payment Intents con idempotency keys** generadas por el frontend en cada intento de checkout. El backend nunca reintenta un cobro sin antes verificar el estado real del Payment Intent asociado a esa idempotency key. Los datos de tarjeta se capturan exclusivamente vía Stripe.js/Elements en el cliente; el backend solo maneja tokens y IDs de Payment Intent, nunca PAN ni CVV.

## Alternatives Considered
### Redsys (pasarela española tradicional)
- A favor: ampliamente usada en el sector retail español, soporta Bizum de forma nativa.
- En contra: decisión explícita del CTO de mantener Stripe (ya integrado en la web actual, migración de menor riesgo); Stripe soporta Bizum a través de su propia integración.
- Motivo de descarte: no aporta ventaja suficiente para justificar cambiar de proveedor y volver a certificar el flujo de pago.

### Cobro diferido (autorizar en checkout, capturar tras picking completo)
- A favor: evitaría cobrar por productos que luego se cancelan por falta de stock durante el picking, reduciendo reembolsos parciales.
- En contra: Carmen López fue explícita en que "le cobramos inmediatamente" es el modelo de negocio actual y esperado; cambiarlo afecta a flujo de caja y a la lógica de facturación (FR-013 exige CONFIRMADO tras pago exitoso, no tras picking).
- Motivo de descarte: cambia el modelo de negocio sin que ningún stakeholder lo haya solicitado; el caso de sustitución/cancelación de línea ya está cubierto por reembolso parcial post-picking si aplica.

## Consequences
### Positivas
- Cumple PCI DSS por delegación completa (REG-003) sin que el equipo deba certificarse como manejador de datos de tarjeta.
- Las idempotency keys resuelven de forma estándar y probada el escenario de doble cobro que el CTO marcó como no negociable.
- Bizum vía Stripe evita integrar un segundo proveedor de pago.

### Negativas (trade-offs aceptados)
- Los reembolsos tardan 5-10 días laborables en reflejarse en la tarjeta del cliente (limitación del propio Stripe/bancos), aunque el sistema los inicie en <48h (FR-040) — esto debe comunicarse claramente al cliente para evitar tickets de soporte innecesarios.
- Dependencia total de la disponibilidad de Stripe para poder confirmar cualquier pedido (ver comportamiento ante fallo en [[04-Behavioral-specs]] y patrón de resiliencia en [[09-Resiliance-spec]]).

### Riesgos
- Riesgo: caída prolongada de Stripe bloquea todos los checkouts. Mitigación: circuit breaker con mensaje claro al usuario y alerta a Ingeniería; no existe fallback de pago aceptable dado que "el pedido no se puede confirmar sin pago" (Carmen López).
- Riesgo: uso incorrecto de idempotency keys en el frontend (ej. generar una nueva en cada reintento en vez de reutilizarla) reintroduciría el riesgo de doble cobro. Mitigación: test de contrato automatizado que verifica este comportamiento antes de cada despliegue (ver OPS-001).

## Decision Makers
Roberto Sánchez (CTO), Miguel Torres (Director Financiero, validó el flujo de reembolsos y plazos).
