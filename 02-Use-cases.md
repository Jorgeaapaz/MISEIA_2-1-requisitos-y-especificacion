# Casos de Uso — OMS FreshDirect

## UC-01: Realizar Pedido

**Cubre:** FR-001, FR-002, FR-003, FR-004, FR-005, FR-006, FR-007, FR-008, FR-009, FR-010, FR-011, FR-012, FR-013, FR-014

- **Actores:** Cliente, Sistema de Pago (Stripe)
- **Precondiciones:** El Cliente tiene una sesión activa (o realiza checkout como invitado); el catálogo tiene productos con stock disponible.
- **Flujo Principal:** Cliente busca productos → Sistema muestra catálogo con stock en tiempo real y precio con IVA → Cliente añade productos al carrito e indica aceptación de sustitutos por producto → Cliente elige franja de entrega e introduce dirección → Sistema geocodifica y valida cobertura → Cliente introduce código de descuento (opcional) → Sistema valida descuento y calcula total con IVA desglosado → Cliente paga con tarjeta/Bizum → Sistema confirma pago con Stripe → Sistema crea pedido en estado CONFIRMADO → Sistema envía email y SMS de confirmación.
- **Postcondiciones:** Existe un pedido en estado CONFIRMADO asociado al Cliente, con stock reservado/descontado según diseño, y el Cliente ha recibido confirmación.
- **Flujos Alternativos:**
  - Dirección fuera de cobertura → Sistema rechaza el pedido y muestra mensaje explicando la zona de cobertura.
  - Código de descuento inválido (caducado, ya usado, importe mínimo no alcanzado) → Sistema rechaza el código y permite continuar sin él.
  - Producto sin stock al confirmar → Sistema elimina la línea o solicita confirmación de sustituto antes de proceder al pago.
  - Stripe no responde durante el pago → Sistema conserva el carrito, muestra mensaje claro de reintento, y verifica el estado del cobro antes de permitir un nuevo intento (evita doble cobro).
  - Dos clientes compiten por la última unidad de un producto → El primero en confirmar el pago se queda con la unidad; al segundo se le ofrece el sustituto (si lo aceptó) o se le cancela la línea con mensaje explícito.

---

## UC-02: Modificar o Cancelar Pedido Antes del Cutoff

**Cubre:** FR-015, FR-016

- **Actores:** Cliente, Sistema
- **Precondiciones:** El pedido está en estado CONFIRMADO y son antes de las 22:00 del día anterior a la entrega.
- **Flujo Principal:** Cliente accede a su pedido desde el historial → Cliente modifica líneas de producto, franja o dirección, o cancela el pedido → Sistema valida que la hora actual es anterior al cutoff → Sistema aplica los cambios y recalcula el total → Sistema envía confirmación de la modificación.
- **Postcondiciones:** El pedido refleja los cambios solicitados, o queda cancelado con el reembolso correspondiente iniciado.
- **Flujos Alternativos:**
  - Intento de modificación después del cutoff (22:00) → Sistema rechaza la modificación y muestra mensaje indicando que el pedido ya está en preparación.
  - Cancelación de un pedido ya pagado → Sistema inicia el reembolso vía Stripe.

---

## UC-03: Preparar Pedido (Picking Guiado)

**Cubre:** FR-017, FR-018, FR-019, FR-020, FR-021, FR-022, FR-023, FR-024

