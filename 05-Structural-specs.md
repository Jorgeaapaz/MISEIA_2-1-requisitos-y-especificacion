# Especificación Estructural — OMS FreshDirect

## 1. Arquitectura de Alto Nivel

```
┌─────────────┐   ┌──────────────┐   ┌───────────────┐
│  Web Cliente │   │  Web/Tablet  │   │  Web Móvil    │
│  (React)     │   │  Almacén     │   │  Repartidor   │
└──────┬───────┘   └──────┬───────┘   └──────┬────────┘
       │                  │                   │
       └────────┬─────────┴─────────┬─────────┘
                 │                   │
         ┌───────▼───────────────────▼────────┐
         │      API Gateway / Load Balancer     │
         │      (ALB) + Rate Limiting            │
         └───────┬───────────────────┬──────────┘
                 │                   │
   ┌─────────────▼────────┐ ┌────────▼──────────────┐
   │  API REST (Node.js /  │ │  Servicio de Rutas y   │
   │  TypeScript, EC2 x N) │ │  Tracking (Node.js)    │
   │  - Catálogo/Pedidos   │ │  - Asignación de rutas │
   │  - Picking            │ │  - Posición repartidor │
   │  - Incidencias        │ │                        │
   │  - Facturación        │ │                        │
   └─────────┬─────────────┘ └───────────┬────────────┘
             │                            │
   ┌─────────▼────────────────────────────▼──────────┐
   │         Cola de eventos (Redis / BullMQ)          │
   │  (descuentos de stock, notificaciones, export)    │
   └─────────┬──────────────────────────────┬─────────┘
             │                              │
   ┌─────────▼─────────┐          ┌─────────▼──────────┐
   │  PostgreSQL (RDS)  │          │  Redis (caché +     │
   │  Multi-AZ           │          │  colas + sesión)    │
   └────────────────────┘          └─────────────────────┘

   ┌───────────────────────── Integraciones externas ─────────────────────────┐
   │  Stripe (pagos)  │ Google Maps (geo/rutas) │ SendGrid (email) │ Twilio(SMS)│
   │  Export nocturno CSV → SAP (job programado 02:00)                         │
   └────────────────────────────────────────────────────────────────────────────┘
```

## 2. Módulos y Dependencias

```typescript
// Servicio de Pedidos: orquesta el ciclo de vida del pedido (FR-013..FR-016, FR-023, FR-034)
class PedidoService {
  constructor(
    private readonly pedidoRepo: PedidoRepository,
    private readonly stockService: StockService,
    private readonly pagoGateway: PagoGateway,
    private readonly notificador: NotificadorService,
    private readonly eventBus: EventBus
  ) {}
}

// Servicio de Stock: única fuente de verdad del inventario en tiempo real (FR-005, FR-019, FR-022)
class StockService {
  constructor(
    private readonly stockRepo: StockRepository,
    private readonly cache: CacheClient, // Redis: lectura rápida de catálogo
    private readonly eventBus: EventBus
  ) {}
}

// Servicio de Picking: genera y valida las listas de picking (FR-017..FR-024)
class PickingService {
  constructor(
    private readonly picklistRepo: PicklistRepository,
    private readonly stockService: StockService,
    private readonly loteRepo: LoteRepository, // FEFO y trazabilidad
    private readonly eventBus: EventBus
  ) {}
}

// Servicio de Rutas: genera y optimiza asignaciones de reparto (FR-025..FR-029)
class RutasService {
  constructor(
    private readonly rutaRepo: RutaRepository,
    private readonly geoProvider: GeoProvider, // Google Maps con fallback
    private readonly eventBus: EventBus
  ) {}
}

// Servicio de Incidencias: gestiona reembolsos y reclamaciones (FR-036..FR-040)
class IncidenciaService {
  constructor(
    private readonly incidenciaRepo: IncidenciaRepository,
    private readonly pagoGateway: PagoGateway,
    private readonly pedidoRepo: PedidoRepository
  ) {}
}

// Servicio de Trazabilidad: soporta alertas sanitarias en <4h (FR-044, FR-045)
class TrazabilidadService {
  constructor(
    private readonly loteRepo: LoteRepository,
    private readonly pedidoRepo: PedidoRepository,
    private readonly notificador: NotificadorService
  ) {}
}

// Servicio de Facturación: genera facturas electrónicas y exporta a SAP (FR-042, FR-043)
class FacturacionService {
  constructor(
    private readonly facturaRepo: FacturaRepository,
    private readonly pedidoRepo: PedidoRepository,
    private readonly exportadorCSV: ExportadorSAP
  ) {}
}
```

