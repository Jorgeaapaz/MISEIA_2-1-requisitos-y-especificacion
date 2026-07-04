# Criterios de Aceptación — OMS FreshDirect

## Parte 1: Escenarios BDD (Requisitos Funcionales)

```gherkin
Feature: Búsqueda y carrito de compra (FR-001, FR-002, FR-003, FR-004, FR-005)
  Como Cliente
  Quiero buscar productos y añadirlos al carrito con su stock e IVA correctos
  Para confiar en que lo que compro está realmente disponible

  Scenario: Búsqueda muestra stock en tiempo real
    Given el producto "Leche entera 1L" tiene 5 unidades en stock
    When el Cliente busca "leche"
    Then el sistema muestra "Leche entera 1L" con 5 unidades disponibles y su precio con IVA del 4% incluido

  Scenario: Confirmar carrito con línea sin stock
    Given el producto "Salmón congelado 400g" tiene 0 unidades en stock
    When el Cliente intenta confirmar un carrito que incluye "Salmón congelado 400g"
    Then el sistema rechaza la línea y solicita al Cliente aceptar un sustituto o eliminarla

  Scenario: Última unidad disputada por dos clientes
    Given el producto "Tarta de queso" tiene 1 unidad en stock
    And dos clientes tienen esa unidad en su carrito simultáneamente
    When ambos clientes confirman el pago al mismo tiempo
    Then solo el pago confirmado primero se queda con la unidad
    And al segundo cliente el sistema le ofrece el sustituto aceptado o cancela la línea con un mensaje claro
```

```gherkin
Feature: Checkout con dirección, descuento e IVA (FR-006, FR-007, FR-008, FR-009, FR-010, FR-011)
  Como Cliente
  Quiero introducir mi dirección y un código de descuento
  Para pagar el importe correcto por un pedido que sí se puede entregar

  Scenario: Dirección dentro de cobertura
    Given el Cliente introduce una dirección en Getafe (Madrid)
    When el sistema geocodifica la dirección
    Then la dirección se acepta y se calcula el coste de envío estándar

  Scenario: Dirección fuera de cobertura
    Given el Cliente introduce una dirección en Palma de Mallorca
    When el sistema geocodifica la dirección
    Then el sistema rechaza el pedido indicando que la zona no tiene cobertura

  Scenario: Código de descuento caducado
    Given existe el código "VERANO10" con fecha de caducidad anterior a hoy
    When el Cliente aplica el código "VERANO10"
    Then el sistema rechaza el código y permite continuar el checkout sin descuento

  Scenario: Desglose de IVA con productos de varios tipos
    Given el carrito contiene pan (IVA 4%), carne (IVA 10%) y detergente (IVA 21%)
    When el sistema calcula el total del pedido
    Then la factura muestra la base imponible y la cuota de IVA desglosadas para cada uno de los tres tipos
```

```gherkin
Feature: Pago del pedido (FR-012, FR-013, FR-014)
  Como Cliente
  Quiero pagar mi pedido de forma segura y sin cobros duplicados
  Para confiar en la plataforma

  Scenario: Pago exitoso confirma el pedido
    Given el Cliente ha completado el checkout con un total de 42,30€
    When el pago con tarjeta se procesa correctamente en Stripe
    Then el pedido pasa a estado CONFIRMADO
    And el Cliente recibe un email y un SMS de confirmación

  Scenario: Stripe no responde durante el pago
    Given el Cliente ha iniciado el pago
    When Stripe no responde dentro del tiempo esperado
    Then el sistema conserva el carrito del Cliente
    And muestra un mensaje indicando que puede reintentar en unos minutos
    And no confirma el pedido hasta verificar el estado real del cobro

  Scenario: Reintento de pago no duplica el cobro
    Given un primer intento de pago quedó en estado incierto por timeout de red
    When el Cliente reintenta el pago
    Then el sistema verifica primero si el cobro anterior se completó
    And no genera un segundo cargo si el primero ya fue exitoso
```

```gherkin
Feature: Modificación y cutoff de pedidos (FR-015, FR-016)
  Como Cliente
  Quiero poder modificar mi pedido antes de que empiece su preparación
  Para corregir errores sin tener que llamar a soporte

  Scenario: Modificación antes del cutoff
    Given son las 21:00 y el pedido está programado para mañana
    When el Cliente elimina una línea de producto
    Then el sistema aplica el cambio y recalcula el total

  Scenario: Modificación rechazada después del cutoff
    Given son las 22:15 y el pedido está programado para mañana
    When el Cliente intenta modificar el pedido
    Then el sistema rechaza el cambio indicando que el pedido ya está en preparación
```

