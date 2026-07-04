## La Empresa

**FreshDirect** es una empresa de alimentación online con sede en Madrid que opera en toda España peninsular. Llevan 2 años vendiendo a través de una web básica con procesamiento manual de pedidos. Han crecido de 50 pedidos/día a 800 pedidos/día y el sistema actual no aguanta más: los errores de stock son constantes, los repartidores reciben rutas incorrectas y las devoluciones se gestionan por email.

Necesitan un OMS que centralice todo el ciclo de vida de un pedido: desde que el cliente añade productos al carrito hasta que el repartidor confirma la entrega en la puerta.

**Presupuesto:** Cloud estándar (AWS). No serverless, no Kubernetes al principio — empezar con EC2 + RDS y escalar después si hace falta.

**Timeline:** MVP en 4 meses. La temporada fuerte es Navidad (noviembre-enero), donde el volumen se multiplica por 3.

---

## Entrevistas con Stakeholders

### Entrevista 1: CEO — Marta Reyes

> **¿Cuál es el problema principal?**
>
> Perdemos dinero. El 12% de los pedidos tienen algún error: enviamos un producto equivocado, o el cliente pide algo que aparece en stock pero no lo tenemos, o el repartidor va a una dirección y el cliente no está. Cada error nos cuesta entre 15€ y 40€ entre reenvío, devolución y compensación. Con 800 pedidos al día, eso son entre 1.400€ y 3.800€ de pérdidas diarias por errores evitables.
>
> **¿Qué es éxito para ti?**
>
> Que el error de pedidos baje del 12% al 2%. Y que cuando haya un error, el cliente pueda resolverlo solo desde la web sin llamar por teléfono. Ahora tenemos 4 personas solo atendiendo llamadas de "¿dónde está mi pedido?" — quiero que eso se reduzca a la mitad.
>
> **¿Cuánto facturáis?**
>
> El ticket medio es de 65€. Con 800 pedidos/día, unos 52.000€/día. En Navidad sube a 2.400 pedidos/día. Necesitamos que el sistema aguante ese pico sin caerse. El año pasado la web se cayó el 23 de diciembre durante 4 horas y perdimos unas 400 pedidos — unos 26.000€.
>
> **¿Hay planes de expansión?**
>
> Sí, en 2027 queremos abrir Portugal y Francia. Pero eso está fuera del MVP. Lo que sí necesitamos desde el principio es que el sistema soporte multi-moneda (EUR) y multi-idioma (español e inglés como mínimo) para no tener que rehacerlo todo después.

---

### Entrevista 2: Directora de Operaciones — Carmen López

