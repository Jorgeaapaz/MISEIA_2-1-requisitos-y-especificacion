# Especificación Funcional — OMS FreshDirect

Contrato de API visto como caja negra. No se referencian aquí detalles de base de datos, caché ni infraestructura (ver [[05-Structural-specs]] para eso).

## 1. `GET /catalogo/productos`

**Descripción:** Busca productos por nombre, categoría o código de barras, devolviendo stock disponible y precio con IVA. Implementa FR-001, FR-002.

**Parámetros de entrada (query string):**

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `q` | string | No | Texto de búsqueda libre (nombre) |
| `categoria` | string | No | Filtro por categoría |
| `codigoBarras` | string | No | Búsqueda exacta por código de barras |
| `page` | number | No | Página de resultados (default 1) |

**Respuesta exitosa (200):**

```typescript
interface CatalogoProductosResponse {
  productos: {
    sku: string;
    nombre: string;
    categoria: string;
    precioConIVA: number;
    tipoIVA: 4 | 10 | 21;
    stockDisponible: number;
    tipoTemperatura: "AMBIENTE" | "REFRIGERADO" | "CONGELADO";
  }[];
  totalResultados: number;
  page: number;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 400 | Ningún parámetro de búsqueda válido enviado | "Debes indicar un término de búsqueda, categoría o código de barras" |

---

## 2. `POST /carrito/lineas`

**Descripción:** Añade una línea de producto al carrito, indicando si se acepta sustituto. Implementa FR-003, FR-004.

**Body de entrada:**

```typescript
interface AnadirLineaCarritoRequest {
  carritoId: string;
  sku: string;               // obligatorio
  cantidad: number;          // obligatorio, > 0
  aceptaSustituto: boolean;  // obligatorio
}
```

**Respuesta exitosa (200):**

```typescript
interface CarritoResponse {
  carritoId: string;
  lineas: { sku: string; cantidad: number; aceptaSustituto: boolean; precioConIVA: number }[];
  subtotal: number;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 404 | SKU no existe en catálogo | "Producto no encontrado" |
| 409 | Cantidad solicitada supera el stock disponible | "No hay suficiente stock disponible para este producto" |

---

## 3. `POST /checkout/validar-direccion`

**Descripción:** Geocodifica una dirección y valida cobertura. Implementa FR-006, FR-007, FR-050 (peso).

**Body de entrada:**

```typescript
interface ValidarDireccionRequest {
  calle: string;
  piso?: string;
  codigoPostal: string;
  ciudad: string;
  instruccionesEntrega?: string;
}
```

**Respuesta exitosa (200):**

```typescript
interface ValidarDireccionResponse {
  direccionId: string;
  zonaCobertura: true;
  franjasDisponibles: ("MANANA" | "TARDE" | "EXPRESS")[];
  costeEnvio: number;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 422 | Dirección fuera de España peninsular o de la zona de cobertura | "Todavía no entregamos en esta zona" |
| 502 | Proveedor de geolocalización no disponible | "No hemos podido validar la dirección, inténtalo de nuevo en unos minutos" |

---

## 4. `POST /checkout/aplicar-descuento`

**Descripción:** Valida y aplica un código de descuento al carrito. Implementa FR-009, FR-010.

**Body de entrada:**

```typescript
interface AplicarDescuentoRequest {
  carritoId: string;
  codigo: string;
}
```

**Respuesta exitosa (200):**

```typescript
interface AplicarDescuentoResponse {
  codigo: string;
  descuentoAplicado: number;
  nuevoTotal: number;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 404 | Código no existe | "Código de descuento no válido" |
| 422 | Código caducado | "Este código ha caducado" |
| 422 | Código ya usado (uso único) | "Este código ya ha sido utilizado" |
| 422 | Importe mínimo del pedido no alcanzado | "El importe mínimo para este descuento es de {importeMinimo}€" |

---

## 5. `POST /checkout/confirmar-pedido`

**Descripción:** Calcula el total con IVA desglosado, procesa el pago y confirma el pedido. Implementa FR-011, FR-012, FR-013, FR-014.

**Body de entrada:**

```typescript
interface ConfirmarPedidoRequest {
  carritoId: string;
  direccionId: string;
  franjaEntrega: "MANANA" | "TARDE" | "EXPRESS";
  metodoPago: "TARJETA" | "BIZUM";
  paymentMethodToken: string; // token generado por Stripe.js en el cliente
  idempotencyKey: string;     // provista por el frontend para evitar doble cobro
}
```

**Respuesta exitosa (201):**

```typescript
interface ConfirmarPedidoResponse {
  pedidoId: string;
  estado: "CONFIRMADO";
  totalBaseImponible: number;
  totalIVA: { tipo4: number; tipo10: number; tipo21: number };
  totalPagado: number;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 402 | Pago rechazado por la pasarela | "El pago no ha podido procesarse, revisa los datos de tu tarjeta" |
| 409 | Alguna línea perdió stock durante el checkout (última unidad ganada por otro cliente) | "Uno de los productos ya no está disponible; se ha aplicado el sustituto o se ha eliminado la línea" |
| 503 | Pasarela de pago no disponible | "No hemos podido confirmar tu pago. Tu carrito está guardado, puedes reintentar en unos minutos" |

---

## 6. `PATCH /pedidos/{pedidoId}`

**Descripción:** Modifica o cancela un pedido antes del cutoff. Implementa FR-015, FR-016.

**Body de entrada:**

```typescript
interface ModificarPedidoRequest {
  accion: "MODIFICAR" | "CANCELAR";
  lineas?: { sku: string; cantidad: number }[];
  franjaEntrega?: "MANANA" | "TARDE" | "EXPRESS";
}
```

**Respuesta exitosa (200):**

```typescript
interface ModificarPedidoResponse {
  pedidoId: string;
  estado: "CONFIRMADO" | "CANCELADO";
  totalPagado: number;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 403 | Ya pasó el cutoff de las 22:00 | "Este pedido ya está en preparación y no admite cambios" |
| 404 | Pedido no existe o no pertenece al Cliente | "Pedido no encontrado" |

---

## 7. `GET /almacen/picking/{operarioId}/siguiente`

**Descripción:** Devuelve la siguiente lista de picking asignada al operario, ordenada por pasillo. Implementa FR-017, FR-020, FR-023.

**Respuesta exitosa (200):**

```typescript
interface ListaPickingResponse {
  pedidoId: string;
  lineas: {
    sku: string;
    nombre: string;
    ubicacionAlmacen: string;
    loteRecomendadoId: string;
    zonaTemperatura: "AMBIENTE" | "REFRIGERADO" | "CONGELADO";
    ordenRecogida: number;
  }[];
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 204 | No hay pedidos pendientes de picking para este operario | (sin contenido) |

---

## 8. `POST /almacen/picking/{pedidoId}/escanear`

**Descripción:** Registra el escaneo de un producto durante el picking y descuenta stock en tiempo real. Implementa FR-018, FR-019, FR-021, FR-022.

**Body de entrada:**

```typescript
interface EscanearProductoRequest {
  operarioId: string;
  sku: string;
  loteId: string;
  timestampEscaneo: string; // ISO 8601, generado en cliente para soportar modo offline
  eventoId: string;          // UUID generado en cliente, usado para idempotencia en sincronización offline
}
```

**Respuesta exitosa (200):**

```typescript
interface EscanearProductoResponse {
  lineaEstado: "PICKEADA" | "SUSTITUIDA" | "CANCELADA";
  sustitutoPropuesto?: { sku: string; nombre: string };
  pedidoCompletado: boolean;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 409 | El lote escaneado caduca en ≤ 2 días | "Este lote no se puede enviar, caduca en menos de 3 días. Usa el siguiente lote sugerido" |
| 409 | El SKU escaneado no corresponde a ninguna línea pendiente del pedido | "Este producto no pertenece a este pedido" |
| 422 | Producto agotado y Cliente no aceptó sustituto | "Línea cancelada por falta de stock; el cliente no aceptó sustitutos para este producto" |

---

## 9. `POST /logistica/rutas/generar`

**Descripción:** Genera las rutas propuestas para una franja de reparto. Implementa FR-025, FR-026.

**Body de entrada:**

```typescript
interface GenerarRutasRequest {
  fecha: string;
  franja: "MANANA" | "TARDE";
}
```

**Respuesta exitosa (200):**

```typescript
interface GenerarRutasResponse {
  rutas: {
    rutaId: string;
    repartidorSugeridoId?: string;
    pedidosIds: string[];
    entregasEstimadas: number;
    optimizada: boolean; // false si se generó en modo fallback por caída de Google Maps
  }[];
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 200 (con `optimizada: false`) | Proveedor de rutas no disponible | (respuesta exitosa degradada, no error) |
| 400 | No hay pedidos PREPARADOS para esa fecha/franja | "No hay pedidos pendientes de asignación para esta franja" |

---

## 10. `PATCH /logistica/rutas/{rutaId}/aprobar`

**Descripción:** El Coordinador aprueba (con ajustes opcionales) una ruta propuesta. Implementa FR-027, FR-028.

**Body de entrada:**

```typescript
interface AprobarRutaRequest {
  coordinadorId: string;
  pedidosIds: string[]; // lista final tras ajustes manuales
  repartidorId: string;
}
```

**Respuesta exitosa (200):**

```typescript
interface AprobarRutaResponse {
  rutaId: string;
  estado: "APROBADA";
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 422 | La ruta ajustada supera 40 entregas | "Una ruta no puede superar 40 entregas" |

---

## 11. `GET /pedidos/{pedidoId}/tracking`

**Descripción:** Devuelve el estado y ubicación en tiempo real del pedido en reparto. Implementa FR-030.

**Respuesta exitosa (200):**

```typescript
interface TrackingResponse {
  pedidoId: string;
  estado: "PENDIENTE" | "EN_CAMINO" | "EN_PUERTA" | "ENTREGADO" | "FALLIDO";
  minutosEstimados?: number;
  ubicacionAproximada?: { lat: number; lng: number };
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 404 | El pedido no está en fase de reparto | "El tracking no está disponible todavía para este pedido" |

---

## 12. `POST /reparto/{pedidoId}/confirmar-entrega`

**Descripción:** El Repartidor confirma la entrega con foto o marca la entrega como fallida. Implementa FR-032, FR-033, FR-034, FR-036.

**Body de entrada:**

```typescript
interface ConfirmarEntregaRequest {
  repartidorId: string;
  resultado: "ENTREGADO" | "FALLIDO";
  fotoUrl?: string;          // obligatorio si resultado = ENTREGADO
  motivoFallo?: "CLIENTE_AUSENTE" | "DIRECCION_INCORRECTA";
  incidenciaProductoDanado?: { sku: string; fotoUrl: string };
}
```

**Respuesta exitosa (200):**

```typescript
interface ConfirmarEntregaResponse {
  pedidoId: string;
  estado: "ENTREGADO" | "FALLIDO";
  facturaId?: string; // presente si estado = ENTREGADO
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 400 | `resultado = ENTREGADO` sin `fotoUrl` | "Se requiere una foto para confirmar la entrega" |

---

## 13. `POST /incidencias`

**Descripción:** El Cliente reporta una incidencia post-entrega. Implementa FR-037, FR-038, FR-039.

**Body de entrada:**

```typescript
interface CrearIncidenciaRequest {
  pedidoId: string;
  tipo: "PRODUCTO_DANADO" | "PRODUCTO_EQUIVOCADO" | "PRODUCTO_FALTANTE";
  sku: string;
  fotoUrl: string;
}
```

**Respuesta exitosa (201):**

```typescript
interface CrearIncidenciaResponse {
  incidenciaId: string;
  resolucion: "REEMBOLSO_AUTOMATICO" | "PENDIENTE";
  reembolsoIniciadoEn?: string; // fecha estimada, presente si REEMBOLSO_AUTOMATICO
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 404 | El pedido no existe o no está ENTREGADO | "No se puede reportar una incidencia sobre este pedido" |
| 422 | Falta la foto adjunta | "Es necesario adjuntar una foto para reportar la incidencia" |

---

## 14. `POST /calidad/alertas-sanitarias/{loteId}/notificar`

**Descripción:** Identifica y notifica a los clientes afectados por un lote en una alerta sanitaria. Implementa FR-044, FR-045.

**Respuesta exitosa (200):**

```typescript
interface NotificarAlertaSanitariaResponse {
  loteId: string;
  clientesAfectados: number;
  notificacionesEnviadas: number;
  notificacionesFallidas: number;
  tiempoTranscurridoMinutos: number;
}
```

**Errores:**

| Código | Condición | Mensaje |
|---|---|---|
| 404 | El lote no aparece en ningún pedido | "No se han encontrado clientes afectados por este lote" |
