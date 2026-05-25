# FSD ligero — FTGO

## 1. Introducción

Este documento de especificación funcional (FSD) describe los casos de uso del sistema FTGO en su arquitectura objetivo de microservicios. Parte del `docs/prd.md` y de las 3 user stories semilla del brief (US-01, US-02, US-03) para formalizar 7 casos de uso con bloques Given/When/Then trazables a capacidades de negocio del PRD, capítulos del libro Richardson o restricciones NFR. El alcance cubre el flujo transaccional completo: toma de pedido, preparación, despacho, pago, tracking, cancelación y notificaciones; queda fuera el diseño de APIs, esquemas de base de datos y operación productiva final.

---

## 2. Tabla de UCs

| UC-ID | Título | Actor primario | Capacidad PRD | Origen |
|---|---|---|---|---|
| UC-01 | Tomar pedido | Consumidor | Order Taking | US-01 (brief) |
| UC-02 | Aceptar / rechazar ticket de pedido | Restaurante | Order Fulfillment / Kitchen | US-02 (brief) |
| UC-03 | Asignar pedido a courier | Dispatcher | Delivery | US-03 (brief) |
| UC-04 | Procesar pago del pedido | Sistema de pagos | Billing & Accounting | Libro Cap 3 — saga de Order Service |
| UC-05 | Tracking en tiempo real del pedido | Consumidor | Delivery | PRD NFR-01 |
| UC-06 | Cancelar pedido | Consumidor / Sistema | Order Fulfillment / Kitchen | Libro Cap 4 — saga de cancelación; PRD §3.4 |
| UC-07 | Notificar cambio de estado al consumidor | Sistema de notificaciones | Notifications | PRD §3.7 |

---

## 3. Detalle de UCs

### UC-01: Tomar pedido

| Campo           | Valor                              |
|-----------------|------------------------------------|
| Actor primario  | Consumidor                         |
| Capacidad PRD   | Order Taking                       |
| Origen          | US-01 (brief)                      |

**Precondiciones**: el Consumidor está autenticado, ha seleccionado un restaurante disponible y tiene al menos 1 ítem en el carrito con un método de pago registrado.

**Flujo principal**:
1. El Consumidor revisa el resumen del carrito con ítems, cantidades y precios.
2. El Consumidor confirma la dirección de entrega.
3. El Consumidor selecciona el método de pago y pulsa "Confirmar pedido".
4. El sistema valida la disponibilidad del restaurante y el stock de los ítems solicitados.
5. El sistema crea el pedido en estado `PENDING_PAYMENT` y asigna un número único de pedido.
6. El sistema devuelve al Consumidor la confirmación con el número único de pedido.

**Flujos alternativos**:
- FA-1 (restaurante no disponible): el sistema rechaza la creación del pedido, muestra el mensaje "Restaurante no disponible en este momento" y el pedido no se crea.
- FA-2 (ítem sin stock): el sistema notifica al Consumidor qué ítems no están disponibles y le permite ajustar el carrito antes de reintentar.

**Postcondiciones**: existe un registro de pedido en estado `PENDING_PAYMENT` con número único asignado, trazable a un Consumidor autenticado y a un Restaurante disponible.

**Given/When/Then**:
- Given: el Consumidor tiene un carrito con ≥ 1 ítem disponible y un método de pago válido registrado, y el restaurante está activo.
- When: el Consumidor confirma el pedido.
- Then: el sistema crea el pedido en estado `PENDING_PAYMENT` y retorna un número único de pedido al Consumidor.

---

### UC-02: Aceptar / rechazar ticket de pedido

| Campo           | Valor                              |
|-----------------|------------------------------------|
| Actor primario  | Restaurante                        |
| Capacidad PRD   | Order Fulfillment / Kitchen        |
| Origen          | US-02 (brief)                      |

**Precondiciones**: existe un pedido en estado `PAYMENT_APPROVED` que ha generado un ticket de cocina; el Restaurante está autenticado en su dashboard y tiene capacidad para recibir nuevos tickets.