> **¿Cómo funciona un pedido hoy?**
>
> 1. El cliente pide por la web. Le cobramos inmediatamente.
> 2. El pedido cae en un Google Sheet que revisa el equipo de almacén.
> 3. Un operario de almacén imprime el pedido, va estantería por estantería recogiendo los productos (picking), los mete en una bolsa isotérmica y lo deja en la zona de carga.
> 4. El coordinador de logística asigna pedidos a repartidores según zona. Lo hace mirando un mapa y un Excel con las direcciones. Tarda entre 30 y 45 minutos cada mañana.
> 5. El repartidor carga los pedidos en la furgoneta y sale. Tiene las direcciones en un papel o en Google Maps que se pone él manualmente.
> 6. Cuando entrega, nos manda un WhatsApp diciendo "entregado". Alguien del equipo lo marca a mano en el Sheet.
>
> **¿Qué falla?**
>
> Todo. El Sheet se descuadra, el stock no se actualiza en tiempo real (se actualiza a mano cada noche), las rutas son ineficientes, los repartidores a veces se pierden o van a una dirección antigua del cliente, y cuando algo sale mal no hay registro de qué pasó.
>
> **¿Qué necesitas del OMS?**
>
> - **Stock en tiempo real**: que cuando un producto se acaba en el almacén, desaparezca de la web inmediatamente. No dentro de 12 horas, inmediatamente.
> - **Picking guiado**: que el sistema le diga al operario exactamente qué productos recoger y en qué orden de estantería, no que vaya con un papel buscando.
> - **Asignación de rutas automática**: que el sistema agrupe los pedidos por zona y genere rutas óptimas para cada repartidor. Ahora lo hacemos a mano y es un desastre.
> - **Tracking del repartidor**: quiero saber dónde está cada repartidor en tiempo real y que el cliente también pueda ver dónde está su pedido.
> - **Confirmación de entrega con foto**: el repartidor saca una foto del pedido en la puerta del cliente como prueba de entrega. No más WhatsApps.
>
> **¿Cuántos operarios y repartidores tenéis?**
>
> 15 operarios de almacén en dos turnos (mañana 6:00–14:00, tarde 14:00–22:00). 20 repartidores con furgonetas propias. En Navidad contratamos 10 repartidores extra temporales.
>
> **¿Cuál es el tiempo de entrega prometido?**
>
> Pedidos antes de las 22:00 se entregan al día siguiente entre 8:00 y 14:00, o entre 16:00 y 20:00 (el cliente elige franja). Pedidos de fin de semana se entregan el lunes. Tenemos un servicio "Express" que entrega en 3 horas, pero solo en Madrid capital y con un sobrecoste de 5,90€.
>
> **¿Hay productos especiales?**
>
> Sí. Tenemos tres categorías de temperatura:
> - **Ambiente** (pasta, conservas, bebidas): sin restricción de tiempo.
> - **Refrigerado** (lácteos, carne, verdura fresca): la cadena de frío no puede romperse más de 2 horas fuera de la cámara.
> - **Congelado** (helados, pescado congelado): máximo 45 minutos fuera del congelador hasta que llega al cliente.
>
> Esto afecta al picking (los congelados se recogen los últimos) y a las rutas (las furgonetas con congelados van primero a los clientes más lejanos... es un lío que ahora resolvemos con sentido común y experiencia de los repartidores).

---

### Entrevista 3: Responsable de Almacén — Javier Ortiz

> **¿Cómo está organizado el almacén?**
>
> Tenemos 3 zonas:
> - **Zona A**: Ambiente (12 pasillos, estanterías A1-A12, cada estantería tiene 5 niveles)
> - **Zona B**: Refrigerado (cámara frigorífica, 6 pasillos B1-B6)
> - **Zona C**: Congelado (cámara de congelación, 4 pasillos C1-C4)
>
> Cada producto tiene una ubicación fija tipo "A3-N2-P15" (pasillo A3, nivel 2, posición 15).
>
> **¿Cómo se actualiza el stock ahora?**
>
> Los proveedores entregan mercancía entre las 4:00 y las 6:00 de la mañana. Un operario cuenta lo que llega y lo apunta en el Sheet. Pero durante el día, cuando los operarios hacen picking, no descuentan el stock en tiempo real. Lo descuentan al final del turno. Eso significa que a las 10:00 de la mañana, el stock que ves en la web es el de las 6:00 — y puede que ya se hayan vendido la mitad de las unidades de un producto.
>
> **¿Cuántos productos gestionáis?**
>
> Unas 3.500 referencias activas. Cada semana entran unas 20-30 nuevas y se retiran unas 15-20 (productos de temporada, rotación).
>
> **¿Qué pasa cuando un producto se agota durante el picking?**
>
> Ahora mismo, el operario llama al coordinador, que llama al equipo de atención al cliente, que llama al cliente para ofrecerle un sustituto o cancelar esa línea del pedido. Tarda entre 10 y 20 minutos resolver cada caso. Con 50-70 agotamientos diarios, nos comemos horas enteras en esto.
>
> **¿Qué necesitarías del sistema?**
>
> - Que el stock se descuente EN EL MOMENTO del picking, no al final del turno
> - Que si un producto se agota, el sistema proponga automáticamente un sustituto (el cliente aprobó sustitutos al hacer el pedido, sí o no, y para qué productos)
> - Que la ruta de picking sea óptima: que no me mande al pasillo A12, luego al A1, luego al A11. Que vaya en orden.
> - Lector de código de barras integrado: el operario escanea cada producto y el sistema confirma que es el correcto. Ahora confían en la vista y a veces meten un yogur de fresa en vez de uno natural.
>
> **¿Hay productos con lote y caducidad?**
>
> Sí, todos los refrigerados y congelados tienen lote y fecha de caducidad. Siempre tenemos que enviar el lote con caducidad más próxima (FEFO: First Expired, First Out). El año pasado tuvimos una retirada sanitaria de un lote de pollo y tardamos 3 días en identificar a qué clientes les habíamos enviado ese lote. Casi nos cierra Sanidad. Necesito trazabilidad de lote: saber exactamente qué lote fue a qué cliente y en qué pedido.

