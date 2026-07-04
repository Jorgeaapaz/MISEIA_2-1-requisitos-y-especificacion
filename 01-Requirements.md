# Documento de Requisitos — OMS FreshDirect

## 1. Contexto y Alcance

**Proyecto:** Sistema de Gestión de Pedidos (Order Management System, OMS) para FreshDirect.

**Problema de negocio:** FreshDirect es una empresa de alimentación online (Madrid, España peninsular) que ha crecido de 50 a 800 pedidos/día en 2 años operando con procesos manuales (Google Sheets, WhatsApp, Excel). Esto genera una tasa de error de pedidos del 12% (pérdidas estimadas de 1.400€–3.800€/día), rutas de reparto ineficientes, stock desactualizado y una gestión de incidencias/devoluciones lenta (2-3 días). El sistema actual (PHP + MySQL sin réplica) sufre caídas periódicas sin monitorización proactiva.

**Objetivo del MVP:** Centralizar el ciclo de vida completo de un pedido —desde que el cliente añade productos al carrito hasta que el repartidor confirma la entrega— reduciendo la tasa de error de pedidos al 2%, automatizando picking y rutas, y soportando el pico de Navidad (x3 volumen) sin caídas.

**Timeline:** MVP en 4 meses. Debe estar operativo y estable antes de la temporada alta (noviembre-enero).

**Presupuesto/infraestructura:** Cloud estándar AWS. Sin serverless ni Kubernetes en el MVP (EC2 + RDS), con posibilidad de escalar la arquitectura después.

**Fuera de alcance (explícito, según entrevistas con stakeholders):**
- App móvil nativa para clientes (solo web responsive).
- App móvil nativa para repartidores (usan web móvil en MVP; nativa en fase 2).
- Integración en tiempo real con SAP (solo export CSV nocturno a las 02:00).
- Multi-idioma (solo español en MVP; inglés se añadirá antes de la expansión a Portugal/Francia en 2027).
- Multi-moneda (solo EUR en MVP).
- Programa de fidelización / puntos.
- Suscripciones recurrentes ("mi pedido semanal").
- Chat con IA para soporte (chat en MVP es un canal con respuestas automáticas simples, no conversacional con IA).
- Marketplace para vendedores externos.
- Cobertura fuera de España peninsular (Baleares, Canarias, Ceuta, Melilla quedan fuera).

## 2. Stakeholders

| Rol | Persona | Interés principal |
|---|---|---|
| CEO | Marta Reyes | Reducir pérdidas por errores de pedido (12% → 2%), aguantar el pico de Navidad sin caídas, reducir llamadas de soporte a la mitad |
| Directora de Operaciones | Carmen López | Stock en tiempo real, picking guiado, asignación automática de rutas, tracking de repartidores, prueba de entrega con foto |
| Responsable de Almacén | Javier Ortiz | Descuento de stock en tiempo real durante picking, ruta de picking óptima, lector de código de barras, trazabilidad de lote (FEFO) |
| Coordinador de Logística | Pablo Ruiz | Asignación automática de rutas, información completa para el repartidor, gestión de entregas fallidas e incidencias en ruta |
| Responsable de Atención al Cliente | Lucía Fernández | Reducir incidencias telefónicas mediante tracking y autoservicio, agilizar devoluciones y reembolsos |
| CTO | Roberto Sánchez | Stack moderno (TS/Node/React/PostgreSQL/Redis/AWS), integraciones externas fiables, alta disponibilidad, cumplimiento normativo, arquitectura de roles |
| Responsable de Calidad y Seguridad Alimentaria | Ana Martínez | Trazabilidad de lote end-to-end, alertas sanitarias en <4h, control de cadena de frío, control de caducidad |
| Director Financiero | Miguel Torres | Facturación electrónica automática con IVA correcto, reembolsos ágiles, reporting diario a SAP, gestión de descuentos/promociones |

## 3. Requisitos Funcionales (FR)

Formato: Actor + SHALL + Acción + Resultado observable. Prioridad: CRÍTICA / IMPORTANTE / NICE-TO-HAVE.