- **Actores:** Operario de Almacén, Sistema
- **Precondiciones:** El pedido está CONFIRMADO y ha pasado el cutoff de las 22:00; el turno de almacén está activo (6:00–14:00 o 14:00–22:00).
- **Flujo Principal:** Sistema genera la lista de picking agrupada por zona (Ambiente, Refrigerado, Congelado) en orden óptimo de pasillo → Operario recibe la lista en su tablet → Operario recorre la ruta y escanea el código de barras de cada producto → Sistema valida producto y lote (FEFO, con al menos 3 días de margen) → Sistema descuenta el stock inmediatamente → Operario recoge los productos congelados en último lugar → Sistema marca el pedido como PREPARADO cuando todas las líneas están completas.
- **Postcondiciones:** El pedido está en estado PREPARADO, con stock actualizado en tiempo real y trazabilidad de lote registrada por línea.
- **Flujos Alternativos:**
  - Producto agotado durante el picking → Sistema propone sustituto (misma categoría, precio igual o inferior) si el Cliente lo aceptó para ese producto; si no, cancela la línea y ajusta el importe del pedido.
  - Producto con caducidad ≤ hoy+2 días → Sistema impide su selección y solicita al Operario tomar el siguiente lote válido.
  - Tablet pierde conexión con el servidor durante el picking → Operario continúa trabajando con la lista ya descargada en modo offline; el sistema sincroniza los eventos de escaneo al recuperar la conexión.
  - Escaneo de un código de barras que no corresponde a la línea esperada → Sistema rechaza el escaneo y solicita al Operario reintentar con el producto correcto.

---

## UC-04: Generar y Aprobar Rutas de Reparto

**Cubre:** FR-025, FR-026, FR-027, FR-028, FR-029

- **Actores:** Coordinador de Logística, Sistema, Repartidor
- **Precondiciones:** Existen pedidos en estado PREPARADO para la franja de reparto correspondiente.
- **Flujo Principal:** Sistema agrupa los pedidos PREPARADOS por zona postal → Sistema optimiza el orden de entrega priorizando productos congelados → Sistema genera rutas de 30-40 entregas por repartidor → Coordinador revisa las rutas propuestas y realiza ajustes manuales si es necesario → Coordinador aprueba las rutas → Repartidor recibe la ruta con direcciones, teléfonos, franja horaria, indicador de congelados e instrucciones especiales → Repartidor carga la furgoneta y sale a repartir → Sistema registra la posición del Repartidor en tiempo real.
- **Postcondiciones:** Cada pedido PREPARADO queda asignado a una ruta aprobada y a un repartidor; los pedidos pasan a estado EN_REPARTO al iniciarse la ruta.
- **Flujos Alternativos:**
  - Google Maps no responde durante la generación de rutas → Sistema genera un orden de respaldo por código postal para que el repartidor pueda salir sin demora.
  - Coordinador detecta una ruta desequilibrada (demasiadas entregas o zona mal cubierta) → Coordinador reasigna manualmente pedidos entre repartidores antes de aprobar.

---

## UC-05: Entregar Pedido al Cliente

**Cubre:** FR-030, FR-031, FR-032, FR-033, FR-034, FR-035, FR-036

- **Actores:** Repartidor, Cliente, Sistema
- **Precondiciones:** El pedido está en estado EN_REPARTO y asignado a una ruta activa.
- **Flujo Principal:** Cliente consulta el tracking en tiempo real de su pedido → Sistema envía SMS al Cliente cuando el Repartidor está a ~15 minutos → Repartidor llega a la dirección y marca "en puerta" → Repartidor entrega el pedido y confirma con una fotografía → Sistema marca el pedido como ENTREGADO → Sistema genera la factura electrónica correspondiente.
- **Postcondiciones:** El pedido queda en estado ENTREGADO con prueba fotográfica y factura generada.
- **Flujos Alternativos:**
  - Cliente ausente → Repartidor espera 5 minutos, llama por teléfono, y si no hay respuesta, el sistema marca la entrega como FALLIDA; los productos de frío asociados se dan de baja del inventario reentregable.
  - Repartidor detecta un producto dañado en el momento de la entrega → Repartidor registra la incidencia con foto y el sistema genera una propuesta de reembolso parcial.
  - Transcurridas 2 horas desde la hora estimada sin confirmación de entrega → Sistema alerta al Coordinador de Logística para que investigue.
  - Temperatura de la furgoneta de congelados supera -15°C durante más de 10 minutos → Sistema alerta al Coordinador, que decide acelerar la entrega o cancelar y reembolsar los productos congelados de esa furgoneta.