---

### Entrevista 4: Coordinador de Logística — Pablo Ruiz

> **¿Cuántas entregas hacéis al día?**
>
> 800 pedidos al día, pero se convierten en unas 700-750 entregas (algunos clientes hacen 2 pedidos el mismo día y los consolidamos en una entrega). Las entregas se reparten en 2 franjas: mañana (8:00–14:00) y tarde (16:00–20:00). El 60% van por la mañana.
>
> **¿Cómo se asignan las rutas?**
>
> Yo, manualmente. Miro las direcciones, las agrupe por código postal, asigno cada grupo a un repartidor. Intento que cada repartidor tenga 30-40 entregas. Pero no optimizo la ruta — el repartidor decide su propio orden. Algunos son buenos y entregan todo, otros se lían y les quedan 5-6 pedidos sin entregar que hay que reprogramar para el día siguiente.
>
> **¿Qué información necesita el repartidor?**
>
> - Dirección completa con piso, puerta y código del portero (si aplica)
> - Nombre del cliente y teléfono (por si no contesta al portero)
> - Franja horaria que eligió el cliente
> - Si hay productos congelados (para saber que tiene que ir rápido)
> - Instrucciones especiales ("dejar en recepción", "llamar 5 min antes", "portero automático código 4523")
>
> **¿Qué pasa cuando el cliente no está?**
>
> Lo intentamos una vez. Si no está, el repartidor espera 5 minutos, llama al cliente, y si no contesta, se lleva el pedido de vuelta. Si hay productos de frío, se pierden (no se puede revender por cadena de frío rota). Eso nos cuesta unos 800€/semana en producto perdido por "cliente ausente".
>
> **¿Cuántas entregas fallan?**
>
> Un 8% de las entregas falla en el primer intento. De esas, la mitad es "cliente no estaba" y la otra mitad es "dirección incorrecta" o "no encontró el portal". Con buenas direcciones y tracking para el cliente ("tu pedido llega en 15 minutos"), podríamos reducir eso al 3%.
>
> **¿El repartidor necesita cobrar algo?**
>
> No. Todo se paga online por adelantado. Pero sí necesita gestionar devoluciones en el momento: si el cliente ve que un producto llegó dañado (aplastado, descongelado), el repartidor puede registrar la incidencia con foto y el sistema genera un reembolso parcial automáticamente.

---

### Entrevista 5: Responsable de Atención al Cliente — Lucía Fernández

> **¿Cuántas incidencias recibís al día?**
>
> Entre 80 y 120. El 40% son "¿dónde está mi pedido?" (tracking), el 25% son "me llegó un producto equivocado", el 20% son "quiero cancelar o modificar mi pedido", y el 15% son reclamaciones de calidad (producto caducado, aplastado, descongelado).
>
> **¿Cuáles podrían resolverse sin intervención humana?**
>
> El tracking, sin duda — si el cliente ve en la web dónde está su pedido, no llama. Eso son 40-50 llamadas diarias menos.
>
> Las modificaciones de pedido son más complicadas. Ahora el cliente puede cambiar el pedido hasta las 22:00 del día anterior a la entrega (ese es el "cutoff" operativo — después de esa hora, el almacén ya empezó el picking). Si pudieran hacerlo solos desde la web en vez de llamar, ganaríamos mucho tiempo.
>
> **¿Qué pasa con las devoluciones?**
>
> El cliente llama, un agente abre un ticket en otro Google Sheet (sí, otro Sheet), valida con el almacén si el producto estaba en buen estado al salir, y decide si reembolsar o reenviar. El proceso tarda 2-3 días. El cliente quiere que se resuelva inmediatamente.
>
> Para productos de menos de 10€, deberíamos reembolsar automáticamente sin preguntas. Para los de más de 10€, revisión rápida (la foto del repartidor debería ser suficiente).
>
> **¿Qué canales de contacto tenéis?**
>
> Teléfono (60%), email (30%), y un chat en la web que casi nadie usa porque tarda mucho en responder (10%). Queremos que el chat sea el canal principal, con respuestas automáticas para casos simples.
>
> **¿El cliente puede ver el historial de sus pedidos?**
>
> Sí, pero solo los últimos 3 meses y sin detalle. Quieren ver todo el historial, con detalle de cada producto, precio pagado, estado, y cualquier incidencia asociada.

