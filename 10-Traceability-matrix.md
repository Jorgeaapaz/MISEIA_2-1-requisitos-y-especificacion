# Matriz de Trazabilidad — OMS FreshDirect

## 1. Trazabilidad de Requisitos Funcionales

| Requisito | Origen | Spec Funcional | Criterio de Aceptación | Contrato | Test |
|---|---|---|---|---|---|
| FR-001, FR-002 | Entrevista Carmen López (stock tiempo real), Miguel Torres (IVA) | `GET /catalogo/productos` | Feature "Búsqueda y carrito de compra" | — | Test unitario de filtro de búsqueda + test de cálculo de precio con IVA |
| FR-003, FR-004, FR-005 | Entrevista Carmen López, Javier Ortiz (sustitutos) | `POST /carrito/lineas` | Feature "Búsqueda y carrito de compra", escenario "Última unidad disputada" | Precondiciones de `confirmarPedido` | Test de concurrencia sobre última unidad |
| FR-006, FR-007, FR-008 | Entrevista Carmen López (franjas), Pablo Ruiz (Express) | `POST /checkout/validar-direccion` | Feature "Checkout con dirección, descuento e IVA" | Precondición 3 de `confirmarPedido` | Test de geocodificación con dirección fuera de cobertura |
| FR-009, FR-010 | Entrevista Miguel Torres (descuentos) | `POST /checkout/aplicar-descuento` | Feature "Checkout...", escenario "Código de descuento caducado" | — | Test de validación de código (caducado, usado, importe mínimo) |
| FR-011 | Entrevista Miguel Torres (IVA por tipo) | `POST /checkout/confirmar-pedido` | Feature "Checkout...", escenario "Desglose de IVA" | Contrato `calcularTotalesConIVA` | Test de función pura con productos de los 3 tipos de IVA |
| FR-012, FR-013, FR-014 | Entrevista Roberto Sánchez (Stripe, no doble cobro) | `POST /checkout/confirmar-pedido` | Feature "Pago del pedido" | Contrato `confirmarPedido` | Test de idempotencia de pago |
| FR-015, FR-016 | Entrevista Lucía Fernández (cutoff 22:00) | `PATCH /pedidos/{pedidoId}` | Feature "Modificación y cutoff de pedidos" | — | Test de rechazo de modificación post-cutoff |
| FR-017, FR-020, FR-023 | Entrevista Javier Ortiz (picking por zona, FEFO) | `GET /almacen/picking/{operarioId}/siguiente` | Feature "Picking guiado...", escenario "Selección de lote por FEFO" | Contrato `escanearProducto` | Test de orden de pasillo y de asignación FEFO |
| FR-018, FR-019, FR-021, FR-022 | Entrevista Javier Ortiz (escaneo, caducidad, sustituto) | `POST /almacen/picking/{pedidoId}/escanear` | Feature "Picking guiado...", escenarios de bloqueo y sustituto | Contrato `escanearProducto` | Test de bloqueo por caducidad ≤2 días, test de sustituto |
| FR-024 | Entrevista Ana Martínez (report de caducidad) | (job interno, no expuesto como API pública) | — (requisito operativo derivado) | — | Test de generación de report diario |
| FR-025, FR-026, FR-027 | Entrevista Pablo Ruiz (rutas automáticas) | `POST /logistica/rutas/generar`, `PATCH /logistica/rutas/{rutaId}/aprobar` | Feature "Generación y aprobación de rutas" | — | Test de agrupación por zona postal y límite 30-40 |
| FR-028, FR-029 | Entrevista Pablo Ruiz (info del repartidor, tracking) | (incluido en response de `AprobarRutaResponse` y servicio de tracking) | — | — | Test de payload completo entregado al repartidor |
| FR-030, FR-031 | Entrevista Carmen López, Lucía Fernández (tracking reduce llamadas) | `GET /pedidos/{pedidoId}/tracking` | Feature "Tracking y entrega...", escenario "Cliente ve el tracking" | — | Test de estados de tracking |
| FR-032, FR-033, FR-034 | Entrevista Carmen López (foto), Pablo Ruiz (cliente ausente) | `POST /reparto/{pedidoId}/confirmar-entrega` | Feature "Tracking y entrega...", escenario "Cliente ausente" | Transición de estado EN_REPARTO→ENTREGADO/FALLIDO | Test de flujo de entrega fallida |
| FR-035 | Entrevista Pablo Ruiz (alerta 2h sin confirmar) | (alerta interna OPS-002) | — | — | Test de disparo de alerta a las 2h |
| FR-036 | Entrevista Pablo Ruiz (incidencia en ruta) | `POST /reparto/{pedidoId}/confirmar-entrega` (campo `incidenciaProductoDanado`) | — | — | Test de creación de incidencia desde reparto |
| FR-037, FR-038, FR-039, FR-040 | Entrevista Lucía Fernández, Miguel Torres (reembolsos) | `POST /incidencias` | Feature "Incidencias y reembolsos post-entrega" | — | Test de reembolso automático <10€ y límite antifraude |
| FR-041 | Entrevista Lucía Fernández (historial completo) | (endpoint de historial, análogo a catálogo) | — | — | Test de listado de historial con detalle |
| FR-042, FR-043 | Entrevista Miguel Torres (factura, export SAP) | (servicio de facturación, job de export) | — | — | Test de desglose de IVA en factura, test de export CSV |
| FR-044, FR-045 | Entrevista Ana Martínez (alerta sanitaria <4h) | `POST /calidad/alertas-sanitarias/{loteId}/notificar` | Feature "Alerta sanitaria y trazabilidad de lote" | — | Test de tiempo de identificación y notificación |
| FR-046 | Entrevista Ana Martínez (temperatura furgoneta) | (ingesta de telemetría, alerta interna) | — | — | Test de alerta por temperatura > -15°C |
| FR-047, FR-048 | Entrevista Miguel Torres (promociones), Roberto Sánchez (rol Administrador) | (endpoints de administración de catálogo/descuentos) | — | — | Test de validación de configuración de descuento |
| FR-049 | Entrevista Lucía Fernández (chat) | (canal de chat, fuera del alcance de API crítica) | — | — | Test manual de flujo de chat |
| FR-050 | Entrevista Carmen López (peso máx. 25kg) | (lógica de división de envío en `confirmar-pedido`) | — | — | Test de división de pedido >25kg |