---

## UC-06: Gestionar Incidencia y Reembolso Post-Entrega

**Cubre:** FR-037, FR-038, FR-039, FR-040, FR-049

- **Actores:** Cliente, Agente de Soporte, Sistema
- **Precondiciones:** El pedido está en estado ENTREGADO.
- **Flujo Principal:** Cliente reporta una incidencia desde la web (producto dañado, equivocado o faltante) adjuntando foto → Sistema evalúa el importe de la línea afectada → Si el importe es < 10€ y hay foto adjunta, el sistema genera un reembolso automático inmediato (máximo 2 al mes por Cliente) → Sistema inicia el reembolso vía Stripe en menos de 48 horas → Sistema envía confirmación al Cliente.
- **Postcondiciones:** La incidencia queda registrada y resuelta (reembolsada o derivada), con el reembolso iniciado dentro del plazo comprometido.
- **Flujos Alternativos:**
  - Importe de la incidencia ≥ 10€ → Sistema deriva el caso a un Agente de Soporte, que revisa la foto del repartidor y aprueba o rechaza el reembolso.
  - Cliente ya alcanzó el límite de 2 reembolsos automáticos ese mes → Sistema deriva el caso a revisión manual aunque el importe sea < 10€ (control antifraude).
  - Cliente usa el chat para una consulta simple ("¿dónde está mi pedido?") → Sistema responde automáticamente sin intervención de un agente.

---

## UC-07: Gestionar Alerta Sanitaria de Lote

**Cubre:** FR-044, FR-045

- **Actores:** Responsable de Calidad y Seguridad Alimentaria, Sistema
- **Precondiciones:** Se recibe una alerta de un organismo regulador (ej. AESAN) sobre un lote específico de un proveedor.
- **Flujo Principal:** Responsable de Calidad introduce el identificador de lote afectado en el sistema → Sistema busca todos los pedidos que contienen ese lote → Sistema obtiene la lista de Clientes afectados con sus datos de contacto → Sistema envía notificación urgente (email + SMS) a los Clientes afectados → Sistema registra el envío y la confirmación de recepción de cada notificación.
- **Postcondiciones:** Todos los Clientes afectados por el lote quedan identificados y notificados, y existe un registro auditable del proceso, completado en menos de 4 horas desde la alerta.
- **Flujos Alternativos:**
  - El lote no aparece en ningún pedido activo (ya fuera de plazo de conservación) → Sistema documenta que no hay clientes afectados y cierra el caso con registro de la consulta realizada.
  - Fallo en el envío de una notificación (email/SMS rebota) → Sistema reintenta por el canal alternativo y escala a notificación manual por el Agente de Soporte si ambos fallan.

---

## UC-08: Administrar Catálogo y Promociones

**Cubre:** FR-047, FR-048

- **Actores:** Administrador, Sistema
- **Precondiciones:** El Administrador tiene sesión autenticada con permisos de gestión de catálogo.
- **Flujo Principal:** Administrador da de alta/modifica/da de baja un producto, asignando su tipo de IVA y ubicación física en almacén → Administrador crea un código de descuento definiendo condiciones (vigencia, uso único/múltiple, importe mínimo) → Sistema valida la configuración y la deja disponible para su aplicación en el checkout.
- **Postcondiciones:** El catálogo y las promociones quedan actualizados y disponibles para los Clientes conforme a las reglas configuradas.
- **Flujos Alternativos:**
  - Administrador intenta dar de baja un producto con stock reservado en pedidos CONFIRMADOS pendientes de picking → Sistema impide la baja inmediata y la programa para cuando no queden pedidos pendientes con ese producto.
  - Administrador configura un código de descuento con fechas contradictorias (fin anterior a inicio) → Sistema rechaza la configuración con mensaje de validación.