## 3. Modelo de Datos

```typescript
interface Cliente {
  id: string;
  nombre: string;          // PII
  email: string;           // PII
  telefono: string;        // PII
  nif?: string;            // PII, requerido solo para factura con NIF
  direcciones: Direccion[]; // PII
  sustitutosAceptados: Record<string, boolean>; // por producto/categoría
  reembolsosAutomaticosMes: { mes: string; conteo: number }; // control antifraude FR-038
}

interface Direccion {
  id: string;
  clienteId: string;
  calle: string;           // PII
  piso?: string;           // PII
  codigoPostal: string;
  ciudad: string;
  instruccionesEntrega?: string; // PII (puede incluir código de portero)
  lat?: number;
  lng?: number;
  zonaCobertura: boolean;  // resultado de validación de cobertura (FR-007)
}

interface Producto {
  sku: string;
  nombre: string;
  categoria: string;
  tipoTemperatura: "AMBIENTE" | "REFRIGERADO" | "CONGELADO";
  tipoIVA: 4 | 10 | 21;    // REG-007
  precio: number;
  ubicacionAlmacen: string; // ej. "A3-N2-P15"
  permiteSustituto: boolean;
}

interface Lote {
  id: string;
  sku: string;
  proveedorId: string;
  fechaCaducidad: string;   // ISO date
  cantidadRecibida: number;
  cantidadDisponible: number;
  fechaRecepcion: string;
}

interface Pedido {
  id: string;
  clienteId: string;
  estado: "CARRITO" | "CONFIRMADO" | "PREPARADO" | "EN_REPARTO" | "ENTREGADO" | "FALLIDO" | "CANCELADO";
  lineas: LineaPedido[];
  direccionEntregaId: string;
  franjaEntrega: "MANANA" | "TARDE" | "EXPRESS";
  totalBaseImponible: number;
  totalIVA: { tipo4: number; tipo10: number; tipo21: number };
  totalPagado: number;
  codigoDescuentoId?: string;
  pesoTotalKg: number;
  envioDivididoEnId?: string; // FR-050: referencia al envío hermano si se dividió por peso
  cutoffAplicado: boolean;
  createdAt: string;
  confirmedAt?: string;
}

interface LineaPedido {
  id: string;
  pedidoId: string;
  sku: string;
  loteId?: string;          // asignado durante picking (FEFO)
  cantidad: number;
  precioUnitario: number;
  tipoIVA: 4 | 10 | 21;
  aceptaSustituto: boolean;
  sustitutoDeSku?: string;   // si esta línea es un sustituto de otra
  estadoLinea: "PENDIENTE" | "PICKEADA" | "SUSTITUIDA" | "CANCELADA";
}

interface EventoPicking {
  id: string;
  pedidoId: string;
  operarioId: string;
  sku: string;
  loteId: string;
  timestampEscaneo: string;
  origenOffline: boolean;   // true si se generó en modo offline y se sincronizó después (NFR-008)
}

interface Ruta {
  id: string;
  repartidorId: string;
  fecha: string;
  franja: "MANANA" | "TARDE";
  pedidosIds: string[];      // 30-40 por ruta (FR-026)
  estado: "PROPUESTA" | "APROBADA" | "EN_CURSO" | "COMPLETADA";
  aprobadaPorCoordinadorId?: string;
}

interface EntregaTracking {
  pedidoId: string;
  rutaId: string;
  estado: "PENDIENTE" | "EN_CAMINO" | "EN_PUERTA" | "ENTREGADO" | "FALLIDO";
  ubicacionRepartidorLat?: number;
  ubicacionRepartidorLng?: number;
  fotoEntregaUrl?: string;
  horaEstimada: string;
  horaReal?: string;
}

interface LecturaTemperaturaFurgoneta {
  id: string;
  furgonetaId: string;
  timestamp: string;
  temperaturaCelsius: number;
  zona: "REFRIGERADO" | "CONGELADO";
}

interface Incidencia {
  id: string;
  pedidoId: string;
  clienteId: string;
  tipo: "PRODUCTO_DANADO" | "PRODUCTO_EQUIVOCADO" | "PRODUCTO_FALTANTE";
  importe: number;
  fotoUrl: string;
  resolucion: "REEMBOLSO_AUTOMATICO" | "REEMBOLSO_MANUAL" | "RECHAZADO" | "PENDIENTE";
  agenteId?: string;
  createdAt: string;
  resueltaAt?: string;
}

interface Factura {
  id: string;
  pedidoId: string;
  clienteNif?: string;      // PII
  baseImponible4: number;
  cuotaIVA4: number;
  baseImponible10: number;
  cuotaIVA10: number;
  baseImponible21: number;
  cuotaIVA21: number;
  total: number;
  fechaEmision: string;
}

// Derivada de necesidad no obvia: alerta sanitaria exige localizar clientes por lote en <4h (REG-001)
// Índice/tabla materializada para no tener que recorrer todas las líneas de pedido en producción.
interface LoteClienteIndex {
  loteId: string;
  pedidoId: string;
  clienteId: string;
  fechaEntrega?: string;
}

// Derivada de NFR-012 (concurrencia sobre última unidad): reserva atómica de stock
interface ReservaStock {
  id: string;
  sku: string;
  pedidoId: string;
  cantidad: number;
  expiraEn: string; // reserva temporal durante el checkout, liberada si el pago no se completa
}

// Derivada de REG-002/REG-003: auditoría de accesos a datos sensibles (PII, pagos)
interface AuditLog {
  id: string;
  actorId: string;
  actorRol: string;
  accion: string;         // ej. "VER_DATOS_CLIENTE", "PROCESAR_REEMBOLSO"
  entidadTipo: string;
  entidadId: string;
  timestamp: string;
}
```