**Flujo principal**:
1. El Restaurante recibe una notificación de nuevo ticket de pedido en su dashboard.
2. El Restaurante revisa el detalle del ticket: ítems, cantidades y tiempo estimado de preparación sugerido.
3. El Restaurante ingresa el tiempo estimado de preparación (en minutos).
4. El Restaurante pulsa "Aceptar ticket".
5. El sistema actualiza el estado del ticket a `ACCEPTED` y notifica al servicio de Order Fulfillment.
6. El sistema envía una notificación de estado actualizado al Consumidor.

**Flujos alternativos**:
- FA-1 (Restaurante rechaza el ticket): el Restaurante pulsa "Rechazar" e ingresa el motivo; el sistema cancela el pedido, inicia la saga de cancelación y notifica al Consumidor con el motivo del rechazo.
- FA-2 (timeout sin respuesta): si el Restaurante no responde en el tiempo configurado (por defecto 5 min), el sistema rechaza automáticamente el ticket e inicia la saga de cancelación.

**Postcondiciones**: el ticket de pedido queda en estado `ACCEPTED` con tiempo estimado de preparación registrado y el Consumidor ha recibido la notificación de aceptación; o el pedido queda cancelado si fue rechazado.

**Given/When/Then**:
- Given: existe un ticket de pedido en el dashboard del Restaurante en estado pendiente de respuesta, y el Restaurante tiene capacidad disponible en cocina.
- When: el Restaurante acepta el ticket con un tiempo estimado de preparación.
- Then: el ticket queda en estado `ACCEPTED`, el tiempo estimado queda registrado en el sistema y el Consumidor recibe notificación de aceptación.

---

### UC-03: Asignar pedido a courier

| Campo           | Valor                              |
|-----------------|------------------------------------|
| Actor primario  | Dispatcher                         |
| Capacidad PRD   | Delivery                           |
| Origen          | US-03 (brief)                      |

**Precondiciones**: el pedido está en estado `READY_FOR_PICKUP` (el restaurante marcó la preparación como completada); existe al menos un Courier disponible en la zona con disponibilidad marcada en la app.

**Flujo principal**:
1. El sistema de Dispatcher detecta que el pedido está listo para retirar.
2. El Dispatcher consulta la lista de couriers disponibles cercanos al restaurante.
3. El Dispatcher selecciona el Courier óptimo según proximidad y carga de trabajo.
4. El sistema envía la oferta de asignación al Courier con detalle del pedido y ruta sugerida.
5. El Courier acepta la asignación dentro del timeout configurado (30 s).
6. El sistema actualiza el estado del pedido a `ASSIGNED_TO_COURIER` y registra el Courier asignado.
7. El Courier recibe la ruta optimizada al restaurante y la dirección de entrega del Consumidor.

**Flujos alternativos**:
- FA-1 (Courier rechaza o no responde en 30 s): el sistema ofrece la asignación al siguiente Courier disponible; si no hay couriers disponibles, el pedido queda en cola de espera y se alerta al operador.
- FA-2 (ningún Courier disponible en la zona): el sistema pone el pedido en estado `AWAITING_COURIER` y reintenta la asignación cada 60 s durante un máximo de 10 min.

**Postcondiciones**: el pedido queda en estado `ASSIGNED_TO_COURIER` con el identificador del Courier registrado; el Courier tiene acceso a la ruta optimizada; el Consumidor recibe notificación de que su pedido está en camino.

**Given/When/Then**:
- Given: el pedido está en estado `READY_FOR_PICKUP` y existe al menos un Courier disponible a menos de 3 km del restaurante.
- When: el Dispatcher asigna el pedido al Courier más cercano disponible.
- Then: el pedido queda en estado `ASSIGNED_TO_COURIER`, el Courier asignado queda registrado en el sistema y el Courier recibe la ruta optimizada.

---

### UC-04: Procesar pago del pedido

| Campo           | Valor                                          |
|-----------------|------------------------------------------------|
| Actor primario  | Sistema de pagos                               |
| Capacidad PRD   | Billing & Accounting                           |
| Origen          | Libro Cap 3 — saga de Order Service            |

**Precondiciones**: existe un pedido en estado `PENDING_PAYMENT`; el Consumidor tiene un método de pago válido registrado; el servicio de Billing & Accounting está operativo o en modo de cola de retry (NFR-04).

