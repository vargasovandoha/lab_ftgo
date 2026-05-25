# ADR-0002: Mecanismo IPC predominante y gestión de transacciones distribuidas

**Status**: Accepted  
**Fecha**: 2026-05-25  
**Decide**: Equipo de arquitectura FTGO  
**Depende de**: ADR-0001 (Macro-services como estilo arquitectónico)

---

## 1. Contexto

Aceptado el estilo macro-services en ADR-0001, los 5 servicios principales (Order, Kitchen, Billing, Delivery, Notifications) deben coordinarse para completar el flujo de toma de pedido (UC-01 → UC-04 → UC-02 → UC-03 → UC-07) sin compartir base de datos. La decisión arquitectónica pendiente es: **¿qué mecanismo de comunicación entre servicios (IPC) se adopta como predominante?**

Esta decisión es urgente porque condiciona el diseño de cada servicio: si se elige comunicación sincrónica, cada servicio expone endpoints REST/gRPC; si se elige asíncrona, se necesita un message broker y un esquema de gestión de transacciones distribuidas (sagas). Una decisión tardía o inconsistente generaría deuda técnica imposible de revertir en el horizonte Strangler Fig (18–24 meses, NFR-07 / R-07).

**Problema central:** el flujo Order Taking cruza múltiples bounded contexts sin transacción ACID compartida. Cuando Stripe falla (R-04), cuando el restaurante rechaza el ticket (UC-02 FA-1) o cuando un courier no acepta (UC-03 FA-1), el sistema debe compensar de forma consistente sin perder pedidos ni generar cobros huérfanos.

### Restricciones que esta decisión debe respetar

| # | Restricción | Origen |
|---|---|---|
| R-01 | Latencia ≤ 200 ms p95 para acciones del consumidor | NFR-01 / Brief §A.4 |
| R-02 | Disponibilidad ≥ 99.9 % mensual en flujo Order Taking | NFR-02 / Brief §A.4 |
| R-03 | Escalabilidad 5× tráfico base; X-axis e Y-axis del Scale Cube | NFR-03 / Brief §A.4 |
| R-04 | Resiliencia ante Stripe caído: encolar pedidos, 0 pérdidas de órdenes | NFR-04 / Brief §A.4 |
| R-05 | Consistencia fuerte dentro del aggregate del pedido; eventual consistency ≤ 5 s en reporting | NFR-05 / Brief §A.4 |
| R-06 | Correlation ID propagado + distributed tracing con retención mínima de 7 días | NFR-06 / Brief §A.4 |
| R-07 | Monolito legacy operativo 18–24 meses; activación/reversión por feature flag | NFR-07 / Brief §A.4 |
| R-08 | Java/Spring Boot en servicios core; PCI-DSS delegado a Stripe; GDPR para datos de consumidores | Brief §A.4 |

---

## 2. Opciones consideradas

### Opción 1: Comunicación sincrónica predominante (REST/gRPC en cadena)

**Descripción**: Order Service llama sincrónicamente a Billing Service (Stripe), Kitchen Service y Delivery Service para completar la toma de pedido. Cada servicio expone endpoints REST o gRPC. Las compensaciones (cancelaciones) se implementan como llamadas síncronas de rollback encadenadas. No se requiere message broker.

**Pros**:
- Simplicidad operativa inicial: no se necesita infraestructura de mensajería — reduce el tiempo de onboarding del equipo durante la fase Strangler Fig (NFR-07 / R-07).
- Trazabilidad directa: el correlation ID se propaga en headers HTTP sin middleware adicional → cubre NFR-06 / R-06 con menor complejidad.
- Modelo mental familiar para el equipo Java/Spring Boot existente → R-08.

**Contras**:
- Acoplamiento temporal: si Stripe está caído, la cadena síncrona bloquea la toma de pedido completa → viola R-04 (0 pérdidas de pedidos) directamente; el pedido se pierde o queda en estado inconsistente.
- Cascading failure: 3 llamadas síncronas en cadena (Order → Billing → Kitchen → Delivery) degradan disponibilidad multiplicativamente; con 99.9 % por servicio → disponibilidad compuesta ≈ 99.6 % → viola R-02 (99.9 % requerido).
- No escala en pico 5×: el thread pool de Order Service se satura esperando respuestas de downstream → viola R-03; escalado X-axis requiere escalar todos los servicios simultáneamente.
- Transacciones distribuidas sin saga: una compensación requiere implementar rollback manual ad-hoc en cada servicio, propenso a inconsistencias (libro Cap 4: "sagas are the solution to distributed transactions").