### 3.1 Catálogo, Búsqueda y Carrito

| ID | Requisito | Prioridad |
|---|---|---|
| FR-001 | El Cliente SHALL poder buscar productos por categoría, nombre o código de barras y SHALL recibir resultados que reflejen el stock disponible en tiempo real. | CRÍTICA |
| FR-002 | El sistema SHALL mostrar el precio de cada producto con el IVA correspondiente (4%, 10% o 21% según categoría fiscal) ya incluido. | CRÍTICA |
| FR-003 | El Cliente SHALL poder añadir y quitar productos de un carrito de compra antes de confirmar el pedido. | CRÍTICA |
| FR-004 | El Cliente SHALL poder indicar, por cada producto del pedido, si acepta un sustituto en caso de agotamiento. | CRÍTICA |
| FR-005 | El sistema SHALL validar la disponibilidad de stock en tiempo real al confirmar el pedido y SHALL impedir la confirmación de líneas sin stock. | CRÍTICA |

### 3.2 Entrega, Cobertura y Descuentos

| ID | Requisito | Prioridad |
|---|---|---|
| FR-006 | El Cliente SHALL poder elegir una franja de entrega (mañana 8:00–14:00 o tarde 16:00–20:00) e introducir la dirección de entrega con instrucciones especiales. | CRÍTICA |
| FR-007 | El sistema SHALL geocodificar la dirección introducida y SHALL rechazar el pedido si la dirección está fuera de la cobertura de España peninsular. | CRÍTICA |
| FR-008 | El sistema SHALL calcular el coste de envío, incluyendo el sobrecoste del servicio Express (3h, solo Madrid capital, 5,90€) cuando el Cliente lo seleccione. | IMPORTANTE |
| FR-009 | El Cliente SHALL poder introducir opcionalmente un código de descuento antes del pago. | IMPORTANTE |
| FR-010 | El sistema SHALL validar el código de descuento (vigencia, uso único no consumido, importe mínimo del pedido) y SHALL rechazar códigos inválidos con un mensaje explícito. | IMPORTANTE |
| FR-011 | El sistema SHALL calcular el importe total del pedido desglosando la base imponible y la cuota de IVA por cada tipo aplicable (4%, 10%, 21%). | CRÍTICA |
| FR-050 | El sistema SHALL dividir en dos envíos los pedidos que superen 25 kg de peso total, asignando ambos envíos al mismo repartidor y a la misma franja horaria. | IMPORTANTE |

### 3.3 Pago y Confirmación de Pedido

| ID | Requisito | Prioridad |
|---|---|---|
| FR-012 | El Cliente SHALL poder pagar el pedido con tarjeta o Bizum a través de la pasarela Stripe. | CRÍTICA |
| FR-013 | El sistema SHALL confirmar el pedido (estado CONFIRMADO) únicamente si el pago se ha completado con éxito, sin generar cobros duplicados en reintentos. | CRÍTICA |
| FR-014 | El sistema SHALL enviar una confirmación por email y SMS al Cliente inmediatamente después de confirmar el pedido. | CRÍTICA |
| FR-015 | El Cliente SHALL poder modificar o cancelar su pedido hasta las 22:00 del día anterior a la entrega (cutoff operativo). | CRÍTICA |
| FR-016 | El sistema SHALL congelar los pedidos programados para el día siguiente a partir de las 22:00 e SHALL impedir cualquier modificación posterior. | CRÍTICA |

### 3.4 Picking y Gestión de Almacén

