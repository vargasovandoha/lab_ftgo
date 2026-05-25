# PRD ligero — FTGO

## 1. Contexto y objetivos

FTGO (Food To Go) es una plataforma de delivery de comida que conecta consumidores con restaurantes. Lleva varios años operando como una aplicación monolítica Java empaquetada como WAR, y ha acumulado los síntomas clásicos del *infierno monolítico*: builds lentos, escalado conflictivo entre módulos, falta de aislamiento de fallos, lock-in tecnológico y un equipo bloqueado por el tamaño de la base de código. Ante el crecimiento sostenido del negocio, la dirección decidió migrar a una arquitectura de microservicios para recuperar velocidad de entrega y resiliencia operativa [Brief §A.1].

Este PRD ligero documenta las capacidades de negocio, stakeholders y requisitos no funcionales que servirán como entrada para un FSD y para dos ADRs arquitectónicos. El alcance es el laboratorio de migración incremental (Strangler Fig, horizonte 18–24 meses); no cubre el diseño detallado de implementación ni la operación productiva final.

---

## 2. Stakeholders

| Rol | Necesidad principal | Origen |
| :--- | :--- | :--- |
| Consumidor | UX rápida, transparencia del estado del pedido, tracking en tiempo real | [Brief §A.2] |
| Restaurante | Gestión de tickets de cocina, control de carga, dashboard de pedidos | [Brief §A.2] |
| Courier | Asignaciones cercanas, rutas optimizadas, pago confiable | [Brief §A.2] |
| Empleado FTGO *(back office)* | Visibilidad operativa, reportes, resolución de incidentes | [Brief §A.2] |
| Equipo de arquitectura | Calidad arquitectónica, trazabilidad de decisiones, mantenibilidad | [Brief §A.2] |
| Sistemas externos | Integración estable con pasarela de pago (Stripe), mapas (Google Maps) y notificaciones (SendGrid / Twilio); SLAs predecibles | [Brief §A.2] |

---

## 3. Capacidades de negocio

### 3.1 Consumer Management

Gestiona el ciclo de vida de los consumidores: registro, autenticación, perfiles, direcciones de entrega y preferencias. Es la fuente de verdad de la identidad del usuario en la plataforma. Los datos de consumidor están sujetos a cumplimiento GDPR/local y deben ser consistentes dentro del aggregate del consumidor, aunque el reporting puede aceptar eventual consistency [Brief §A.3].

### 3.2 Restaurant Management

Cubre el alta y configuración de restaurantes en la plataforma: gestión de menús, horarios de operación y disponibilidad en tiempo real. Provee la información que Order Taking necesita para validar pedidos. Las actualizaciones de menú pueden propagarse con eventual consistency hacia otros servicios consumidores [Brief §A.3].

### 3.3 Order Taking

Responsable de la toma de pedidos: recepción de la selección del consumidor, validación de disponibilidad del restaurante, cálculo del total, confirmación del pedido y emisión del número de pedido único. Es el punto de entrada crítico del flujo transaccional; requiere alta disponibilidad (99.9 % mensual) y consistencia fuerte dentro del aggregate del pedido [Brief §A.3].

### 3.4 Order Fulfillment / Kitchen

Entrega los tickets al restaurante y gestiona el estado de preparación (aceptado, en preparación, listo para retirar). Orquesta la transición del pedido desde la toma hasta que el courier puede recogerlo. Si el restaurante rechaza el ticket, coordina la cancelación y la notificación al consumidor [Brief §A.3].

### 3.5 Delivery

Gestiona la asignación de couriers disponibles, el cálculo de rutas optimizadas y el tracking en tiempo real de la entrega. Consume eventos de disponibilidad del courier y de pedido listo, y publica actualizaciones de posición. El tracking puede degradar a 99.5 % de disponibilidad sin afectar el flujo crítico de pedidos [Brief §A.3].

### 3.6 Billing & Accounting

Procesa los cobros al consumidor (delegando datos de tarjeta a Stripe para cumplimiento PCI-DSS), calcula comisiones de FTGO, y gestiona los payouts a restaurantes y couriers. Debe poder encolar reintentos de cobro si la pasarela de pago está temporalmente caída [Brief §A.3].

### 3.7 Notifications

Envía confirmaciones, alertas de estado y recibos por email (SendGrid), SMS y push (Twilio). Consume eventos de los demás servicios y garantiza entrega al menos una vez. Puede operar con eventual consistency respecto al estado del pedido sin afectar la experiencia crítica del flujo de compra [Brief §A.3].

---

## 4. Requisitos no funcionales

### NFR-01: Latencia UX
- **Métrica:** Tiempo de respuesta ≤ 200 ms p95 para acciones del consumidor en la app (toma de pedido, confirmación, consulta de estado).
- **Origen:** [Brief §A.4 Latencia UX]
- **Justificación:** La experiencia móvil en horarios pico determina la retención del consumidor; superar el umbral percibido como lento aumenta el abandono del carrito.

