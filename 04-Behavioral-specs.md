# Especificación Comportamental — OMS FreshDirect

## 1. Máquina de Estados: Pedido

```
CARRITO ──(pago exitoso)──> CONFIRMADO ──(cutoff + picking completo)──> PREPARADO
                                  │                                          │
                                  │ (cancelación cliente,                   │ (rutas aprobadas)
                                  │  antes de cutoff)                       ▼
                                  ▼                                    EN_REPARTO
                              CANCELADO                               │        │
                                                        (entrega OK)  │        │ (cliente ausente /
                                                                       ▼        │  dirección incorrecta)
                                                                  ENTREGADO     ▼
                                                                             FALLIDO ──(reprogramación)──> PREPARADO
```

### Tabla de transiciones válidas

| Desde | Hacia | Trigger | Acción |
|---|---|---|---|
| CARRITO | CONFIRMADO | Pago exitoso en Stripe (FR-013) | Reservar/descontar stock reservado, enviar email+SMS de confirmación |
| CONFIRMADO | CANCELADO | Cliente cancela antes del cutoff (FR-015) | Liberar stock reservado, iniciar reembolso si ya se cobró |
| CONFIRMADO | PREPARADO | Picking completo, todas las líneas escaneadas (FR-023) | Marcar listo para asignación de ruta |
| PREPARADO | EN_REPARTO | Ruta aprobada por Coordinador y repartidor inicia salida (FR-027) | Notificar tracking activo al Cliente |
| EN_REPARTO | ENTREGADO | Confirmación de entrega con foto (FR-032, FR-034) | Generar factura electrónica, cerrar tracking |
| EN_REPARTO | FALLIDO | Cliente ausente tras 5 min + llamada sin respuesta (FR-033) | Marcar productos de frío como no reentregables, notificar a Coordinador |
| FALLIDO | PREPARADO | Reprogramación para el día siguiente | Reasignar a nueva ronda de picking/ruta si aplica (productos ambiente) |

### Tabla de transiciones inválidas

| Desde | Hacia (rechazada) | Razón |
|---|---|---|
| CARRITO | PREPARADO | No se puede preparar un pedido que no ha sido pagado ni confirmado |
| CONFIRMADO | ENTREGADO | No se puede entregar un pedido que no ha pasado por picking ni reparto |
| CONFIRMADO (post-cutoff) | CANCELADO | El cutoff bloquea cambios/cancelación por autoservicio; requiere gestión manual de excepción |
| PREPARADO | CANCELADO | Una vez iniciado el picking, cancelar requiere flujo de excepción con reembolso manual, no transición directa |
| ENTREGADO | * (cualquiera) | Estado terminal; incidencias post-entrega se gestionan como una entidad `Incidencia` separada, no como retroceso del pedido |
| FALLIDO | ENTREGADO | Una entrega fallida no puede marcarse entregada sin pasar de nuevo por reparto |

### Invariante de la máquina de estados (TypeScript)

```typescript
type EstadoPedido =
  | "CARRITO"
  | "CONFIRMADO"
  | "PREPARADO"
  | "EN_REPARTO"
  | "ENTREGADO"
  | "FALLIDO"
  | "CANCELADO";

const VALID_TRANSITIONS: Record<EstadoPedido, EstadoPedido[]> = {
  CARRITO: ["CONFIRMADO"],
  CONFIRMADO: ["PREPARADO", "CANCELADO"],
  PREPARADO: ["EN_REPARTO"],
  EN_REPARTO: ["ENTREGADO", "FALLIDO"],
  FALLIDO: ["PREPARADO"],
  ENTREGADO: [],
  CANCELADO: [],
};

function canTransition(from: EstadoPedido, to: EstadoPedido): boolean {
  return VALID_TRANSITIONS[from].includes(to);
}

function transition(pedido: { estado: EstadoPedido }, to: EstadoPedido): void {
  if (!canTransition(pedido.estado, to)) {
    throw new Error(
      `Transición inválida: ${pedido.estado} -> ${to}`
    );
  }
  pedido.estado = to;
}
```

## 2. Comportamiento ante Fallos de Dependencias Externas