## 2. Trazabilidad de Requisitos No-Funcionales

| Requisito | Origen | Spec | Criterio (EARS) | Derivados | ADR | Resiliencia |
|---|---|---|---|---|---|---|
| NFR-001 | Entrevista Marta Reyes (800→1000 pedidos/día) | [[04-Behavioral-specs]] §4 SLI/SLO | EARS-NFR-001 | Métrica `http_request_duration_seconds` | [[002-ADR-arquitectura-general]] | Sección 4 SLAs de [[09-Resiliance-spec]] |
| NFR-002 | Entrevista Marta Reyes (pico Navidad x3) | [[04-Behavioral-specs]] §4 SLI/SLO | EARS-NFR-002 | Auto Scaling Group | [[002-ADR-arquitectura-general]] | Error budget nulo en temporada alta |
| NFR-003 | Entrevista Roberto Sánchez (1h/mes downtime) | [[04-Behavioral-specs]] §4 | EARS-NFR-003 | Métrica de uptime | [[001-ADR-base-de-datos]] | Error budget 60 min/mes |
| NFR-004 | Entrevista Roberto Sánchez, Marta Reyes (zero tolerance) | [[04-Behavioral-specs]] §4 | EARS-NFR-004 | RDS Multi-AZ | [[001-ADR-base-de-datos]] | Error budget 0 min |
| NFR-005 | Entrevista Javier Ortiz (stock desactualizado) | [[04-Behavioral-specs]] §2.2 | EARS-NFR-005 | `stock_deduction_latency_seconds` | [[003-ADR-cache-y-consistencia-de-stock]] | Patrón `redis_cache_colas` |
| NFR-006 | Entrevista Roberto Sánchez (monitorización proactiva) | [[07-Operative-specs]] §2 | EARS-NFR-006 | Alertas de saturación de recursos | — | — |
| NFR-007 | Entrevista Roberto Sánchez (PCI DSS logs) | [[07-Operative-specs]] §5 Logging | EARS-NFR-007 | Campo `paymentIntentId` en logs | [[004-ADR-pasarela-de-pago-stripe]] | — |
| NFR-008 | Entrevista Javier Ortiz, Roberto Sánchez (modo offline) | [[04-Behavioral-specs]] §2.2 | EARS-NFR-008 | `EventoPicking.origenOffline` | — | Patrón `tablet_almacen_conexion` |
| NFR-009 | Entrevista Pablo Ruiz (Google Maps caído) | [[04-Behavioral-specs]] §2.3 | EARS-NFR-009 | Fallback por código postal | [[005-ADR-optimizacion-de-rutas-y-picking]] | Patrón `google_maps_rutas` |
| NFR-010 | Entrevista Javier Ortiz (tiempo de picking) | [[04-Behavioral-specs]] §4 | EARS-NFR-010 | `picking_time_seconds` | [[005-ADR-optimizacion-de-rutas-y-picking]] | — |
| NFR-011 | Entrevista Roberto Sánchez (no doble cobro) | [[08-Contracts]] §2 | EARS-NFR-011 | Idempotency key | [[004-ADR-pasarela-de-pago-stripe]] | Patrón `stripe_pagos` |
| NFR-012 | Entrevista Roberto Sánchez (última unidad) | [[08-Contracts]] §4 | EARS-NFR-012 | `ReservaStock`, bloqueo de fila | [[003-ADR-cache-y-consistencia-de-stock]] | — |
| NFR-013 | Entrevista Lucía Fernández (resolución de incidencias) | [[04-Behavioral-specs]] §4 | EARS-NFR-013 | `incident_resolution_time_seconds` | — | — |