---

### Entrevista 6: CTO — Roberto Sánchez

> **¿Qué stack tenéis ahora?**
>
> Una web en PHP con MySQL. El PHP tiene 6 años y nadie quiere tocarlo. MySQL tiene una sola instancia sin réplica. Se cae una o dos veces al mes durante unos 15-20 minutos. No tenemos monitorización real — nos enteramos de las caídas porque los clientes llaman.
>
> **¿Qué stack queréis para el OMS?**
>
> TypeScript con Node.js en backend, React en frontend. PostgreSQL como base de datos principal. Redis para caché y colas. Todo en AWS. Queremos API REST bien documentada para que en el futuro podamos hacer una app móvil para repartidores sin rehacer el backend.
>
> **¿Qué integraciones externas necesitáis?**
>
> - **Pasarela de pago**: Stripe. Ya la usamos en la web actual. El cliente paga con tarjeta o con Bizum (a través de Stripe).
> - **Geolocalización y rutas**: Google Maps API para geocodificar direcciones, calcular rutas y tracking de repartidores.
> - **Email transaccional**: SendGrid para confirmaciones, tracking, facturas.
> - **SMS**: Twilio para notificar al cliente "tu pedido llega en 15 minutos" y para el código de verificación del repartidor.
> - **ERP / Contabilidad**: Exportación diaria de pedidos y facturas a un fichero CSV que importa el departamento de contabilidad en SAP. No necesitamos integración en tiempo real con SAP, pero sí un export automático cada noche a las 02:00.
>
> **¿Cuánto puede caerse el sistema?**
>
> En temporada alta (noviembre-enero), nada. Zero tolerance. El 23 de diciembre no se puede caer.
> En temporada normal, podemos tolerar 1 hora de downtime al mes como máximo.
> Quiero que el sistema me avise ANTES de que se caiga: monitorización proactiva.
>
> **¿Datos sensibles?**
>
> Sí:
> - Datos personales de clientes (nombre, dirección, teléfono, email) — GDPR
> - Datos de pago — PCI DSS (delegado a Stripe, pero ojo con los logs)
> - Datos de salud alimentaria — trazabilidad de lotes, alertas sanitarias. Regulación española de seguridad alimentaria: debemos poder identificar en menos de 4 horas todos los clientes que recibieron un lote específico de cualquier producto.
>
> **¿Quién va a usar el sistema?**
>
> 6 roles:
> 1. **Cliente** (web): buscar productos, hacer pedidos, ver tracking, gestionar incidencias
> 2. **Operario de almacén** (tablet): picking guiado, escaneo de códigos de barras, confirmación de productos
> 3. **Coordinador de logística** (desktop): asignar rutas, ver estado de repartidores, gestionar entregas fallidas
> 4. **Repartidor** (móvil): ver ruta, navegar, confirmar entrega con foto, registrar incidencias
> 5. **Agente de soporte** (desktop): gestionar incidencias, ver historial del cliente, procesar devoluciones
> 6. **Administrador** (desktop): gestionar catálogo, ver métricas, configurar precios, gestionar promociones

---

### Entrevista 7: Responsable de Calidad y Seguridad Alimentaria — Ana Martínez