**Impacto en NFRs**: NFR-01 ✓ (latencia baja en condiciones normales, ~50–80 ms por llamada); NFR-02 ✗ (disponibilidad compuesta < 99.9 % con ≥ 3 servicios en cadena); NFR-04 ✗ (no puede encolar pedidos con Stripe caído); NFR-05 ⚠ (consistencia fuerte posible pero sin compensación robusta); NFR-03 ✗ (escalado acoplado)

---

### Opción 2: Mensajería asíncrona con sagas de coreografía (Kafka + eventos sin orquestador)

**Descripción**: Los servicios se comunican publicando y consumiendo eventos en Kafka (o similar). Order Service publica `OrderCreated`; Billing Service reacciona publicando `PaymentApproved` o `PaymentFailed`; Kitchen Service reacciona a `PaymentApproved` publicando `TicketAccepted`; etc. No hay orquestador central: cada servicio conoce su rol en la saga y reacciona al evento entrante. Las compensaciones se disparan con eventos de tipo `XxxFailed` que cada servicio interpreta y propaga (Richardson Cap 4 — Choreography-based saga).

**Pros**:
- Desacoplamiento temporal máximo: Order Service no bloquea esperando Stripe → R-04 cubierto; pedidos encolados mientras Stripe está caído → 0 pérdidas.
- Alta disponibilidad: el fallo de Billing Service no impide que Order Service acepte el pedido → mejora cobertura de NFR-02 / R-02.
- Escalado independiente: cada consumidor Kafka escala sus particiones de forma autónoma → NFR-03 / R-03 cubierto.
- Alineación con libro: Richardson Cap 4 describe coreografía como solución para sagas simples de pocos pasos.

**Contras**:
- Riesgo de lógica de saga fragmentada: con 5+ servicios, la lógica de compensación está distribuida en múltiples consumers → difícil de razonar, probar y depurar; Richardson Cap 4 advierte explícitamente: "choreography works for simple sagas; for complex ones, orchestration is clearer".
- Dificultad de observabilidad: no hay un único punto que muestre el estado de la saga en curso; sin orquestador, reconstruir el historial de un pedido fallido requiere correlacionar eventos en múltiples topics → impacta NFR-06 / R-06 operativamente.
- Acoplamiento implícito: los servicios se acoplan vía contratos de eventos (schema del evento); cambios en el schema propagan breaking changes silenciosamente sin versioning explícito.
- Complejidad de testing: probar una cancelación con 4 pasos de compensación requiere orquestar 4+ consumers en test de integración → eleva el tiempo de CI en la fase Strangler Fig (R-07).

**Impacto en NFRs**: NFR-01 ⚠ (latencia percibida por el consumidor puede aumentar ≥ 2–5 s hasta confirmación de pago por naturaleza asíncrona, a menos que se disocie "aceptación de pedido" de "confirmación de cobro"); NFR-02 ✓; NFR-04 ✓; NFR-03 ✓; NFR-05 ✓ (con idempotencia en consumers); NFR-06 ⚠ (trazabilidad compleja sin orquestador central)

---

### Opción 3: Mensajería asíncrona con sagas de orquestación (Kafka + saga orchestrator en Order Service)

**Descripción**: Order Service actúa como orquestador de la saga de toma de pedido. Publica comandos a servicios downstream (e.g., `AuthorizePayment` a Billing, `CreateTicket` a Kitchen) a través de Kafka y espera respuestas en reply channels. Si un paso falla, el orquestador emite comandos de compensación (`RejectTicket`, `RefundPayment`). El orquestador persiste el estado de la saga en su propia base de datos. Los eventos externos (e.g., `OrderDelivered`) se publican en topics separados para consumidores downstream como Notifications. El mecanismo predominante para **comandos transaccionales** es async + saga orquestada; REST/gRPC se usa para **consultas** (CQRS: queries síncronas, commands asíncronos) (Richardson Cap 4 — Orchestration-based saga; Cap 3 — Command/Query separation).

