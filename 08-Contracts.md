# Contratos Formales (Design by Contract) — OMS FreshDirect

## 1. Clases de Error de Contrato

```typescript
// Culpa del caller: violó una precondición documentada. El caller debe corregir su entrada.
class PreconditionError extends Error {
  constructor(operacion: string, condicion: string) {
    super(`[Precondición violada] ${operacion}: ${condicion}`);
    this.name = "PreconditionError";
  }
}

// Bug propio: el sistema no cumplió lo que prometió. Indica un defecto interno.
class PostconditionError extends Error {
  constructor(operacion: string, condicion: string) {
    super(`[Postcondición violada] ${operacion}: ${condicion}`);
    this.name = "PostconditionError";
  }
}

// Estado corrupto del sistema: un invariante que debía cumplirse siempre ha sido violado.
// Requiere investigación inmediata — no se debe capturar y continuar silenciosamente.
class InvariantError extends Error {
  constructor(entidad: string, condicion: string) {
    super(`[Invariante violado] ${entidad}: ${condicion}`);
    this.name = "InvariantError";
  }
}
```

## 2. Contrato: Confirmar Pedido (`PedidoService.confirmarPedido`)

Operación crítica que implementa FR-011..FR-014, y garantiza NFR-011 (no doble cobro) y NFR-012 (no sobreventa).

```typescript
/**
 * Precondiciones:
 *   1. El carrito existe y tiene al menos una línea.
 *   2. Todas las líneas del carrito tienen stock reservado vigente (ReservaStock no expirada).
 *   3. La dirección de entrega asociada tiene zonaCobertura = true.
 *   4. El idempotencyKey no ha sido usado previamente con un resultado distinto de "pendiente".
 *
 * Postcondiciones:
 *   1. Si el pago fue exitoso, existe un Pedido en estado CONFIRMADO con el mismo total que el cobro procesado en Stripe.
 *   2. El stock reservado de cada línea queda descontado definitivamente (no solo reservado).
 *   3. Se ha enviado exactamente una notificación de confirmación (email + SMS), nunca cero ni duplicada.
 *   4. Si el pago falla, no existe ningún Pedido en estado CONFIRMADO asociado a ese carrito.
 *
 * Invariantes:
 *   - Un mismo idempotencyKey nunca produce dos cargos distintos en Stripe.
 *   - totalPagado === totalBaseImponible + totalIVA.tipo4 + totalIVA.tipo10 + totalIVA.tipo21 (siempre, sin excepción).
 */
async function confirmarPedido(
  carritoId: string,
  direccionId: string,
  paymentMethodToken: string,
  idempotencyKey: string
): Promise<Pedido> {
  // --- Precondiciones ---
  const carrito = await carritoRepo.obtener(carritoId);
  if (!carrito || carrito.lineas.length === 0) {
    throw new PreconditionError("confirmarPedido", "el carrito no existe o está vacío");
  }

  const reservasValidas = await stockService.verificarReservasVigentes(carrito.lineas);
  if (!reservasValidas) {
    throw new PreconditionError("confirmarPedido", "alguna línea perdió su reserva de stock");
  }

  const direccion = await direccionRepo.obtener(direccionId);
  if (!direccion?.zonaCobertura) {
    throw new PreconditionError("confirmarPedido", "la dirección no está en zona de cobertura");
  }

  const cobroPrevio = await pagoGateway.buscarPorIdempotencyKey(idempotencyKey);
  if (cobroPrevio?.estado === "succeeded") {
    // Ya se cobró antes: no se vuelve a cobrar, se continúa con la confirmación (evita doble cobro).
  }

  // --- Lógica de negocio ---
  const totales = calcularTotalesConIVA(carrito.lineas); // función pura, ver sección 3
  const resultadoPago =
    cobroPrevio?.estado === "succeeded"
      ? cobroPrevio
      : await pagoGateway.cobrar({
          monto: totales.totalPagado,
          paymentMethodToken,
          idempotencyKey,
        });

  if (resultadoPago.estado !== "succeeded") {
    return null as never; // el caller interpreta el rechazo de pago como resultado esperado, no como excepción
  }

  const pedido = await pedidoRepo.crear({
    carritoId,
    direccionId,
    totales,
    estado: "CONFIRMADO",
  });

  await stockService.confirmarDescuentoDefinitivo(carrito.lineas);
  await notificador.enviarConfirmacion(pedido);

  // --- Postcondiciones ---
  const totalEsperado =
    totales.totalBaseImponible +
    totales.totalIVA.tipo4 +
    totales.totalIVA.tipo10 +
    totales.totalIVA.tipo21;
  if (pedido.totalPagado !== totalEsperado) {
    throw new PostconditionError(
      "confirmarPedido",
      "totalPagado no coincide con la suma de base imponible y cuotas de IVA"
    );
  }

  const stockAunReservado = await stockService.tieneReservaPendiente(carrito.lineas);
  if (stockAunReservado) {
    throw new PostconditionError("confirmarPedido", "el stock no quedó descontado definitivamente");
  }

  return pedido;
}
```