> **¿Qué normativa aplica?**
>
> - **Reglamento (CE) 178/2002**: Trazabilidad alimentaria. Debemos poder rastrear cualquier producto desde el proveedor hasta el cliente final ("de la granja a la mesa"). Si hay una alerta sanitaria, tenemos que identificar en menos de 4 horas qué clientes recibieron el lote afectado y notificarles.
> - **GDPR**: Datos personales de clientes.
> - **PCI DSS**: Datos de pago (delegado a Stripe).
> - **LSSI-CE** (Ley de Servicios de la Sociedad de la Información): Comunicaciones comerciales, cookies, aviso legal.
> - **Normativa de facturación electrónica española**: Desde 2026 obligatoria para empresas con facturación > 8M€. Cada pedido debe generar una factura electrónica válida.
>
> **¿Cómo funciona una alerta sanitaria?**
>
> Ejemplo real del año pasado: la AESAN (Agencia Española de Seguridad Alimentaria) emite una alerta sobre el lote L2025-PO-447 de pechuga de pollo del proveedor "Avícola del Norte". Necesitamos:
> 1. Buscar en el sistema: ¿qué pedidos contienen ese lote?
> 2. Obtener la lista de clientes afectados con sus datos de contacto
> 3. Enviar notificación urgente (email + SMS) a todos los afectados
> 4. Registrar que la notificación fue enviada y recibida
> 5. Todo esto en menos de 4 horas desde que recibimos la alerta
>
> El año pasado tardamos 3 días porque buscamos a mano en Sheets. Casi nos sancionan.
>
> **¿Cómo controlamos la cadena de frío?**
>
> Las furgonetas tienen un sensor de temperatura que registra cada 5 minutos. Ahora los datos se guardan en una tarjeta SD que se descarga al final del día. Necesitamos que eso se transmita en tiempo real al OMS: si la temperatura de la furgoneta de congelados sube por encima de -15°C, quiero una alerta inmediata al coordinador de logística.
>
> **¿Qué pasa con los productos con fecha de caducidad muy próxima?**
>
> Si un producto caduca en menos de 3 días, no se puede enviar al cliente (política interna). El sistema debe impedir que un operario haga picking de un producto cuya caducidad es hoy + 2 días o menos. Y debería generar un report diario de productos que caducan en las próximas 48 horas para que el almacén los retire o los pase a la zona de "liquidación" (venta con descuento en tienda física).

---

### Entrevista 8: Director Financiero — Miguel Torres

> **¿Qué necesitas del OMS?**
>
> - **Facturación automática**: Cada pedido entregado genera una factura electrónica con todos los campos legales (NIF del cliente si lo proporcionó, desglose de IVA por tipo — los alimentos tienen IVA reducido del 4% y 10% según el producto, no todos son al 21%).
> - **Gestión de reembolsos**: Cuando se aprueba una devolución, el reembolso debe procesarse en menos de 48 horas. Stripe tarda 5-10 días laborables en devolver el dinero a la tarjeta del cliente, pero el reembolso tiene que INICIARSE en 48h desde nuestra parte.
> - **Report diario de ventas**: Cada noche a las 02:00, un CSV con todos los pedidos del día: ID pedido, cliente, importe, IVA desglosado, estado, método de pago, código de descuento aplicado (si hay). Esto lo importamos en SAP.
> - **Gestión de descuentos y promociones**: Códigos de descuento (10% en primera compra, envío gratis si > 80€, 2×1 en productos seleccionados). El sistema debe validar que el descuento es aplicable (no caducado, no ya usado si es de un solo uso, importe mínimo cumplido).
>
> **¿Cuántos tipos de IVA manejáis?**
>
> Tres:
> - **4% (superreducido)**: Pan, leche, huevos, frutas, verduras, legumbres, cereales, quesos
> - **10% (reducido)**: Carne, pescado, agua, zumos, conservas
> - **21% (general)**: Bebidas alcohólicas, refrescos, productos de limpieza, bolsas
>
> Cada producto del catálogo tiene asignado su tipo de IVA. El ticket/factura debe desglosar la base imponible y la cuota de IVA por cada tipo. Esto no es opcional — es obligación fiscal.

---

## Datos Cuantitativos Extraídos