**Pros**:
- Visibilidad centralizada del estado de la saga: el orquestador persiste explícitamente cada transición → facilita debugging, auditoría y distributed tracing (NFR-06 / R-06); un solo punto de consulta para "¿en qué paso está el pedido X?".
- Compensación explícita y testeable: los pasos de rollback se implementan como comandos nombrados en el orquestador → fácil de probar unitariamente y de razonar sobre casos de fallo → Richardson Cap 4 recomienda orquestación para sagas de ≥ 4 pasos.
- Desacoplamiento temporal preservado: Stripe caído → comando `AuthorizePayment` queda en cola Kafka sin bloquear Order Service → R-04 cubierto, 0 pérdidas de pedidos.
- Escalado independiente: Billing, Kitchen, Delivery escalan sus consumers de forma autónoma (Scale Cube X-axis) → R-03 ✓; el orquestador escala verticalmente o con particionado por Order ID (Y-axis).
- Compatible con Strangler Fig: el orquestador puede fanout a servicios nuevos (extraídos del monolito) Y al monolito legacy vía adapter topic, activando/desactivando por feature flag → R-07 ✓.

**Contras**:
- El orquestador puede convertirse en God Object si no se acotan sus responsabilidades: debe manejar solo el flujo transaccional del pedido, no lógica de negocio de otros bounded contexts → riesgo de Distributed Monolith por acoplamiento lógico.
- Latencia percibida en confirmación de pago: el consumidor no recibe `PaymentApproved` de forma sincrónica → la pantalla de confirmación debe diseñarse como UX asíncrona (polling o WebSocket); esto puede superar los 200 ms p95 de NFR-01 si el diseño UX no desacopla "pedido aceptado" de "pago confirmado" → riesgo cuantificado: si Stripe tarda ≥ 150 ms + overhead Kafka ≥ 30–50 ms → confirmación de cobro llega en ~200–300 ms, superando NFR-01 para ese sub-paso específico.
- Complejidad de infraestructura: se requiere Kafka (o broker equivalente), schema registry y reply channels → mayor costo operativo inicial que Opción 1.
- Idempotencia obligatoria: todos los consumers deben ser idempotentes para tolerar reintentos de Kafka → disciplina de desarrollo adicional; fallo en idempotencia genera cobros duplicados (Billing) o tickets duplicados (Kitchen).

**Impacto en NFRs**: NFR-01 ✓ para "aceptación de pedido" (< 200 ms); NFR-01 ⚠ para "confirmación de pago" (~200–300 ms; mitigado con UX asíncrona y WebSocket); NFR-02 ✓; NFR-03 ✓; NFR-04 ✓ (cola Kafka absorbe caídas de Stripe); NFR-05 ✓ (orquestador mantiene consistencia fuerte del aggregate pedido; eventual consistency ≤ 5 s para Notifications); NFR-06 ✓ (correlation ID propagado en headers Kafka + estado de saga persistido); NFR-07 ✓ (adapter topic permite coexistencia con monolito)

---

## 3. Decisión

**Se adopta la Opción 3: Mensajería asíncrona con sagas de orquestación en Order Service como mecanismo IPC predominante para comandos transaccionales, combinada con REST/gRPC para consultas (patrón CQRS).**

### Justificación

**Richardson Cap 4** distingue explícitamente entre sagas de coreografía (apropiadas para flujos simples de ≤ 3 pasos) y sagas de orquestación (recomendadas para flujos complejos de ≥ 4 pasos con múltiples compensaciones). El flujo de Order Taking de FTGO involucra 5 pasos transaccionales (crear pedido → autorizar pago → crear ticket cocina → asignar courier → notificar) y hasta 3 rutas de compensación distintas (rechazo restaurante, fallo Stripe, rechazo courier) — excede claramente el umbral donde la coreografía se vuelve inmanejable.

**Brief §A.4 — Tolerancia a fallos externos (R-04):** la arquitectura sincrónica (Opción 1) viola este requisito directamente al no poder encolar pedidos con Stripe caído. La orquestación asíncrona con Kafka satisface R-04 de forma estructural: el comando `AuthorizePayment` queda persistido en el broker hasta que Stripe responda.

**Brief §A.4 — Disponibilidad (R-02):** la disponibilidad compuesta de una cadena sincrónica de 4 servicios (99.9 %⁴ ≈ 99.6 %) viola el SLA del 99.9 % mensual requerido para Order Taking. El desacoplamiento asíncrono rompe esta dependencia multiplicativa.

