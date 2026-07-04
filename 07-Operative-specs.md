# Especificación Operativa — OMS FreshDirect

Deriva directamente de los requisitos OPS-001..OPS-006.

## 1. Pipeline de CI/CD (OPS-001)

```yaml
# .github/workflows/deploy.yml (pseudo-YAML)
name: oms-pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-test-build:
    steps:
      - checkout
      - install-dependencies
      - run: npm run lint
      - run: npm run test -- --coverage
      - run: npm run build
      - fail-fast: true   # cualquier fallo detiene el pipeline

  deploy-staging:
    needs: lint-test-build
    if: github.ref == 'refs/heads/main'
    steps:
      - deploy-to: staging (EC2 ASG staging)
      - run-smoke-tests
      - run: k6 run load-test-basico.js   # valida NFR-001 antes de producción

  approval-gate:
    needs: deploy-staging
    steps:
      - require-manual-approval:
          approvers: [CTO, Lead Backend]

  deploy-production:
    needs: approval-gate
    steps:
      - deploy-to: production (EC2 ASG prod, rolling deployment)
      - health-check:
          endpoint: /health
          timeout: 60s
          failure_action: rollback-automatico
      - notify: canal-deploys-slack
```

**Regla de rollback:** si el health check post-despliegue falla 3 veces en 60 segundos, el pipeline revierte automáticamente a la versión anterior y notifica al canal de guardia.

**Congelación de despliegues:** entre el 1 de noviembre y el 31 de enero solo se permiten despliegues críticos (hotfixes de seguridad o de disponibilidad) aprobados explícitamente por el CTO, en línea con NFR-004.

## 2. Métricas y Alertas (OPS-002)

### Métricas expuestas (formato Prometheus)

```
http_request_duration_seconds{route, method, status}
http_requests_total{route, method, status}
checkout_confirmations_total{status="success|failed"}
payment_gateway_errors_total{provider="stripe"}
stock_deduction_latency_seconds
picking_events_total{origen="online|offline_sync"}
picking_time_seconds{pedido_id}
route_generation_duration_seconds{modo="optimizada|fallback"}
delivery_confirmation_pending_seconds{pedido_id}
cold_chain_temperature_celsius{furgoneta_id, zona}
incident_resolution_time_seconds{tipo, importe_bucket}
db_connections_active
redis_queue_depth{cola}
```

### Reglas de alerta

| Alerta | Expresión | Umbral | Duración | Severidad | Runbook |
|---|---|---|---|---|---|
| Tasa de error de pago elevada | `rate(payment_gateway_errors_total[5m]) / rate(checkout_confirmations_total[5m])` | > 5% | 5 min | Alta | RUNBOOK-002 |
| Latencia de API degradada | `histogram_quantile(0.95, http_request_duration_seconds)` | > 500ms | 5 min | Media | RUNBOOK-006 |
| Riesgo de saturación de recursos (proactiva) | `db_connections_active / max_connections` o `cpu_usage` | > 80% | 5 min | Alta (preventiva, NFR-006) | RUNBOOK-006 |
| Tablet de almacén offline prolongado | `tablet_offline_duration_seconds` | > 900s (15 min) | — | Media | RUNBOOK-001 |
| Fallo de proveedor de rutas | `rate(route_generation_duration_seconds{modo="fallback"}[5m])` | > 0 sostenido 2 min | 2 min | Media | RUNBOOK-003 |
| Entrega sin confirmar | `delivery_confirmation_pending_seconds` | > 7200s (2h) | — | Alta | RUNBOOK-007 |
| Rotura de cadena de frío | `cold_chain_temperature_celsius{zona="CONGELADO"}` | > -15 | 10 min | Crítica | RUNBOOK-005 |
| Fallo de notificación de alerta sanitaria | evento `notificacion_sanitaria_fallida` | ≥ 1 ocurrencia | — | Crítica | RUNBOOK-004 |
| Cola de eventos acumulándose | `redis_queue_depth` | > 1000 | 5 min | Media | RUNBOOK-006 |

## 3. Runbooks

### RUNBOOK-001: Tablet de Almacén Offline Prolongado

**Síntomas:** Alerta `tablet_offline_duration_seconds > 900`. El operario reporta que la app sigue mostrando "Modo offline" más de 15 minutos.

**Diagnóstico paso a paso:**
1. Verificar el estado de la red del almacén: `ping <gateway_wifi_almacen>`.
2. Comprobar el estado del backend: `curl -f https://api.freshdirect.internal/health`.
3. Revisar logs de la tablet específica en el panel de monitorización de dispositivos: buscar `device_id` y el último `heartbeat`.
4. Comprobar si el problema es aislado (una tablet) o generalizado (todo el almacén).