| Métrica | Valor actual | Objetivo |
|---|---|---|
| Pedidos/día (normal) | 800 | 1.000 |
| Pedidos/día (Navidad) | 2.400 | 3.000 |
| Ticket medio | 65€ | — |
| Líneas por pedido (media) | 12 productos | — |
| Referencias en catálogo | 3.500 | 4.000 |
| Tasa de error de pedidos | 12% | < 2% |
| Entregas fallidas (1er intento) | 8% | < 3% |
| Incidencias/día | 80-120 | < 40 |
| Tiempo resolución incidencia | 2-3 días | < 24h (< 1h para < 10€) |
| Operarios de almacén | 15 (2 turnos) | — |
| Repartidores | 20 (+10 en Navidad) | — |
| Agotamientos de stock/día | 50-70 | < 10 |
| Producto perdido por "cliente ausente" | 800€/semana | < 200€/semana |
| Downtime actual | 15-20 min, 1-2 veces/mes | < 1h/mes (0 en Navidad) |
| Tiempo de picking por pedido | 8-12 min | < 5 min |

---

## Flujo Completo del Pedido (Camino Feliz)

```
CLIENTE                          OMS                            ALMACÉN / LOGÍSTICA
────────                         ───                            ───────────────────

1. Busca productos            
   por categoría, nombre       
   o código de barras          
                              2. Muestra catálogo con         
                                 stock en tiempo real          
                                 y precio con IVA              

3. Añade productos al         
   carrito. Elige              
   sustitutos (sí/no          
   por producto)               
                              4. Valida stock en              
                                 tiempo real                   

5. Elige franja de            
   entrega (mañana            
   8-14 / tarde 16-20)        
   Introduce dirección         
   e instrucciones             
                              6. Geocodifica dirección        
                                 Valida zona de cobertura      
                                 Calcula coste de envío         

7. Introduce código           
   de descuento (opcional)     
                              8. Valida descuento:            
                                 - No caducado                 
                                 - No ya usado                 
                                 - Importe mínimo OK           
                                 Calcula total con IVA          
                                 desglosado (4%, 10%, 21%)     

9. Paga (tarjeta / Bizum)     
                             10. Procesa pago (Stripe)        
                                 Genera pedido con estado      
                                 CONFIRMADO                    
                                 Envía email + SMS              
                                 confirmación                  
                                                              
                             11. A las 22:00 (cutoff):        
                                 Congela pedidos del día       
                                 siguiente. No más cambios.    
                                                              12. Genera listas de picking
                                                                  por zona (A, B, C)
                                                                  Optimiza ruta de pasillo

                                                              13. Operario recibe picking
                                                                  en tablet. Va producto
                                                                  por producto:
                                                                  - Escanea código de barras
                                                                  - Sistema confirma producto
                                                                    y lote correcto
                                                                  - Descuenta stock en
                                                                    tiempo real
                                                                  - Si producto agotado:
                                                                    propone sustituto o
                                                                    cancela línea

                                                              14. Picking completado:
                                                                  pedido pasa a estado
                                                                  PREPARADO.
                                                                  Congelados se recogen
                                                                  los ÚLTIMOS.

                             15. Asignación automática        15. Coordinador revisa y
                                 de rutas:                        ajusta si necesario.
                                 - Agrupa por zona postal         Aprueba rutas.
                                 - Optimiza orden de entregas
                                 - Prioriza congelados
                                   (menos tiempo fuera)
                                 - 30-40 entregas/repartidor

                                                              16. Repartidor recibe ruta
                                                                  en app móvil.
                                                                  Carga furgoneta.
                                                                  Sale a repartir.

                                                              17. En cada entrega:
                                                                  - Navega con GPS
                                                                  - Llega, marca "en puerta"
                                                                  - Cliente recibe SMS:
                                                                    "Tu pedido llega ahora"
                                                                  - Entrega + foto
                                                                  - Si cliente no está:
                                                                    espera 5 min, llama,
                                                                    si no contesta → falla

18. Cliente ve tracking      
    en tiempo real:           
    "En preparación" →        
    "En camino" →             
    "A 15 min" →              
    "Entregado"               
                             19. Pedido pasa a ENTREGADO     
                                 Genera factura electrónica    
                                 Datos a export nocturno       

─── SI HAY PROBLEMA ──────────────────────────────────────────

20. Cliente reporta          
    incidencia desde web:     
    - Producto dañado (foto)  
    - Producto equivocado     
    - Falta un producto       
                             21. Si importe < 10€:           
                                 Reembolso automático          
                                 inmediato.                    
                                 Si importe >= 10€:            
                                 Revisar foto →                
                                 aprobación de agente          

                             22. Reembolso → Stripe           
                                 (se inicia en < 48h)          
                                 Email de confirmación          
```