### NFR-02: Disponibilidad del flujo de pedidos
- **Métrica:** 99.9 % de disponibilidad mensual para el flujo Order Taking; el servicio de tracking puede degradar a 99.5 % mensual.
- **Origen:** [Brief §A.4 Disponibilidad]
- **Justificación:** La toma de pedidos es el flujo de ingresos central; su caída impacta directamente la facturación de FTGO, restaurantes y couriers.

### NFR-03: Escalabilidad horizontal ante tráfico pico
- **Métrica:** El sistema debe soportar hasta 5× el tráfico base durante franjas de almuerzo (12:00–14:00) y cena (19:00–22:00) hora local, escalando cada componente de forma independiente (X-axis y Y-axis del Scale Cube).
- **Origen:** [Brief §A.4 Carga] [Brief §A.4 Escalabilidad horizontal]
- **Justificación:** El tráfico de FTGO es altamente predecible y concentrado; escalar el monolito entero para absorber el pico de un solo módulo es ineficiente y costoso.

### NFR-04: Tolerancia a fallos externos
- **Métrica:** El sistema debe poder aceptar y encolar pedidos aunque la pasarela de pago (Stripe) esté caída; degradación de mapas (Google Maps) aceptable sin bloquear la toma de pedidos. Tiempo de retry configurable; 0 pérdidas de pedidos por fallo externo.
- **Origen:** [Brief §A.4 Tolerancia a fallos externos]
- **Justificación:** Las dependencias externas tienen SLAs menores al 100 %; el negocio no puede perder una orden por un fallo transitorio de un proveedor.

### NFR-05: Consistencia de datos
- **Métrica:** Consistencia fuerte requerida dentro del aggregate de un pedido (Order Taking / Fulfillment); eventual consistency aceptada para reporting y notificaciones, con convergencia en ≤ 5 s en condiciones normales.
- **Origen:** [Brief §A.4 Consistencia de datos]
- **Justificación:** La integridad del pedido es un invariante de negocio; los reportes financieros pueden tolerar un breve retraso sin afectar la operación.

### NFR-06: Trazabilidad end-to-end
- **Métrica:** 100 % de las acciones del consumidor deben llevar correlation ID propagado a todos los servicios; distributed tracing habilitado con retención mínima de 7 días.
- **Origen:** [Brief §A.4 Trazabilidad]
- **Justificación:** Sin trazabilidad distribuida, la resolución de incidentes en una arquitectura de microservicios es exponencialmente más costosa.

### NFR-07: Migración incremental sin downtime
- **Métrica:** El monolito legacy permanece operativo durante todo el período de migración (18–24 meses); cada servicio extraído debe poder activarse y revertirse con feature flag sin afectar la disponibilidad del flujo principal.
- **Origen:** [Brief §A.4 Migración incremental]
- **Justificación:** Una migración big-bang es inaceptable para FTGO; el patrón Strangler Fig requiere coexistencia controlada entre el monolito y los nuevos servicios.

---

## 5. Alcance

### In scope
- Documentar la arquitectura objetivo de microservicios para las 7 capacidades de negocio identificadas en el Cap 2 del libro Richardson.
- Definir bounded contexts candidatos y sus interacciones como entrada para ADR-01 (descomposición de servicios).
- Establecer la estrategia de migración incremental mediante patrón Strangler Fig durante un horizonte de 18–24 meses.
- Especificar NFRs medibles y trazables al brief como contrato de calidad para el FSD y los ADRs.
- Identificar integraciones con sistemas externos (Stripe, Google Maps, SendGrid/Twilio) y sus restricciones de cumplimiento (PCI-DSS, GDPR).

### Out of scope
- Implementación o refactor del monolito legacy Java/WAR en este entregable.
- Diseño detallado de APIs, esquemas de base de datos o contratos de eventos entre servicios (corresponde al FSD).
- Definición de SLOs operativos finales, runbooks y procedimientos de on-call en producción.
- Selección de plataforma de infraestructura (Kubernetes, cloud provider, service mesh) — decisión diferida a ADR-02.
- Cobertura exhaustiva de todos los flujos de negocio; este PRD es ligero y cubre lo esencial para derivar FSD y ADRs.

---

## Verification

| # | Criterio | Cumple |
| :---: | :--- | :---: |
| V1 | Existen las 5 secciones H2 del Output (§1–§5) | S |
| V2 | Tabla Stakeholders con **6** filas y origen `[Brief §A.2]` en cada una | S |
| V3 | Capacidades **3.1–3.7** presentes (7 H3, nombres del brief §A.3) | S |
| V4 | **≥ 5** NFRs; cada uno con **Métrica**, **Origen** `[Brief §A.4 …]` y **Justificación** | S — 7 NFRs |
| V5 | **0** NFRs sin tag `[Brief §A.4` (trazabilidad 100 %) | S |
| V6 | Alcance: **≥ 4** bullets *In scope* (incluye Strangler Fig / migración incremental) | S — 5 bullets |
| V7 | Alcance: **≥ 4** bullets *Out of scope* (incluye monolito legacy o implementación) | S — 5 bullets |
| V8 | No hay stakeholders, capacidades ni NFRs que no estén en `brief.md` | S |