## 3. Contrato: Función Pura de Cálculo (`calcularTotalesConIVA`)

Función sin efectos secundarios, implementa FR-011 (REG-007). Incluso una función "trivial" de cálculo tiene contrato.

```typescript
/**
 * Precondiciones:
 *   1. `lineas` no está vacío.
 *   2. Cada línea tiene `tipoIVA` en {4, 10, 21}.
 *   3. Cada línea tiene `cantidad` > 0 y `precioUnitario` >= 0.
 *
 * Postcondiciones:
 *   1. El resultado no tiene efectos secundarios: no modifica `lineas` ni ningún estado externo.
 *   2. totalBaseImponible + suma(totalIVA.*) === suma(cantidad * precioUnitario) de todas las líneas,
 *      con un margen de redondeo de ±0.01€ por línea.
 *   3. Cada cuota de IVA es no negativa.
 *
 * Invariantes:
 *   - Llamar a la función dos veces con el mismo `lineas` produce exactamente el mismo resultado (determinismo).
 */
function calcularTotalesConIVA(lineas: LineaPedido[]): {
  totalBaseImponible: number;
  totalIVA: { tipo4: number; tipo10: number; tipo21: number };
  totalPagado: number;
} {
  if (lineas.length === 0) {
    throw new PreconditionError("calcularTotalesConIVA", "lineas no puede estar vacío");
  }
  for (const linea of lineas) {
    if (![4, 10, 21].includes(linea.tipoIVA)) {
      throw new PreconditionError("calcularTotalesConIVA", `tipoIVA inválido: ${linea.tipoIVA}`);
    }
    if (linea.cantidad <= 0 || linea.precioUnitario < 0) {
      throw new PreconditionError("calcularTotalesConIVA", "cantidad o precioUnitario inválidos");
    }
  }

  const resultado = { totalBaseImponible: 0, totalIVA: { tipo4: 0, tipo10: 0, tipo21: 0 }, totalPagado: 0 };

  for (const linea of lineas) {
    const importeLineaConIVA = linea.cantidad * linea.precioUnitario;
    const base = importeLineaConIVA / (1 + linea.tipoIVA / 100);
    const cuota = importeLineaConIVA - base;

    resultado.totalBaseImponible += base;
    if (linea.tipoIVA === 4) resultado.totalIVA.tipo4 += cuota;
    if (linea.tipoIVA === 10) resultado.totalIVA.tipo10 += cuota;
    if (linea.tipoIVA === 21) resultado.totalIVA.tipo21 += cuota;
    resultado.totalPagado += importeLineaConIVA;
  }

  // --- Postcondición verificable ---
  const sumaCuotas =
    resultado.totalIVA.tipo4 + resultado.totalIVA.tipo10 + resultado.totalIVA.tipo21;
  if (Math.abs(resultado.totalBaseImponible + sumaCuotas - resultado.totalPagado) > 0.01 * lineas.length) {
    throw new PostconditionError(
      "calcularTotalesConIVA",
      "la suma de base imponible y cuotas de IVA no coincide con el total pagado"
    );
  }

  return resultado;
}
```

## 4. Contrato: Escanear Producto en Picking (`PickingService.escanearProducto`)

Implementa FR-018..FR-022, garantiza REG-006 (bloqueo de caducidad) y el invariante FEFO.