---

## Comportamiento Esperado ante Fallos

La directora de operaciones y el CTO mencionaron estos escenarios concretos:

**"¿Qué pasa si Stripe se cae justo cuando un cliente está pagando?"**
> Carmen: "El pedido no se puede confirmar sin pago. Pero no queremos que el carrito se pierda. Que el cliente vea un mensaje claro y pueda reintentar en unos minutos."
> Roberto: "Nada de cobrar dos veces. Si hay duda de si se cobró o no, que el sistema lo verifique antes de reintentar."

**"¿Qué pasa si la base de datos se cae durante el picking?"**
> Javier: "El operario tiene que poder seguir trabajando. Si la tablet pierde conexión con el servidor, que siga mostrando la lista de picking que ya descargó y sincronice cuando vuelva la conexión."
> Roberto: "Modo offline para la tablet del almacén. Es crítico."

**"¿Qué pasa si Google Maps no responde?"**
> Pablo: "Sin rutas, los repartidores pueden salir con las direcciones en orden de código postal — no es óptimo pero funciona. Lo que no puede pasar es que se queden parados esperando."

**"¿Qué pasa si un repartidor no confirma la entrega?"**
> Pablo: "Si después de 2 horas desde la hora estimada de entrega no hay confirmación, el coordinador recibe una alerta. Puede ser que el repartidor olvidó marcarla o que hubo un problema."

**"¿Qué pasa si la temperatura de la furgoneta sube?"**
> Ana: "Si los congelados están por encima de -15°C durante más de 10 minutos, esos productos no se pueden entregar. El coordinador recibe alerta y decide: si está cerca del cliente, acelera la entrega; si no, cancela las entregas de congelados de esa furgoneta y reembolsa."

**"¿Qué pasa si dos clientes piden la última unidad de un producto al mismo tiempo?"**
> Roberto: "Uno gana, uno pierde. El que confirma el pago primero se queda con la unidad. Al otro se le ofrece el sustituto si lo aceptó, o se le cancela esa línea con un mensaje claro."

---

## Restricciones Explícitas

1. **Cutoff operativo a las 22:00**: Después de esa hora, no se admiten cambios en pedidos del día siguiente. El almacén empieza picking a las 6:00.
2. **Cadena de frío**: Refrigerados máx 2h fuera de cámara. Congelados máx 45 min. El sistema debe tener en cuenta estos tiempos al generar rutas.
3. **FEFO obligatorio**: First Expired, First Out. Siempre se envía el lote con caducidad más próxima (siempre que queden al menos 3 días de margen).
4. **Cobertura geográfica**: Solo España peninsular. Sin Baleares, Canarias, Ceuta ni Melilla (en MVP).
5. **Peso máximo por pedido**: 25 kg. Si el pedido supera 25 kg, se divide en 2 envíos (mismo repartidor, mismo horario).
6. **Sustitutos**: El cliente decide al hacer el pedido si acepta sustitutos, y para qué productos. El sustituto debe ser de la misma categoría y precio igual o inferior.
7. **Reembolso automático**: Solo para incidencias de importe < 10€ y con foto del repartidor adjunta. Máximo 2 reembolsos automáticos por cliente al mes (anti-fraude).

---

## Lo que NO está en el MVP

- App móvil para clientes (solo web responsive)
- App móvil para repartidores (en MVP usan web móvil; app nativa en fase 2)
- Integración en tiempo real con SAP (solo export CSV nocturno)
- Multi-idioma (solo español en MVP)
- Multi-moneda (solo EUR)
- Programa de fidelización / puntos
- Suscripciones recurrentes ("mi pedido semanal")
- Chat con IA para soporte
- Marketplace para vendedores externos

---