### 2.1 Stripe (pasarela de pago)

- **Detección del fallo:** timeout de red al invocar la API de Stripe, o respuesta de error 5xx.
- **Impacto:** el checkout no puede confirmar el pedido; el resto del sistema (catálogo, tracking, picking de pedidos ya confirmados) no se ve afectado.
- **Secuencia de reintentos/fallback:**
  1. El cliente backend usa una idempotency key generada a partir del ID de la sesión de checkout.
  2. Ante timeout, el sistema NO reintenta automáticamente el cobro; primero consulta el estado real del PaymentIntent en Stripe.
  3. Si el estado es `succeeded`, el pedido se confirma sin nuevo cobro.
  4. Si el estado es `requires_payment_method` o no existe, se permite un reintento manual del cliente, reutilizando la misma idempotency key.
  5. Tras 3 fallos consecutivos en 10 minutos, se bloquea el reintento automático y se exige recargar el checkout.
- **UX resultante:** mensaje "No hemos podido confirmar tu pago. Tu carrito está a salvo, puedes reintentar en unos minutos."
- **Alerta:** métrica `payment_failure_rate` supera el 5% en 5 minutos → alerta a Ingeniería (severidad alta).
- **Runbook asociado:** `RUNBOOK-002` (ver [[07-Operative-specs]]).

### 2.2 Base de datos (PostgreSQL) durante picking

- **Detección del fallo:** la tablet de almacén pierde conexión con el backend (timeout o error de red).
- **Impacto:** sin mitigación, el operario no podría seguir haciendo picking.
- **Secuencia de fallback:**
  1. La tablet mantiene en almacenamiento local (IndexedDB/SQLite embebido) la lista de picking ya descargada al inicio del turno.
  2. Los eventos de escaneo se registran localmente con timestamp y se encolan.
  3. Al recuperar conexión, la tablet sincroniza la cola de eventos en orden, y el backend valida y aplica los descuentos de stock de forma idempotente (por ID de evento).
  4. Si se detecta un conflicto (p. ej. el mismo producto ya fue descontado por otro operario), se prioriza el primer evento por timestamp de servidor y se marca el segundo para revisión manual.
- **UX resultante:** el operario ve un indicador "Modo offline" pero puede seguir escaneando sin interrupción.
- **Alerta:** métrica `tablet_offline_duration` > 15 minutos → alerta al responsable de almacén.
- **Runbook asociado:** `RUNBOOK-001`.

### 2.3 Google Maps API (geocodificación y rutas)

- **Detección del fallo:** error o timeout en las llamadas de geocodificación/routing.
- **Impacto:** no se pueden calcular rutas óptimas; en checkout, no se puede geocodificar una dirección nueva.
- **Secuencia de fallback:**
  1. Checkout: si la geocodificación falla, se usa el código postal introducido manualmente para validar cobertura (lista estática de códigos postales cubiertos) y se marca la dirección como "pendiente de geocodificar" para reintento asíncrono.
  2. Generación de rutas: si el fallo persiste más de 2 minutos, el sistema genera rutas de respaldo ordenando por código postal y orden alfabético de calle, sin optimización de distancia.
- **UX resultante:** el Coordinador ve un aviso "Rutas generadas en modo básico (sin optimización) por indisponibilidad de Google Maps".
- **Alerta:** `maps_api_error_rate` > 20% en 5 minutos → alerta a Ingeniería.
- **Runbook asociado:** `RUNBOOK-003`.

### 2.4 SendGrid / Twilio (email y SMS)

- **Detección del fallo:** error de la API o de entrega reportado por el proveedor.
- **Impacto:** el cliente no recibe confirmación, tracking o alertas sanitarias por ese canal.
- **Secuencia de fallback:** reintento con backoff exponencial (3 intentos); si falla persistentemente, se usa el canal alternativo (si el fallo es en SMS, se refuerza el email, y viceversa); para alertas sanitarias (REG-001), tras 2 fallos en ambos canales se escala a llamada telefónica manual por el Agente de Soporte.
- **Alerta:** cualquier fallo de notificación de alerta sanitaria dispara alerta inmediata de severidad crítica (por el requisito de las 4 horas).
- **Runbook asociado:** `RUNBOOK-004`.