| ID | Requisito | Prioridad |
|---|---|---|
| FR-017 | El sistema SHALL generar, para cada pedido confirmado tras el cutoff, una lista de picking agrupada por zona de almacén (Ambiente, Refrigerado, Congelado) ordenada según la secuencia óptima de pasillo. | CRÍTICA |
| FR-018 | El Operario de Almacén SHALL escanear el código de barras de cada producto durante el picking, y el sistema SHALL confirmar que el producto y el lote escaneados corresponden a la línea de pedido. | CRÍTICA |
| FR-019 | El sistema SHALL descontar el stock del producto escaneado en el momento del escaneo, no al final del turno. | CRÍTICA |
| FR-020 | El sistema SHALL indicar al Operario de Almacén el lote a recoger siguiendo la política FEFO (First Expired, First Out), siempre que ese lote conserve al menos 3 días de margen antes de caducar. | CRÍTICA |
| FR-021 | El sistema SHALL impedir el picking de un producto cuya fecha de caducidad sea igual o inferior a hoy + 2 días. | CRÍTICA |
| FR-022 | Cuando un producto esté agotado durante el picking, el sistema SHALL proponer automáticamente un sustituto de igual categoría y precio igual o inferior si el Cliente lo aceptó para ese producto, o SHALL cancelar la línea del pedido en caso contrario. | CRÍTICA |
| FR-023 | El sistema SHALL marcar el pedido como PREPARADO cuando todas sus líneas hayan completado el picking, garantizando que los productos congelados se recogen en último lugar. | CRÍTICA |
| FR-024 | El sistema SHALL generar diariamente un informe de productos cuya caducidad vence en las siguientes 48 horas. | IMPORTANTE |

### 3.5 Rutas y Logística

| ID | Requisito | Prioridad |
|---|---|---|
| FR-025 | El sistema SHALL asignar automáticamente los pedidos PREPARADOS a rutas de reparto, agrupándolos por zona postal, optimizando el orden de entrega y priorizando la entrega temprana de productos congelados. | CRÍTICA |
| FR-026 | El sistema SHALL limitar cada ruta generada a un máximo de 30-40 entregas por repartidor. | IMPORTANTE |
| FR-027 | El Coordinador de Logística SHALL poder revisar y ajustar manualmente las rutas propuestas antes de aprobarlas para su ejecución. | CRÍTICA |
| FR-028 | El Repartidor SHALL recibir en su dispositivo la ruta asignada con dirección completa, teléfono del cliente, franja horaria, indicador de productos congelados e instrucciones especiales de entrega. | CRÍTICA |
| FR-029 | El sistema SHALL registrar y actualizar la posición del Repartidor en tiempo real mientras ejecuta su ruta. | IMPORTANTE |

### 3.6 Tracking y Entrega

| ID | Requisito | Prioridad |
|---|---|---|
| FR-030 | El Cliente SHALL poder consultar en tiempo real el estado y la ubicación estimada de su pedido en tránsito. | CRÍTICA |
| FR-031 | El sistema SHALL enviar un SMS al Cliente cuando el Repartidor esté a aproximadamente 15 minutos de la entrega. | IMPORTANTE |
| FR-032 | El Repartidor SHALL confirmar cada entrega adjuntando una fotografía como prueba, sin depender de confirmación manual por WhatsApp. | CRÍTICA |
| FR-033 | Si el Cliente no está presente en la entrega, el Repartidor SHALL esperar 5 minutos, SHALL intentar contactar telefónicamente, y si no hay respuesta, el sistema SHALL marcar la entrega como fallida. | CRÍTICA |
| FR-034 | El sistema SHALL marcar el pedido como ENTREGADO al confirmarse la entrega y SHALL generar automáticamente la factura electrónica asociada. | CRÍTICA |
| FR-035 | Si no se recibe confirmación de entrega transcurridas 2 horas desde la hora estimada, el sistema SHALL generar una alerta al Coordinador de Logística. | IMPORTANTE |
| FR-036 | El Repartidor SHALL poder registrar en el momento de la entrega una incidencia de producto dañado con fotografía, generando una propuesta de reembolso parcial. | IMPORTANTE |

### 3.7 Incidencias, Devoluciones y Reembolsos