**Flujo principal**:
1. El sistema de Order Service publica el evento `OrderCreated` con el total del pedido.
2. El sistema de Billing & Accounting consume el evento e inicia el cargo al método de pago del Consumidor a través de Stripe.
3. Stripe procesa la transacción y devuelve confirmación de cobro.
4. El sistema de Billing & Accounting publica el evento `PaymentApproved`.
5. El sistema de Order Service consume el evento y actualiza el estado del pedido a `PAYMENT_APPROVED`.
6. El sistema de Billing & Accounting registra la transacción para el cálculo futuro de comisiones y payouts.

**Flujos alternativos**:
- FA-1 (Stripe rechaza la transacción): el sistema publica el evento `PaymentFailed`; el sistema de Order Service cancela el pedido y notifica al Consumidor que el pago fue rechazado.
- FA-2 (Stripe no disponible temporalmente): el sistema encola el intento de cobro con retry configurable; el pedido permanece en `PENDING_PAYMENT`; si supera el timeout máximo, el pedido se cancela y se notifica al Consumidor.

**Postcondiciones**: el pedido queda en estado `PAYMENT_APPROVED` con el identificador de transacción de Stripe registrado; se ha creado el registro contable de la transacción en Billing & Accounting.

**Given/When/Then**:
- Given: existe un pedido en estado `PENDING_PAYMENT` con un método de pago válido del Consumidor y el evento `OrderCreated` ha sido publicado en el bus de eventos.
- When: el sistema de Billing & Accounting procesa el cargo a través de Stripe.
- Then: Stripe confirma el cobro, el evento `PaymentApproved` es publicado y el pedido queda en estado `PAYMENT_APPROVED` con el identificador de transacción de Stripe registrado.

---

### UC-05: Tracking en tiempo real del pedido

| Campo           | Valor                              |
|-----------------|------------------------------------|
| Actor primario  | Consumidor                         |
| Capacidad PRD   | Delivery                           |
| Origen          | PRD NFR-01                         |

**Precondiciones**: el pedido está en estado `ASSIGNED_TO_COURIER` o posterior; el Courier tiene la app activa y está publicando su posición GPS; el Consumidor tiene la app activa en la pantalla de seguimiento.

**Flujo principal**:
1. El Consumidor navega a la pantalla de seguimiento de su pedido activo.
2. El sistema muestra el mapa con la posición actual del Courier y la ruta estimada al destino.
3. El sistema actualiza la posición del Courier en el mapa con latencia ≤ 200 ms p95 (NFR-01).
4. El sistema muestra el tiempo estimado de llegada (ETA) calculado con la posición del Courier y las condiciones de tráfico vía Google Maps.
5. Cuando el Courier marca la entrega como completada, el sistema actualiza el estado del pedido a `DELIVERED` y muestra la confirmación al Consumidor.

**Flujos alternativos**:
- FA-1 (pérdida de señal GPS del Courier): el sistema muestra la última posición conocida con indicador de "señal perdida" y reintenta la actualización; el flujo de pedido no se bloquea.
- FA-2 (Google Maps no disponible): el sistema muestra la posición del Courier sin ruta optimizada y sin ETA; la entrega continúa sin afectar el flujo crítico de pedidos.

**Postcondiciones**: el Consumidor ha podido visualizar la posición del Courier durante el transcurso de la entrega con latencia ≤ 200 ms p95; el pedido queda en estado `DELIVERED` al completarse la entrega.

**Given/When/Then**:
- Given: el pedido está en estado `ASSIGNED_TO_COURIER` y el Courier publica su posición GPS cada ≤ 5 s.
- When: el Consumidor consulta la pantalla de tracking de su pedido activo.
- Then: el sistema muestra la posición actualizada del Courier en el mapa con latencia ≤ 200 ms p95 y el ETA calculado visible en pantalla.

---

### UC-06: Cancelar pedido

| Campo           | Valor                                                  |
|-----------------|--------------------------------------------------------|
| Actor primario  | Consumidor / Sistema                                   |
| Capacidad PRD   | Order Fulfillment / Kitchen                            |
| Origen          | Libro Cap 4 — saga de cancelación; PRD §3.4            |

**Precondiciones**: existe un pedido en un estado que admite cancelación (`PENDING_PAYMENT` o `PAYMENT_APPROVED`); el pedido no ha sido marcado como `IN_PREPARATION` por el restaurante si la cancelación la inicia el Consumidor; o bien el Sistema ha detectado un evento de fallo en la saga que requiere compensación.