```typescript
/**
 * Precondiciones:
 *   1. El pedido existe y está en estado CONFIRMADO (post-cutoff).
 *   2. El SKU escaneado corresponde a una línea PENDIENTE de ese pedido.
 *   3. El loteId escaneado pertenece al SKU indicado.
 *
 * Postcondiciones:
 *   1. Si el lote tiene caducidad > hoy+2 días: la línea pasa a PICKEADA y el stock del lote se descuenta en cantidad exacta.
 *   2. Si el lote tiene caducidad <= hoy+2 días: la operación se rechaza (PreconditionError) y el stock no se modifica.
 *   3. Si el producto está agotado (cantidadDisponible = 0 en todos los lotes válidos) y el Cliente aceptó sustituto:
 *      la línea pasa a SUSTITUIDA con un nuevo SKU de igual categoría y precio <= original.
 *   4. Si el producto está agotado y el Cliente no aceptó sustituto: la línea pasa a CANCELADA y el total del pedido se recalcula.
 *
 * Invariantes:
 *   - cantidadDisponible de un lote nunca es negativa.
 *   - Nunca se asigna a una línea un lote con menos margen de caducidad que otro lote disponible del mismo SKU (FEFO).
 */
async function escanearProducto(
  pedidoId: string,
  sku: string,
  loteId: string
): Promise<LineaPedido> {
  const pedido = await pedidoRepo.obtener(pedidoId);
  if (!pedido || pedido.estado !== "CONFIRMADO") {
    throw new PreconditionError("escanearProducto", "el pedido no está en estado CONFIRMADO");
  }

  const linea = pedido.lineas.find((l) => l.sku === sku && l.estadoLinea === "PENDIENTE");
  if (!linea) {
    throw new PreconditionError("escanearProducto", "no existe línea pendiente para este SKU");
  }

  const lote = await loteRepo.obtener(loteId);
  if (!lote || lote.sku !== sku) {
    throw new PreconditionError("escanearProducto", "el lote no corresponde al SKU escaneado");
  }

  const margenDias = diferenciaEnDias(lote.fechaCaducidad, hoy());
  if (margenDias <= 2) {
    throw new PreconditionError("escanearProducto", "el lote caduca en 2 días o menos (REG-006)");
  }

  const loteFEFO = await loteRepo.obtenerMasProximoACaducar(sku, /*margenMinimoDias=*/ 3);
  if (loteFEFO && loteFEFO.id !== loteId) {
    throw new InvariantError("Lote", `se esperaba recoger el lote ${loteFEFO.id} (FEFO) pero se escaneó ${loteId}`);
  }

  if (lote.cantidadDisponible <= 0) {
    if (linea.aceptaSustituto) {
      const sustituto = await catalogoRepo.buscarSustituto(sku);
      linea.estadoLinea = "SUSTITUIDA";
      linea.sustitutoDeSku = sustituto.sku;
    } else {
      linea.estadoLinea = "CANCELADA";
      await pedidoRepo.recalcularTotal(pedidoId);
    }
    return linea;
  }

  await loteRepo.descontarStock(loteId, linea.cantidad);
  linea.estadoLinea = "PICKEADA";
  linea.loteId = loteId;

  // --- Invariante verificable tras la operación ---
  const loteActualizado = await loteRepo.obtener(loteId);
  if (loteActualizado.cantidadDisponible < 0) {
    throw new InvariantError("Lote", "cantidadDisponible quedó negativa tras el descuento");
  }

  return linea;
}
```

## 5. Interfaces de Dependencias Externas

```typescript
interface PagoGateway {
  cobrar(params: { monto: number; paymentMethodToken: string; idempotencyKey: string }): Promise<{
    estado: "succeeded" | "failed" | "requires_payment_method";
    paymentIntentId: string;
  }>;
  buscarPorIdempotencyKey(key: string): Promise<{ estado: string; paymentIntentId: string } | null>;
  reembolsar(params: { paymentIntentId: string; monto: number }): Promise<{ estado: "iniciado" | "fallido" }>;
}

interface GeoProvider {
  geocodificar(direccion: { calle: string; codigoPostal: string; ciudad: string }): Promise<{
    lat: number;
    lng: number;
    dentroDeCobertura: boolean;
  }>;
  calcularRutaOptima(paradas: { lat: number; lng: number }[]): Promise<{ ordenParadas: number[]; distanciaTotalKm: number }>;
}

interface NotificadorService {
  enviarEmail(destinatario: string, plantilla: string, datos: Record<string, unknown>): Promise<void>;
  enviarSMS(telefono: string, mensaje: string): Promise<void>;
}

interface StockRepository {
  verificarReservasVigentes(lineas: LineaPedido[]): Promise<boolean>;
  confirmarDescuentoDefinitivo(lineas: LineaPedido[]): Promise<void>;
  tieneReservaPendiente(lineas: LineaPedido[]): Promise<boolean>;
}

interface LoteRepository {
  obtener(loteId: string): Promise<Lote | null>;
  obtenerMasProximoACaducar(sku: string, margenMinimoDias: number): Promise<Lote | null>;
  descontarStock(loteId: string, cantidad: number): Promise<void>;
}
```