| ID | Requisito | Prioridad |
|---|---|---|
| FR-037 | El Cliente SHALL poder reportar desde la web una incidencia post-entrega (producto dañado, equivocado o faltante) adjuntando una fotografía. | CRÍTICA |
| FR-038 | El sistema SHALL generar un reembolso automático inmediato cuando el importe de la incidencia sea inferior a 10€ y la reclamación incluya fotografía, respetando un máximo de 2 reembolsos automáticos por Cliente al mes. | CRÍTICA |
| FR-039 | El sistema SHALL derivar a revisión de un Agente de Soporte toda incidencia de importe igual o superior a 10€. | CRÍTICA |
| FR-040 | El sistema SHALL iniciar el reembolso a través de Stripe en menos de 48 horas desde su aprobación. | CRÍTICA |
| FR-049 | El Cliente SHALL poder usar el canal de chat de la web para incidencias simples, recibiendo respuestas automáticas cuando el caso lo permita. | NICE-TO-HAVE |

### 3.8 Historial, Facturación y Reporting

| ID | Requisito | Prioridad |
|---|---|---|
| FR-041 | El Cliente SHALL poder consultar el historial completo de sus pedidos, con detalle de producto, precio pagado, estado e incidencias asociadas a cada uno. | IMPORTANTE |
| FR-042 | El sistema SHALL generar automáticamente una factura electrónica por cada pedido entregado, desglosando el IVA por tipo aplicable e incluyendo el NIF del Cliente cuando se haya proporcionado. | CRÍTICA |
| FR-043 | El sistema SHALL exportar cada noche a las 02:00 un fichero CSV con los pedidos y facturas del día para su importación en SAP. | CRÍTICA |
| FR-048 | El Administrador SHALL poder crear y gestionar códigos de descuento y promociones, definiendo sus condiciones de aplicabilidad. | IMPORTANTE |

### 3.9 Trazabilidad y Seguridad Alimentaria

| ID | Requisito | Prioridad |
|---|---|---|
| FR-044 | El sistema SHALL permitir identificar, en menos de 4 horas, todos los pedidos y Clientes afectados por un lote específico de producto ante una alerta sanitaria. | CRÍTICA |
| FR-045 | El sistema SHALL enviar una notificación urgente (email + SMS) a los Clientes afectados por una alerta sanitaria y SHALL registrar el envío y la recepción de dicha notificación. | CRÍTICA |
| FR-046 | El sistema SHALL recibir y registrar la temperatura de cada furgoneta cada 5 minutos y SHALL generar una alerta al Coordinador de Logística si la temperatura de la zona de congelados supera -15°C. | CRÍTICA |

### 3.10 Administración

| ID | Requisito | Prioridad |
|---|---|---|
| FR-047 | El Administrador SHALL poder dar de alta, modificar y dar de baja productos del catálogo, incluyendo su tipo de IVA y su ubicación física en almacén. | IMPORTANTE |

## 4. Requisitos No-Funcionales (NFR)