**Brief §A.4 — Migración incremental (R-07):** el orquestador en Order Service puede enrutar comandos simultáneamente al monolito legacy (vía adapter) y a los nuevos servicios extraídos, activando/desactivando por feature flag sin downtime — patrón Strangler Fig compatible.

**Richardson Cap 3 (CQRS):** las consultas (estado del pedido, tracking, menús) se sirven vía REST/gRPC sincrónico directamente desde las read-models de cada servicio — no requieren pasar por el message broker, lo que satisface NFR-01 (≤ 200 ms p95) para las acciones de consulta del consumidor.

---

## 4. Consecuencias

### Consecuencias positivas

- **Tolerancia a fallos externos garantizada estructuralmente**: los comandos transaccionales persisten en Kafka independientemente del estado de Stripe, Google Maps o cualquier servicio downstream — 0 pérdidas de pedidos por fallo externo (R-04).
- **Disponibilidad compuesta desacoplada**: Order Service puede aceptar y confirmar pedidos aunque Billing, Kitchen o Delivery estén degradados; la disponibilidad del flujo Order Taking ya no depende multiplicativamente de sus downstream.
- **Observabilidad centralizada del estado de saga**: el orquestador en Order Service persiste cada transición de estado → un único punto de consulta para diagnosticar pedidos atascados; facilita la resolución de incidentes (NFR-06 / R-06).
- **Compensaciones explícitas y trazables**: cada paso de rollback es un comando nombrado en el orquestador (`RefundPayment`, `RejectTicket`) → testeable unitariamente, auditable y visible en dashboards de tracing.
- **Coexistencia con monolito durante Strangler Fig**: los adapter topics permiten que el orquestador enrute comandos al monolito o a los servicios extraídos con un feature flag, sin downtime (R-07).

### Consecuencias negativas

- **Latencia de confirmación de pago puede superar NFR-01**: la confirmación de cobro (`PaymentApproved`) llega al consumidor de forma asíncrona; con Stripe processing ≥ 150 ms + overhead Kafka 30–50 ms → la confirmación de pago puede llegar en 200–350 ms p95, superando el umbral de 200 ms del NFR-01 para ese sub-paso específico. Mitigación: diseño UX asíncrono (confirmación inmediata de "pedido recibido"; actualización posterior de "pago aprobado" vía WebSocket o push notification) — requiere coordinación con equipo de producto antes de implementar.
- **Idempotencia obligatoria en todos los consumers**: un fallo de Kafka puede re-entregar mensajes; sin idempotencia, Billing procesaría cobros duplicados o Kitchen crearía tickets dobles. Cada consumer debe implementar deduplicación por message ID → disciplina de desarrollo adicional y riesgo de bugs sutiles en las primeras iteraciones.
- **Complejidad operativa de Kafka**: se añade infraestructura de broker, schema registry y monitoreo de consumer lag → incrementa el tiempo de setup de la plataforma en ~2–4 semanas respecto a Opción 1; requiere un SRE o proceso de on-call adicional para el broker durante la migración Strangler Fig.
- **Riesgo de God Object en Order Service (orquestador)**: si no se limita estrictamente el scope del orquestador al flujo transaccional del pedido, puede absorber lógica de negocio de otros bounded contexts y convertirse en un Distributed Monolith por acoplamiento lógico (anti-patrón identificado en ADR-0001). Mitigación: el orquestador solo emite comandos y recibe respuestas; nunca implementa lógica de dominio de Billing, Kitchen o Delivery.

### Follow-ups

| # | Artefacto | Descripción | Depende de |
|---|---|---|---|
| ADR-0003 | Estrategia de datos y bounded contexts | Decidir si cada servicio tiene BD propia (Database per Service) o si algunos comparten esquema durante la transición Strangler Fig; incluir decisión sobre Outbox Pattern para garantizar exactly-once publishing | ADR-0002 |
| POC-01 | Latencia end-to-end del flujo Order Taking | Medir latencia p95 de UC-01 completo (crear pedido + autorizar pago) con Kafka en la ruta; validar si el diseño UX asíncrono (confirmación en dos fases) satisface la percepción de ≤ 200 ms del consumidor | ADR-0002 |
| POC-02 | Idempotencia de consumers bajo redelivery | Probar el comportamiento de Billing y Kitchen consumers ante mensaje duplicado de Kafka; validar mecanismo de deduplicación (e.g., `processed_messages` table con message ID) | ADR-0003 |