```gherkin
Feature: Picking guiado con escaneo y FEFO (FR-017, FR-018, FR-019, FR-020, FR-021, FR-022, FR-023)
  Como Operario de Almacén
  Quiero una lista de picking guiada con escaneo de código de barras
  Para recoger el producto y lote correctos sin errores

  Scenario: Picking exitoso descuenta stock en tiempo real
    Given un pedido confirmado con 3 líneas de producto
    When el Operario escanea correctamente las 3 líneas
    Then el sistema descuenta el stock de cada producto en el momento del escaneo
    And el pedido pasa a estado PREPARADO

  Scenario: Selección de lote por FEFO
    Given el producto "Yogur natural" tiene dos lotes: uno caduca en 4 días y otro en 10 días
    When el sistema indica al Operario qué lote recoger
    Then el sistema indica el lote que caduca en 4 días (el más próximo con al menos 3 días de margen)

  Scenario: Bloqueo de producto próximo a caducar
    Given un lote de "Pechuga de pollo" caduca hoy + 1 día
    When el Operario intenta escanearlo para el picking
    Then el sistema rechaza el escaneo y solicita el siguiente lote válido

  Scenario: Producto agotado durante el picking con sustituto aceptado
    Given el Cliente aceptó sustitutos para "Yogur natural"
    And el stock de "Yogur natural" se agota durante el picking
    When el Operario intenta escanear el producto agotado
    Then el sistema propone automáticamente el sustituto configurado de igual categoría y precio igual o inferior

  Scenario: Producto agotado sin sustituto aceptado
    Given el Cliente no aceptó sustitutos para "Salmón fresco"
    And el stock de "Salmón fresco" se agota durante el picking
    When el Operario intenta escanear el producto agotado
    Then el sistema cancela la línea del pedido y ajusta el importe total
```

```gherkin
Feature: Generación y aprobación de rutas (FR-025, FR-026, FR-027, FR-028)
  Como Coordinador de Logística
  Quiero rutas generadas automáticamente y optimizadas
  Para no perder 30-45 minutos cada mañana asignándolas a mano

  Scenario: Generación automática de rutas por zona postal
    Given existen 320 pedidos PREPARADOS para la franja de mañana
    When el sistema genera las rutas
    Then los pedidos quedan agrupados por zona postal en rutas de entre 30 y 40 entregas cada una
    And los productos congelados quedan priorizados al inicio de cada ruta

  Scenario: Coordinador ajusta una ruta antes de aprobarla
    Given una ruta generada automáticamente tiene 42 entregas asignadas a un repartidor
    When el Coordinador reasigna 3 entregas a otro repartidor
    Then la ruta ajustada queda dentro del rango de 30-40 entregas antes de su aprobación
```

```gherkin
Feature: Tracking y entrega con prueba fotográfica (FR-030, FR-031, FR-032, FR-033, FR-034)
  Como Cliente
  Quiero ver dónde está mi pedido y tener prueba de su entrega
  Para no depender de llamar a soporte

  Scenario: Cliente ve el tracking en tiempo real
    Given el pedido está en estado EN_REPARTO
    When el Cliente consulta el estado de su pedido
    Then el sistema muestra la ubicación estimada y el estado actualizado del pedido

  Scenario: Entrega exitosa con foto
    Given el Repartidor llega a la dirección del Cliente
    When el Repartidor confirma la entrega adjuntando una fotografía
    Then el pedido pasa a estado ENTREGADO
    And el sistema genera la factura electrónica del pedido

  Scenario: Cliente ausente en la entrega
    Given el Repartidor llega a la dirección y el Cliente no responde
    When han pasado 5 minutos y la llamada telefónica no tiene respuesta
    Then el sistema marca la entrega como FALLIDA
    And los productos de cadena de frío asociados quedan marcados como no reentregables
```

```gherkin
Feature: Incidencias y reembolsos post-entrega (FR-037, FR-038, FR-039, FR-040)
  Como Cliente
  Quiero que las incidencias de bajo importe se resuelvan automáticamente
  Para no esperar 2-3 días como ahora

  Scenario: Reembolso automático para incidencia de bajo importe
    Given el Cliente reporta un producto dañado de 6€ con foto adjunta
    And el Cliente no ha alcanzado el límite mensual de reembolsos automáticos
    When el sistema evalúa la incidencia
    Then el sistema genera un reembolso automático inmediato
    And el reembolso se inicia en Stripe en menos de 48 horas

  Scenario: Incidencia de importe alto requiere revisión
    Given el Cliente reporta un producto equivocado de 18€ con foto adjunta
    When el sistema evalúa la incidencia
    Then el sistema deriva el caso a un Agente de Soporte para revisión antes de reembolsar

  Scenario: Límite antifraude de reembolsos automáticos alcanzado
    Given el Cliente ya recibió 2 reembolsos automáticos este mes
    When el Cliente reporta una nueva incidencia de 4€ con foto
    Then el sistema deriva el caso a revisión manual en lugar de reembolsar automáticamente
```

```gherkin
Feature: Alerta sanitaria y trazabilidad de lote (FR-044, FR-045)
  Como Responsable de Calidad y Seguridad Alimentaria
  Quiero identificar y notificar a los clientes afectados por un lote en menos de 4 horas
  Para cumplir con la normativa y evitar sanciones

  Scenario: Identificación de clientes afectados por lote
    Given se recibe una alerta sanitaria sobre el lote "L2025-PO-447"
    When el Responsable de Calidad busca ese lote en el sistema
    Then el sistema devuelve la lista completa de pedidos y clientes que recibieron ese lote en menos de 4 horas

  Scenario: Notificación urgente enviada y registrada
    Given se ha identificado la lista de clientes afectados por el lote "L2025-PO-447"
    When el sistema envía la notificación urgente
    Then cada cliente recibe un email y un SMS
    And el sistema registra el envío y la recepción de cada notificación
```