| ID | Requisito | SLO | Medición |
|---|---|---|---|
| NFR-001 | El sistema SHALL soportar el volumen de pedidos de temporada normal sin degradación perceptible. | 1.000 pedidos/día sostenidos, p95 de tiempo de respuesta de API < 500ms | Load test con k6 sobre entornos de staging equivalentes a producción |
| NFR-002 | El sistema SHALL soportar el pico de tráfico de Navidad sin caídas. | 3.000 pedidos/día (x3 sobre temporada normal), 0 incidentes de caída entre el 1 nov y el 31 ene | Load test de pico sostenido + simulacro de Black Friday/Navidad previo a temporada alta |
| NFR-003 | El sistema SHALL mantener una disponibilidad elevada en temporada normal. | ≤ 1 hora de downtime/mes | Monitorización de uptime (ej. Pingdom/CloudWatch Synthetics), reporte mensual de SLA |
| NFR-004 | El sistema SHALL mantener disponibilidad total en temporada alta. | 0 minutos de downtime no planificado entre nov-ene | Monitorización de uptime con alerta en tiempo real, postmortem obligatorio ante cualquier incidente |
| NFR-005 | El stock mostrado al Cliente SHALL reflejar el descuento realizado durante el picking. | Latencia de propagación de descuento de stock < 5 segundos desde el escaneo | Test de integración midiendo tiempo entre evento de escaneo y actualización visible en catálogo |
| NFR-006 | El sistema SHALL detectar proactivamente condiciones de degradación antes de una caída total. | Alertas disparadas con al menos 5 minutos de antelación a un fallo total conocido (saturación de CPU/memoria/conexiones DB) | Dashboards de monitorización con umbrales de alerta (ver [[07-Operative-specs]]) |
| NFR-007 | El sistema SHALL proteger los datos de pago sin almacenar información sensible de tarjeta. | 0 datos de tarjeta (PAN completo, CVV) almacenados o logueados en los sistemas propios | Auditoría de logs y esquema de base de datos; escaneo automatizado de logs en CI |
| NFR-008 | La aplicación de picking en tablet SHALL seguir operativa durante una pérdida de conexión con el servidor. | Continuidad de la lista de picking ya descargada durante toda la pérdida de conexión, sincronización íntegra al recuperar conexión en < 30s | Test de desconexión simulada en entorno de staging |
| NFR-009 | El sistema SHALL seguir permitiendo la salida a reparto si el proveedor de geolocalización no responde. | Repartidores pueden iniciar ruta con orden por código postal en < 2 minutos de indisponibilidad detectada de Google Maps | Test de fallo simulado (mock de la API caída) |
| NFR-010 | El tiempo de picking por pedido SHALL reducirse respecto al proceso manual actual. | Tiempo medio de picking por pedido < 5 minutos (frente a 8-12 min actuales) | Medición de timestamps de inicio/fin de picking por pedido en producción |
| NFR-011 | El sistema SHALL garantizar que un pedido no se cobra más de una vez ante reintentos de pago. | 0 cobros duplicados detectados por reintento de pago | Test de idempotencia sobre el flujo de pago (ver [[08-Contracts]]) e idempotency keys de Stripe |
| NFR-012 | El sistema SHALL resolver de forma consistente la concurrencia de compra sobre la última unidad de un producto. | 0 casos de sobreventa (stock negativo) detectados en producción | Test de concurrencia (carga simultánea sobre el mismo SKU) en staging |
| NFR-013 | El sistema SHALL permitir resolver incidencias más rápido que el proceso manual actual. | Tiempo medio de resolución de incidencia < 24h; incidencias < 10€ resueltas en < 1h | Medición de timestamps de apertura/cierre de incidencia |

**Conflicto potencial señalado:** NFR-005 (propagación de stock en <5s) y NFR-001/NFR-002 (rendimiento bajo carga de pico) pueden entrar en tensión si la actualización de stock se implementa de forma síncrona y bloqueante en cada escritura — ver decisión de arquitectura de caché/colas en [[../08-Bloque2/ADR/003-ADR-cache-y-consistencia-de-stock]] (a definir en ADRs). También existe tensión entre NFR-007 (minimizar datos de pago en logs) y OPS de observabilidad (que normalmente querría más contexto en logs de errores de pago): se resuelve enmascarando campos sensibles antes de loguear (ver [[07-Operative-specs]]).

## 5. Requisitos Regulatorios (REG)

| ID | Requisito | Normativa |
|---|---|---|
| REG-001 | El sistema SHALL permitir rastrear cualquier producto desde el proveedor hasta el Cliente final, identificando en menos de 4 horas los pedidos afectados por un lote concreto. | Reglamento (CE) 178/2002, principio de trazabilidad "de la granja a la mesa" |
| REG-002 | El sistema SHALL tratar los datos personales del Cliente (nombre, dirección, teléfono, email) conforme a las bases legales, derechos de acceso/rectificación/supresión y minimización de datos. | GDPR (Reglamento UE 2016/679) |
| REG-003 | El sistema SHALL delegar el procesamiento y almacenamiento de datos de pago en un proveedor certificado (Stripe) y SHALL evitar almacenar o loguear datos sensibles de tarjeta. | PCI DSS |
| REG-004 | El sistema SHALL cumplir con los requisitos de comunicaciones comerciales, gestión de cookies y aviso legal en el sitio web. | LSSI-CE (Ley 34/2002 de Servicios de la Sociedad de la Información) |
| REG-005 | El sistema SHALL generar una factura electrónica válida por cada pedido, conforme a la normativa de facturación electrónica española aplicable a empresas con facturación superior a 8M€ desde 2026. | Normativa española de facturación electrónica (2026) |
| REG-006 | El sistema SHALL impedir el envío de productos con caducidad igual o inferior a hoy + 2 días y SHALL aplicar la política FEFO en el picking de productos con lote. | Normativa española de seguridad alimentaria / política interna derivada de REG-001 |
| REG-007 | El sistema SHALL aplicar y desglosar en factura el tipo de IVA correspondiente a cada producto (4% superreducido, 10% reducido, 21% general). | Normativa fiscal española del IVA |