### 2.5 Sensor de temperatura de furgoneta

- **Detección del fallo:** ausencia de lecturas de temperatura durante más de 15 minutos, o lectura por encima de -15°C en zona de congelados durante más de 10 minutos.
- **Impacto:** riesgo de rotura de cadena de frío en productos congelados.
- **Secuencia de fallback:** alerta inmediata al Coordinador de Logística con la furgoneta, ruta y pedidos afectados; el Coordinador decide acelerar la entrega de los pedidos con congelados o cancelarlos y reembolsar.
- **Runbook asociado:** `RUNBOOK-005`.

## 3. Timeouts y Reintentos

| Servicio | Timeout | Reintentos | Backoff | Circuit Breaker |
|---|---|---|---|---|
| Stripe (pago) | 10s | 0 automáticos (se verifica estado antes de reintento manual) | N/A | Sí — abre tras 5 fallos consecutivos en 1 min |
| Stripe (reembolso) | 15s | 3 | Exponencial (2s, 4s, 8s) | Sí — abre tras 5 fallos en 5 min |
| Google Maps (geocodificación) | 3s | 2 | Fijo (1s) | Sí — abre tras 10 fallos en 1 min, fallback a código postal |
| Google Maps (routing) | 5s | 1 | Fijo (2s) | Sí — abre tras 5 fallos en 2 min, fallback a orden por CP |
| SendGrid (email) | 5s | 3 | Exponencial (1s, 3s, 9s) | Sí — abre tras 10 fallos en 5 min |
| Twilio (SMS) | 5s | 3 | Exponencial (1s, 3s, 9s) | Sí — abre tras 10 fallos en 5 min |
| PostgreSQL (consulta estándar) | 2s | 1 | Fijo (500ms) | Sí — abre tras 20 fallos en 30s |
| Sensor de temperatura (ingesta) | 5s | 2 | Fijo (2s) | No aplica (ingesta best-effort con alerta por ausencia de datos) |

## 4. SLI / SLO / SLA

| Operación | SLI (qué se mide) | SLO (objetivo interno) | SLA (compromiso con cliente) |
|---|---|---|---|
| Checkout (confirmar pedido) | % de requests exitosas / p95 latencia | 99.9% éxito, p95 < 500ms | Ninguno formal al cliente final; compromiso interno de negocio |
| Consulta de catálogo/stock | p95 latencia, frescura del dato de stock | p95 < 300ms, staleness < 5s | "Stock mostrado en tiempo real" (comunicación de marketing, no contractual) |
| Disponibilidad general (temporada normal) | Uptime mensual | 99.86% (≤ 1h downtime/mes) | Ninguno externo formal |
| Disponibilidad (temporada alta nov-ene) | Uptime | 100% (0 downtime no planificado) | Compromiso interno crítico del CEO/CTO |
| Notificación de alerta sanitaria | Tiempo desde alerta recibida hasta notificación enviada a todos los afectados | < 4 horas | Obligación regulatoria (REG-001) |
| Inicio de reembolso | Tiempo desde aprobación hasta inicio en Stripe | < 48 horas | Comunicado al cliente en política de devoluciones |
| Resolución de incidencia < 10€ | Tiempo desde reporte hasta resolución | < 1 hora | Comunicado al cliente |
| Resolución de incidencia ≥ 10€ | Tiempo desde reporte hasta resolución | < 24 horas | Comunicado al cliente |

**Cómo el SLO condiciona la arquitectura:** el SLO de disponibilidad de temporada alta (100%, cero downtime no planificado) es el que justifica, frente a un SLO más relajado como 99.5%, decisiones como: RDS Multi-AZ con failover automático, al menos dos instancias EC2 detrás de un load balancer, y un plan de scaling programado antes de noviembre (ver [[06-ADRs/002-ADR-arquitectura-general]] y [[07-Operative-specs]]). Un SLO de 99.5% habría permitido una sola instancia con backup frío; el objetivo real de FreshDirect exige redundancia activa y monitorización proactiva (NFR-006) para poder actuar antes de que el SLO se incumpla, no solo detectarlo después.