## 3. Trazabilidad de Requisitos Regulatorios

| Requisito | Normativa | Spec | Criterio (EARS) | Implementación | Verificación |
|---|---|---|---|---|---|
| REG-001 | Reglamento (CE) 178/2002 | [[04-Behavioral-specs]] §4 (SLI/SLO alerta sanitaria) | EARS-REG-001 | `TrazabilidadService`, `LoteClienteIndex` (ver [[05-Structural-specs]]) | Test de tiempo de identificación de clientes por lote <4h |
| REG-002 | GDPR | [[07-Operative-specs]] §5 Logging | EARS-REG-002 | Campos PII marcados en `Cliente`, `Direccion` (ver [[05-Structural-specs]]) | Auditoría de logs sin PII en nivel INFO/DEBUG |
| REG-003 | PCI DSS | [[08-Contracts]] §5 (interfaz `PagoGateway`) | EARS-REG-003 | Delegación total a Stripe | Auditoría: 0 datos de tarjeta almacenados |
| REG-004 | LSSI-CE | (fuera del alcance técnico central, requisito de frontend/legal) | EARS-REG-004 | Banner de cookies, aviso legal en frontend | Revisión legal previa a lanzamiento |
| REG-005 | Normativa de facturación electrónica española (2026) | [[06-Functional-specs]] (generación de factura tras ENTREGADO) | EARS-REG-005 | `FacturacionService`, entidad `Factura` | Test de validez del formato de factura electrónica |
| REG-006 | Normativa seguridad alimentaria / política interna FEFO | [[08-Contracts]] §4 | EARS-REG-006 | Bloqueo de picking en `escanearProducto` | Test de bloqueo de lote con caducidad ≤2 días |
| REG-007 | Normativa fiscal IVA española | [[08-Contracts]] §3 (`calcularTotalesConIVA`) | EARS-REG-007 | Campo `tipoIVA` en `Producto`, `LineaPedido`, `Factura` | Test de desglose de IVA por los 3 tipos |