**Remediación:**
- Si es un problema de red local: reiniciar el punto de acceso Wi-Fi del almacén (procedimiento físico, responsable de turno).
- Si es un problema de backend: seguir RUNBOOK-006 (degradación de API).
- Si es la tablet individual: reiniciar la app; los eventos de picking en cola local no se pierden (NFR-008) y se sincronizan al reconectar.

**Criterios de escalada:** si tras 30 minutos el problema persiste y afecta a más de 3 tablets simultáneamente, escalar a guardia de Ingeniería (severidad alta) — riesgo de retraso en el cutoff de picking de las 6:00.

---

### RUNBOOK-002: Tasa de Error de Pago Elevada (Stripe)

**Síntomas:** Alerta de `payment_gateway_errors_total` > 5% durante 5 minutos. Aumento de tickets de soporte sobre "no puedo pagar".

**Diagnóstico paso a paso:**
1. Consultar el [Status Page de Stripe](https://status.stripe.com) para descartar incidente del proveedor.
2. Revisar logs de la API de pagos filtrando por `payment_gateway_errors_total` y el código de error devuelto por Stripe (`card_declined`, `api_connection_error`, etc.).
3. Verificar si el error es específico de un método de pago (tarjeta vs. Bizum) o generalizado.
4. Comprobar que las claves de API de Stripe no hayan expirado o rotado incorrectamente en el último despliegue.

**Remediación:**
- Si es un incidente de Stripe: activar el mensaje de aviso en checkout ("Estamos teniendo problemas con los pagos, inténtalo en unos minutos") y monitorizar el status page hasta resolución.
- Si es un error de configuración (claves, webhook): revertir el último despliegue relacionado con pagos.
- Verificar que el circuit breaker de Stripe se ha abierto correctamente (no se están enviando reintentos agresivos que empeoren la situación).

**Criterios de escalada:** si la tasa de error supera el 20% o persiste más de 15 minutos, escalar a CTO y notificar a atención al cliente para que active mensaje proactivo a clientes en checkout.

## 4. Backup y Disaster Recovery (OPS-003)

- **Backups de base de datos:** snapshot automático completo diario de RDS a las 03:00 (fuera de horas de mayor tráfico), con snapshots incrementales (WAL) continuos. Retención: 35 días para diarios, 12 meses para snapshots mensuales.
- **Backups de S3 (fotos de entrega/incidencias):** versionado habilitado + replicación cross-region a una segunda región de AWS.
- **RTO (Recovery Time Objective):** 1 hora en temporada normal, 15 minutos en temporada alta (nov-ene), coherente con NFR-003/NFR-004.
- **RPO (Recovery Point Objective):** 5 minutos (dado por la frecuencia de WAL shipping de RDS Multi-AZ).
- **Simulacro de DR:** trimestral, y obligatorio en octubre (previo a la temporada alta). Procedimiento: promover una réplica Multi-AZ a primaria en un entorno de staging aislado, medir el tiempo real de failover y validar la integridad de los últimos 100 pedidos frente al primario original.
- **Criterios de éxito del simulacro:** failover completado en ≤ RTO objetivo, 0 pérdida de pedidos confirmados (dentro del RPO), checklist de validación de integridad de datos firmado por Ingeniería y Operaciones.

## 5. Logging

**Campos estándar de log (formato JSON estructurado):**

```
timestamp, nivel, servicio, requestId, actorId (si aplica), actorRol, ruta, metodo, statusCode, latenciaMs, mensaje
```

**NUNCA debe loguearse:**
- Número completo de tarjeta, CVV, o cualquier dato de pago sensible (REG-003, NFR-007) — solo el `paymentIntentId` de Stripe.
- Contraseñas o tokens de sesión en texto plano.
- Contenido completo de direcciones o teléfonos en logs de aplicación de nivel INFO/DEBUG (se registran solo IDs de entidad; el dato completo vive únicamente en la base de datos y en el audit log controlado).
- NIF completo del cliente fuera del audit log de facturación.

**Qué sí va a un audit log separado (acceso restringido, ver `AuditLog` en [[05-Structural-specs]]):**
- Accesos de Agentes de Soporte y Administradores a datos personales de clientes (GDPR, REG-002).
- Aprobaciones de reembolsos manuales.
- Consultas de trazabilidad de lote durante una alerta sanitaria (REG-001), incluyendo quién consultó y cuándo.
- Cambios de precio, catálogo y códigos de descuento realizados por Administradores.