## 4. Infraestructura

| Componente | Tecnología | Justificación |
|---|---|---|
| Cómputo backend | AWS EC2 (Auto Scaling Group, ≥2 instancias) | Restricción explícita del CEO: sin serverless ni Kubernetes en el MVP; ASG cubre NFR-002 (pico de Navidad) escalando horizontalmente sin cambiar de paradigma. Merece ADR ([[ADR-002]]). |
| Base de datos | AWS RDS PostgreSQL Multi-AZ | Sustituye MySQL sin réplica (causa raíz de caídas actuales); Multi-AZ cubre NFR-004 (cero downtime en temporada alta). Merece ADR ([[ADR-001]]). |
| Caché y colas | Redis (AWS ElastiCache) | Cubre NFR-005 (propagación de stock <5s) mediante invalidación de caché de catálogo, y desacopla notificaciones/exportes vía colas (BullMQ) para no bloquear el flujo de checkout. Merece ADR ([[ADR-003]]). |
| Balanceo de carga | AWS Application Load Balancer | Distribuye tráfico entre instancias EC2 del ASG; health checks habilitan el rollback automático de CI/CD (OPS-001). |
| Almacenamiento de fotos | AWS S3 | Fotos de entrega (FR-032) y de incidencias (FR-036, FR-037) requieren almacenamiento durable y barato, con URLs firmadas para acceso controlado (GDPR). |
| Frontend Cliente | React (SPA servida vía CloudFront + S3) | Stack decidido por CTO; CDN reduce latencia percibida en catálogo. |
| App almacén/repartidor | Web responsive con Service Worker | Cubre NFR-008 (modo offline en tablet) sin requerir desarrollo de app nativa en el MVP. |
| Pasarela de pago | Stripe | Decisión explícita del CTO; delega PCI DSS (REG-003). Merece ADR ([[ADR-004]]). |
| Geolocalización/Rutas | Google Maps API | Decisión explícita del CTO para geocodificación, cálculo de rutas y tracking. |
| Email transaccional | SendGrid | Decisión explícita del CTO. |
| SMS | Twilio | Decisión explícita del CTO para notificaciones de tracking y verificación. |
| Export contable | Job programado (cron en EC2 o AWS EventBridge + Lambda puntual) → CSV → SAP | OPS-004; no requiere integración en tiempo real, solo un job nocturno fiable con reintento. |
| Observabilidad | CloudWatch + alerting (o Prometheus/Grafana autogestionado) | Cubre NFR-006 y OPS-002 (monitorización proactiva, explícitamente pedida por el CTO tras el histórico de caídas sin visibilidad). |
