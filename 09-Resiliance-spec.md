# Especificación de Fallas, Resiliencia y SLAs — OMS FreshDirect

Principio rector: el fallo es una condición normal de operación. Cada dependencia externa se diseña asumiendo que fallará.

## 1. Patrones de Resiliencia por Dependencia

```yaml
stripe_pagos:
  timeout: 10s
  retry:
    attempts: 0
    backoff: none
    backoff_base: 0s
  circuit_breaker:
    enabled: true
    failure_threshold: 5
    recovery_timeout: 60s
  fallback_behavior:
    checkout_en_curso: "conservar carrito, mostrar mensaje de reintento, verificar estado del PaymentIntent antes de recobrar"
    circuit_abierto: "bloquear checkout con mensaje claro; no se acepta ningún pago sin pasarela (no negociable, Carmen López)"
  impact_when_down: "BLOCKED"

stripe_reembolsos:
  timeout: 15s
  retry:
    attempts: 3
    backoff: "exponential"
    backoff_base: 2s
  circuit_breaker:
    enabled: true
    failure_threshold: 5
    recovery_timeout: 300s
  fallback_behavior:
    circuit_abierto: "encolar el reembolso para reintento automático; el SLA de 48h se mide desde la aprobación, no desde el primer intento fallido"
  impact_when_down: "DEGRADED"

google_maps_geocodificacion:
  timeout: 3s
  retry:
    attempts: 2
    backoff: "fixed"
    backoff_base: 1s
  circuit_breaker:
    enabled: true
    failure_threshold: 10
    recovery_timeout: 60s
  fallback_behavior:
    checkout: "validar cobertura contra lista estática de códigos postales; marcar dirección para geocodificación asíncrona posterior"
  impact_when_down: "DEGRADED"

google_maps_rutas:
  timeout: 5s
  retry:
    attempts: 1
    backoff: "fixed"
    backoff_base: 2s
  circuit_breaker:
    enabled: true
    failure_threshold: 5
    recovery_timeout: 120s
  fallback_behavior:
    generacion_rutas: "ordenar entregas por código postal y calle (sin optimización de distancia); repartidores nunca esperan (NFR-009)"
  impact_when_down: "DEGRADED"

sendgrid_email:
  timeout: 5s
  retry:
    attempts: 3
    backoff: "exponential"
    backoff_base: 1s
  circuit_breaker:
    enabled: true
    failure_threshold: 10
    recovery_timeout: 300s
  fallback_behavior:
    notificacion_estandar: "reforzar canal SMS si el email falla persistentemente"
    alerta_sanitaria: "escalar a llamada telefónica manual del Agente de Soporte tras 2 fallos en ambos canales (REG-001, plazo de 4h)"
  impact_when_down: "DEGRADED"

twilio_sms:
  timeout: 5s
  retry:
    attempts: 3
    backoff: "exponential"
    backoff_base: 1s
  circuit_breaker:
    enabled: true
    failure_threshold: 10
    recovery_timeout: 300s
  fallback_behavior:
    notificacion_estandar: "reforzar canal email si el SMS falla persistentemente"
    alerta_sanitaria: "escalar a llamada telefónica manual tras 2 fallos en ambos canales"
  impact_when_down: "DEGRADED"

postgresql_rds:
  timeout: 2s
  retry:
    attempts: 1
    backoff: "fixed"
    backoff_base: 500ms
  circuit_breaker:
    enabled: true
    failure_threshold: 20
    recovery_timeout: 30s
  fallback_behavior:
    lectura_catalogo: "servir desde caché Redis con TTL corto y aviso de datos potencialmente desactualizados"
    escritura_pedido: "sin fallback; el checkout queda bloqueado hasta que la base de datos primaria (o su réplica promovida) responda"
  impact_when_down: "BLOCKED"

redis_cache_colas:
  timeout: 1s
  retry:
    attempts: 2
    backoff: "fixed"
    backoff_base: 200ms
  circuit_breaker:
    enabled: true
    failure_threshold: 15
    recovery_timeout: 30s
  fallback_behavior:
    lectura_catalogo: "leer directamente de PostgreSQL con rate limiting temporal"
    colas_notificacion: "encolar en memoria local del proceso con persistencia a disco como último recurso, hasta que Redis se recupere"
  impact_when_down: "DEGRADED"

tablet_almacen_conexion:
  timeout: N/A
  retry:
    attempts: "ilimitados mientras dure la desconexión"
    backoff: "exponential"
    backoff_base: 5s
  circuit_breaker:
    enabled: false
  fallback_behavior:
    picking_en_curso: "continuar con lista descargada localmente, encolar eventos de escaneo, sincronizar íntegramente al reconectar (NFR-008)"
  impact_when_down: "MINIMAL"

sensor_temperatura_furgoneta:
  timeout: 5s
  retry:
    attempts: 2
    backoff: "fixed"
    backoff_base: 2s
  circuit_breaker:
    enabled: false
  fallback_behavior:
    ausencia_de_lecturas: "alertar por falta de datos tras 15 minutos sin lectura; el Coordinador decide manualmente si la ruta puede continuar"
  impact_when_down: "DEGRADED"
```