## 4. Trazabilidad de Requisitos Operativos

| Requisito | Spec | Implementación | Verificación |
|---|---|---|---|
| OPS-001 | [[07-Operative-specs]] §1 Pipeline CI/CD | GitHub Actions (lint/test/build/staging/aprobación/producción) | Verificación de que un fallo de test bloquea el despliegue |
| OPS-002 | [[07-Operative-specs]] §2 Métricas y Alertas | Métricas Prometheus + reglas de alerta | Revisión de dashboards y simulacro de disparo de alertas |
| OPS-003 | [[07-Operative-specs]] §4 Backup y DR | Snapshots RDS + réplica cross-region S3 | Simulacro de DR trimestral (obligatorio en octubre) |
| OPS-004 | [[07-Operative-specs]] §1 (mención), [[06-Functional-specs]] (export) | Job programado a las 02:00 con reintento | Test de generación de CSV y reintento ante fallo |
| OPS-005 | [[07-Operative-specs]] §3 Runbooks | RUNBOOK-001 a RUNBOOK-007 | Simulacro de incidente (game day) por cada runbook crítico |
| OPS-006 | [[07-Operative-specs]] §1 (relacionado con NFR-008) | Sincronización de `EventoPicking` offline | Test de sincronización íntegra tras reconexión |

## 5. Cadena de Derivación (ejemplos del dominio real)

```
USUARIO DICE                                          REQUISITO      DERIVADO                                    DECISIÓN
──────────────────────────────────────────────        ─────────      ─────────                                   ─────────
"El año pasado la web se cayó el 23 de diciembre       → NFR-004    → "Necesitamos redundancia activa de BD     → ADR-001
 durante 4 horas y perdimos unas 400 pedidos"                          y failover automático, no backup frío"

"Nada de cobrar dos veces. Si hay duda de si se        → NFR-011    → "Cada intento de pago necesita una         → ADR-004
 cobró o no, que el sistema lo verifique antes de                      idempotency key verificable antes de
 reintentar" (Roberto Sánchez)                                         reintentar el cobro"

"A las 10:00 de la mañana, el stock que ves en la      → NFR-005    → "Se necesita una capa de caché invalidada  → ADR-003
 web es el de las 6:00" (Javier Ortiz)                                 en el mismo request que el descuento
                                                                        de stock, no de forma diferida"

"Sin rutas, los repartidores pueden salir con las      → NFR-009    → "Se necesita un algoritmo de fallback      → ADR-005
 direcciones en orden de código postal... Lo que no                    determinista que no dependa de Google
 puede pasar es que se queden parados" (Pablo Ruiz)                    Maps para poder generar una ruta básica"

"Empezar con EC2 + RDS y escalar después si hace       → (restricción → "Monolito modular desplegado en Auto      → ADR-002
 falta" (Marta Reyes, CEO)                                de negocio,    Scaling Group, sin microservicios ni
                                                          no un NFR)     serverless en el MVP"

"Tardamos 3 días porque buscamos a mano en Sheets.     → REG-001     → "Se necesita un índice materializado      → (Estructural:
 Casi nos sancionan" (Ana Martínez)                                    lote→cliente→pedido, no una consulta         LoteClienteIndex,
                                                                        recorriendo todas las líneas de pedido"      ver [[05-Structural-specs]])
```

**Verificación de ausencia de huérfanos:** todo FR-001..FR-050, NFR-001..NFR-013, REG-001..REG-007 y OPS-001..OPS-006 definido en [[01-Requirements]] aparece en al menos una fila de esta matriz. Todo ADR referenciado (001-005) existe como archivo. Toda spec referenciada (04 a 09) existe como archivo generado en este paquete.