## Parte 2: Criterios EARS (Requisitos No-Funcionales, Regulatorios y Operativos)

```
# NFR-001 — Patrón Ubiquitous
EARS-NFR-001: The system SHALL respond to catalog and checkout API requests with a p95 latency below 500ms while sustaining 1,000 orders/day.

# NFR-002 — Patrón State-driven
EARS-NFR-002: While the system is operating during the November-January peak season, the system SHALL sustain 3,000 orders/day without service degradation.

# NFR-003 — Patrón Ubiquitous
EARS-NFR-003: The system SHALL maintain no more than 1 hour of cumulative downtime per month during normal season.

# NFR-004 — Patrón State-driven
EARS-NFR-004: While operating during the November-January peak season, the system SHALL have zero unplanned downtime.

# NFR-005 — Patrón Event-driven
EARS-NFR-005: When a warehouse operator scans a product during picking, the system SHALL propagate the stock deduction to the public catalog within 5 seconds.

# NFR-006 — Patrón Unwanted
EARS-NFR-006: If system resource usage (CPU, memory, DB connections) approaches a critical threshold, the system SHALL trigger a proactive alert at least 5 minutes before a full outage would occur.

# NFR-007 — Patrón Ubiquitous
EARS-NFR-007: The system SHALL NOT store or log full card numbers or CVV data at any point in the order flow.

# NFR-008 — Patrón Unwanted
EARS-NFR-008: If the warehouse tablet loses connection to the server during picking, the system SHALL allow the operator to continue using the already-downloaded picking list and SHALL synchronize all captured events once connectivity is restored.

# NFR-009 — Patrón Unwanted
EARS-NFR-009: If the Google Maps API is unavailable, the system SHALL provide delivery drivers with a fallback postal-code-ordered route within 2 minutes of detecting the outage.

# NFR-010 — Patrón Ubiquitous
EARS-NFR-010: The system SHALL enable an average picking time per order below 5 minutes.

# NFR-011 — Patrón Unwanted
EARS-NFR-011: If a payment retry occurs after an inconclusive prior attempt, the system SHALL verify the original charge status before allowing a new charge, preventing duplicate payments.

# NFR-012 — Patrón Unwanted
EARS-NFR-012: If two concurrent checkouts target the last unit of the same product, the system SHALL grant the unit to only one order and SHALL prevent negative stock.

# NFR-013 — Patrón Ubiquitous
EARS-NFR-013: The system SHALL enable resolution of customer incidents in under 24 hours, and under 1 hour for incidents below 10€.

# REG-001 — Patrón Ubiquitous
EARS-REG-001: The system SHALL enable identification of all orders and customers associated with a specific product batch within 4 hours of a food safety alert.

# REG-002 — Patrón Ubiquitous
EARS-REG-002: The system SHALL process customer personal data (name, address, phone, email) in accordance with GDPR data minimization and retention principles.

# REG-003 — Patrón Ubiquitous
EARS-REG-003: The system SHALL delegate storage and processing of cardholder payment data to a PCI DSS-certified provider (Stripe).

# REG-004 — Patrón Ubiquitous
EARS-REG-004: The system SHALL present legally required cookie consent and legal notice information in compliance with LSSI-CE.

# REG-005 — Patrón Event-driven
EARS-REG-005: When an order reaches the ENTREGADO state, the system SHALL generate a legally valid electronic invoice in accordance with Spanish e-invoicing regulation.

# REG-006 — Patrón Unwanted
EARS-REG-006: If a product batch's expiry date is today + 2 days or sooner, the system SHALL prevent that batch from being selected during picking.

# REG-007 — Patrón Ubiquitous
EARS-REG-007: The system SHALL apply and itemize the correct VAT rate (4%, 10%, or 21%) for each product line on every invoice.

# OPS-001 — Patrón Event-driven
EARS-OPS-001: When code is merged to the main branch, the CI/CD pipeline SHALL run lint, tests and build, and SHALL require manual approval before deploying to production.

# OPS-002 — Patrón Ubiquitous
EARS-OPS-002: The system SHALL expose infrastructure and business metrics with alerting configured on critical thresholds.

# OPS-003 — Patrón Ubiquitous
EARS-OPS-003: The system SHALL maintain a documented backup strategy with defined RTO and RPO for disaster recovery.

# OPS-004 — Patrón Event-driven
EARS-OPS-004: When the clock reaches 02:00 daily, the system SHALL export the day's orders and invoices to CSV for SAP import, and SHALL retry automatically on failure.

# OPS-005 — Patrón Optional
EARS-OPS-005: Where a critical incident occurs (site outage in peak season, cold-chain breach, unconfirmed deliveries), the operations team SHALL follow a documented runbook.

# OPS-006 — Patrón Event-driven
EARS-OPS-006: When network connectivity is restored on a warehouse tablet, the system SHALL synchronize all offline-captured picking events without data loss.
```