## 2. Jerarquía de Degradación

**critical (siempre funciona, sin excepción):**
- Confirmación de pago y creación de pedido (FR-012, FR-013).
- Descuento de stock durante picking (FR-019).
- Confirmación de entrega (FR-032, FR-034).
- Notificación de alerta sanitaria (FR-045, REG-001) — no se degrada nunca, se escala a proceso manual si los canales automáticos fallan.

**important (se degrada bajo presión, pero no desaparece):**
- Optimización de rutas (FR-025) — degrada a ordenamiento por código postal si Google Maps falla (NFR-009).
- Tracking en tiempo real del cliente (FR-030) — degrada a "estado general" (En preparación/En camino/Entregado) sin ubicación GPS exacta si el proveedor de geolocalización está degradado.
- SMS de "tu pedido llega en 15 minutos" (FR-031) — se omite si Twilio está caído, sin bloquear la entrega.

**nice_to_have (se desactiva primero bajo presión de carga o incidente):**
- Chat con respuestas automáticas (FR-049) — se desactiva y redirige a email/teléfono si el sistema está bajo alta carga.
- Reporte diario de productos próximos a caducar (FR-024) — puede retrasarse unas horas sin impacto operativo inmediato.
- Sugerencias de sustituto "inteligente" más allá de la regla básica de categoría/precio — se limita a la regla determinista si hay presión de carga.

## 3. Feature Flags de Degradación Controlada

| Flag | Default | Descripción | Impacto de desactivarlo |
|---|---|---|---|
| `rutas_optimizacion_maps` | true | Usa Google Maps para optimizar el orden de entrega | Si se desactiva (manual o automáticamente por circuit breaker), rutas se generan por código postal/calle sin optimización de distancia |
| `tracking_gps_tiempo_real` | true | Muestra ubicación GPS exacta del repartidor al cliente | Si se desactiva, el cliente solo ve el estado general del pedido (sin mapa en vivo) |
| `chat_respuestas_automaticas` | true | Habilita el canal de chat con respuestas automáticas | Si se desactiva, el chat redirige a email/teléfono; no afecta a incidencias ya abiertas |
| `sms_notificacion_proximidad` | true | Envía SMS "tu pedido llega en 15 min" | Si se desactiva, el cliente sigue viendo el tracking web pero no recibe el SMS |
| `sugerencia_sustituto_avanzada` | true | Motor de sugerencia de sustituto más allá de la regla básica | Si se desactiva, se aplica solo la regla determinista de categoría/precio (FR-022) |

## 4. SLAs

| Métrica | Objetivo | NFR relacionado |
|---|---|---|
| Disponibilidad temporada normal | 99.86% mensual (≤ 1h downtime/mes) | NFR-003 |
| Disponibilidad temporada alta (nov-ene) | 100% (0 downtime no planificado) | NFR-004 |
| Tiempo de respuesta API — p50 | < 150ms | NFR-001 |
| Tiempo de respuesta API — p95 | < 500ms | NFR-001 |
| Tiempo de respuesta API — p99 | < 1200ms | NFR-001 |
| RTO (Recovery Time Objective) | 1h (normal) / 15 min (temporada alta) | OPS-003 |
| RPO (Recovery Point Objective) | 5 minutos | OPS-003 |
| Notificación de alerta sanitaria | < 4 horas desde recepción de la alerta | REG-001 |
| Inicio de reembolso | < 48 horas desde aprobación | FR-040 |

## 5. Error Budget

| SLO | Presupuesto mensual (minutos) | Política según % de presupuesto restante |
|---|---|---|
| Disponibilidad 99.86% (temporada normal) | 60 minutos/mes | > 50% restante: operación normal. 20-50% restante: precaución, revisar causas raíz de incidentes recientes antes de nuevos despliegues de riesgo. < 20% restante: freeze de despliegues no críticos, solo hotfixes aprobados por el CTO. 0% agotado: stop total de despliegues, foco exclusivo en estabilización, postmortem obligatorio. |
| Disponibilidad 100% (temporada alta, nov-ene) | 0 minutos — presupuesto de error nulo por decisión de negocio | Cualquier incidente de downtime activa automáticamente el protocolo de freeze total de despliegues no críticos y postmortem inmediato con el CTO y la CEO. |

**Nota:** el error budget de temporada alta es intencionalmente cero, reflejando la exigencia explícita del CEO y el CTO ("Zero tolerance. El 23 de diciembre no se puede caer"). Esto implica que todo el trabajo de fiabilidad (Multi-AZ, Auto Scaling, monitorización proactiva) debe estar validado en el simulacro de DR de octubre, antes de que arranque la ventana de error budget cero.