## 6. Requisitos Operativos (OPS)

| ID | Requisito |
|---|---|
| OPS-001 | El sistema SHALL contar con un pipeline de CI/CD que ejecute lint, tests y build antes de desplegar a staging, y que requiera aprobación manual antes de desplegar a producción. |
| OPS-002 | El sistema SHALL exponer métricas de infraestructura y negocio (latencia, tasa de error, pedidos/hora, fallos de picking) con alertas configuradas sobre umbrales críticos. |
| OPS-003 | El sistema SHALL contar con una estrategia de backup de base de datos (incremental diario + completo semanal) y un procedimiento de recuperación ante desastres con RTO/RPO definidos. |
| OPS-004 | El sistema SHALL ejecutar automáticamente cada noche a las 02:00 el export de pedidos/facturas a CSV para SAP, con reintento automático si falla. |
| OPS-005 | El equipo de operaciones SHALL disponer de runbooks documentados para los incidentes críticos identificados (caída de la web en temporada alta, rotura de cadena de frío, entregas sin confirmar). |
| OPS-006 | El sistema SHALL sincronizar automáticamente los datos de picking capturados offline en la tablet del almacén en cuanto se recupere la conexión, sin pérdida de eventos. |

## 7. Glosario

| Término | Definición |
|---|---|
| OMS | Order Management System: sistema que gestiona el ciclo de vida completo de un pedido, desde el carrito hasta la entrega. |
| Cutoff | Hora límite (22:00 del día anterior a la entrega) a partir de la cual un pedido no admite modificaciones y entra en preparación. |
| Picking | Proceso de recogida física de los productos de un pedido en las estanterías del almacén. |
| FEFO | First Expired, First Out: política de rotación de stock que prioriza el envío del lote con fecha de caducidad más próxima. |
| SKU | Referencia única de producto en el catálogo. |
| Lote | Conjunto de unidades de un producto producidas/recibidas en un mismo evento, identificado con código de lote y fecha de caducidad. |
| Zona Ambiente | Zona de almacén (A1-A12) para productos sin restricción de temperatura (pasta, conservas, bebidas). |
| Zona Refrigerado | Zona de almacén (B1-B6) para productos que no pueden estar más de 2h fuera de cámara (lácteos, carne, verdura fresca). |
| Zona Congelado | Zona de almacén (C1-C4) para productos que no pueden estar más de 45 min fuera de congelador (helados, pescado congelado). |
| Sustituto | Producto alternativo de la misma categoría y precio igual o inferior que el sistema propone cuando el original está agotado, solo si el Cliente lo autorizó para esa línea. |
| Alerta sanitaria | Comunicación de un organismo regulador (ej. AESAN) sobre un lote de producto potencialmente inseguro, que obliga a identificar y notificar a los clientes afectados en menos de 4 horas. |
| Cadena de frío | Mantenimiento continuo de la temperatura adecuada de un producto refrigerado o congelado desde el almacén hasta la entrega. |
| Entrega Express | Servicio de entrega en 3 horas, disponible solo en Madrid capital, con sobrecoste de 5,90€. |
| Estado del pedido | Uno de: CARRITO, CONFIRMADO, PREPARADO, EN_REPARTO, ENTREGADO, FALLIDO, CANCELADO (ver máquina de estados en [[04-Behavioral-specs]]). |
| Reembolso automático | Reembolso procesado sin intervención humana para incidencias < 10€ con foto adjunta, limitado a 2 por Cliente/mes. |
| AESAN | Agencia Española de Seguridad Alimentaria y Nutrición, organismo emisor de alertas sanitarias. |