**Flujo principal**:
1. El Consumidor solicita la cancelación desde la app, o el Sistema detecta un evento de fallo de saga que requiere compensación.
2. El sistema verifica que el estado del pedido es elegible para cancelación.
3. El sistema inicia la saga de cancelación publicando el evento `CancelOrder`.
4. El servicio de Billing & Accounting consume el evento y ejecuta el reembolso si el pago fue procesado previamente.
5. El servicio de Order Fulfillment / Kitchen consume el evento y marca el ticket como cancelado en el dashboard del Restaurante.
6. El sistema actualiza el estado del pedido a `CANCELLED`.
7. El sistema notifica al Consumidor con el motivo de cancelación y la confirmación del reembolso si aplica.

**Flujos alternativos**:
- FA-1 (pedido ya en preparación): el sistema rechaza la cancelación iniciada por el Consumidor, muestra "El pedido ya está en preparación y no puede cancelarse" y mantiene el estado actual.
- FA-2 (fallo en el reembolso): el sistema registra el reembolso pendiente en cola de reintentos con backoff; el pedido queda en `CANCELLED` con indicador de reembolso pendiente y el equipo de back office recibe alerta.

**Postcondiciones**: el pedido queda en estado `CANCELLED`; si existía cargo procesado, se ha iniciado el reembolso al método de pago del Consumidor; el ticket de cocina queda cancelado en el dashboard del Restaurante.

**Given/When/Then**:
- Given: existe un pedido en estado `PAYMENT_APPROVED` que no ha entrado en preparación en el restaurante.
- When: el Consumidor solicita la cancelación del pedido desde la app.
- Then: el sistema actualiza el estado del pedido a `CANCELLED`, publica el evento `CancelOrder`, inicia el reembolso vía Billing & Accounting y el Consumidor recibe la notificación de cancelación con confirmación de reembolso.

---

### UC-07: Notificar cambio de estado al consumidor

| Campo           | Valor                              |
|-----------------|------------------------------------|
| Actor primario  | Sistema de notificaciones          |
| Capacidad PRD   | Notifications                      |
| Origen          | PRD §3.7                           |

**Precondiciones**: se ha publicado un evento de cambio de estado de un pedido activo en el bus de eventos (eventos: `OrderCreated`, `PaymentApproved`, `TicketAccepted`, `OrderAssignedToCourier`, `OrderDelivered`, `OrderCancelled`); el Consumidor tiene al menos un canal de notificación configurado (push, SMS o email).

**Flujo principal**:
1. El servicio de Notifications consume el evento de cambio de estado desde el bus de eventos.
2. El sistema determina el tipo de notificación correspondiente al evento recibido.
3. El sistema selecciona el canal de entrega según la preferencia del Consumidor (push > SMS > email).
4. El sistema envía la notificación a través del proveedor correspondiente: push/SMS vía Twilio, email vía SendGrid.
5. El proveedor confirma la entrega de la notificación al sistema.
6. El sistema registra el evento de notificación enviada con timestamp y canal utilizado.

**Flujos alternativos**:
- FA-1 (canal push no disponible): el sistema reintenta por el siguiente canal preferido (SMS o email) garantizando entrega al menos una vez.
- FA-2 (proveedor externo temporalmente caído): el sistema encola la notificación y reintenta con backoff exponencial; la notificación llega con eventual consistency respecto al evento origen, con convergencia ≤ 5 s en condiciones normales (PRD NFR-05).

**Postcondiciones**: el Consumidor ha recibido la notificación del cambio de estado en al menos uno de sus canales configurados; el evento de notificación queda registrado con timestamp y canal de entrega en el sistema.

**Given/When/Then**:
- Given: el pedido ha transitado al estado `PAYMENT_APPROVED` y el Consumidor tiene un canal de notificación push activo configurado.
- When: el servicio de Notifications consume el evento `PaymentApproved` desde el bus de eventos.
- Then: el sistema envía una notificación push al Consumidor en ≤ 5 s desde la emisión del evento, el Consumidor recibe el mensaje de confirmación de pago aprobado y el evento queda registrado con timestamp en el sistema.
