# ADR-0001: Estilo arquitectónico de FTGO

**Status**: Accepted  
**Fecha**: 2026-05-25  
**Deciders**: Equipo de arquitectura (Henry Vargas)  
**Documentos fuente**: `docs/prd.md`, `docs/fsd.md`, `brief.md`, Richardson cap. 1–2

---

## 1. Contexto

FTGO opera desde hace varios años como una aplicación monolítica Java empaquetada como WAR. El monolito ha acumulado los síntomas clásicos del *infierno monolítico* descritos en Richardson cap. 1: builds lentos (decenas de minutos), escalado conflictivo entre módulos (escalar Order Taking obliga a escalar todo el WAR), falta de aislamiento de fallos (un bug en Notifications puede tumbar Order Taking), lock-in tecnológico y un equipo frenado por el tamaño del código base [Brief §A.1].

La dirección decidió migrar a microservicios para sostener el crecimiento del negocio. Esta decisión de **estilo arquitectónico** debe tomarse ahora porque es la restricción fundacional que condiciona todas las decisiones posteriores: estrategia de datos, mecanismo IPC, descomposición de servicios y topología C4. Una vez elegido el estilo, revertirlo tiene un costo prohibitivo.

El problema a resolver es: **¿qué estilo arquitectónico permite a FTGO cumplir sus NFRs (escalabilidad 5×, disponibilidad 99.9 %, latencia ≤ 200 ms p95, migración incremental sin downtime) mientras mantiene el monolito operativo durante 18–24 meses?**

---

## 2. Restricciones que esta decisión debe respetar

| # | Restricción | Origen |
|---|---|---|
| R-01 | Latencia ≤ 200 ms p95 para acciones del consumidor | NFR-01 / Brief §A.4 Latencia UX |
| R-02 | Disponibilidad ≥ 99.9 % mensual en flujo Order Taking | NFR-02 / Brief §A.4 Disponibilidad |
| R-03 | Escalabilidad 5× tráfico base; X-axis e Y-axis del Scale Cube | NFR-03 / Brief §A.4 Escalabilidad horizontal |
| R-04 | Resiliencia ante Stripe caído: encolar pedidos, 0 pérdidas | NFR-04 / Brief §A.4 Tolerancia a fallos externos |
| R-05 | Consistencia fuerte dentro del aggregate del pedido; eventual ≤ 5 s en reporting | NFR-05 / Brief §A.4 Consistencia de datos |
| R-06 | Correlation ID + distributed tracing con retención mínima de 7 días | NFR-06 / Brief §A.4 Trazabilidad |
| R-07 | Monolito legacy operativo 18–24 meses; cada servicio activable con feature flag | NFR-07 / Brief §A.4 Migración incremental |
| R-08 | Java/Spring Boot en servicios core; PCI-DSS delegado a Stripe; GDPR para consumidores | Brief §A.4 Tecnología / Cumplimiento |

---

## 3. Opciones consideradas

### Opción 1: Monolito modular (refactor interno sin distribución)

**Descripción**: Refactorizar el WAR existente en módulos con fronteras explícitas (packages/jars separados, interfaces internas bien definidas), manteniendo un único artefacto desplegable. No hay comunicación de red entre módulos: las llamadas son in-process. El escalado sigue siendo a nivel de todo el proceso, pero la estructura interna mejora la mantenibilidad.

**Pros**:
- Sin overhead de red entre capacidades → preserva NFR-01 (latencia ≤ 200 ms p95) sin esfuerzo adicional.
- Transacciones ACID locales → consistencia fuerte nativa, cubre NFR-05 sin sagas.
- Menor curva operativa: un solo proceso, un solo log, un solo deployment pipeline.
- Reutiliza todo el conocimiento del equipo en Java/Spring Boot (R-08).

**Contras**:
- Escalado sigue siendo de grano grueso: escalar Order Taking (pico 5×) obliga a replicar todo el WAR → no cubre NFR-03 (Scale Cube Y-axis por componente).
- Aislamiento de fallos inexistente en tiempo de ejecución: un crash en Notifications puede tumbar Order Taking → no cubre NFR-02 (99.9 %).
- Lock-in tecnológico persiste: Delivery podría beneficiarse de un runtime diferente, pero no es posible.
- No resuelve los builds lentos ni los conflictos de merge en un equipo grande.
- No es compatible con feature flags por servicio para la migración incremental (NFR-07).

**Impacto en NFRs**: NFR-01 ✓ (in-process, 0 overhead); NFR-02 ✗ (sin aislamiento de fallos entre módulos); NFR-03 ✗ (escalado monolítico, no independiente); NFR-04 parcial (cola de retry posible pero acoplada); NFR-07 ✗ (no hay unidades independientes activables con feature flag).

---

### Opción 2: Microservicios de grano fino (7 servicios, uno por capacidad)

**Descripción**: Descomponer las 7 capacidades de negocio identificadas en Brief §A.3 en 7 microservicios independientes, cada uno con su propia base de datos (Database per Service, Richardson cap. 2). La migración se realiza incrementalmente mediante Strangler Fig: el monolito coexiste con los nuevos servicios durante 18–24 meses, y el tráfico se migra gradualmente por capacidad con feature flags (NFR-07).

**Pros**:
- Escalabilidad independiente por servicio → cubre NFR-03 (Scale Cube X-axis y Y-axis): Order Taking puede escalar 5× sin tocar Notifications.
- Aislamiento de fallos: un crash en Delivery no afecta Order Taking → cubre NFR-02 (99.9 %).
- Compatible con Strangler Fig nativo: cada servicio extraído del monolito es una unidad autónoma activable con feature flag → cubre NFR-07.
- Permite messaging asíncrono (eventos de dominio) para desacoplar Billing de Stripe → cubre NFR-04 (cola de retry sin bloquear el flujo de pedidos).
- Trazado al Cap 2 del libro: *Decompose by Business Capability* es el patrón canónico para FTGO.
- Libertad tecnológica para servicios satélite (R-08): Delivery puede usar un runtime optimizado para geo-queries.

**Contras**:
- 7 microservicios desde el día 1 → complejidad operativa alta: 7 pipelines CI/CD, 7 bases de datos, service discovery, load balancing.
- Latencia de red entre servicios: el flujo UC-01 (Tomar pedido) involucra ≥ 3 saltos sincrónicos (Consumer, Restaurant, Order Taking) → riesgo de superar NFR-01 (200 ms p95) si no se diseña cuidadosamente.
- Consistencia distribuida: las transacciones que cruzan servicios (UC-04 pago + UC-01 pedido) requieren sagas con compensación (Richardson cap. 4) → complejidad de implementación alta.
- Distributed Monolith latente si los servicios se acoplan síncronamente sin disciplina de bounded contexts.
- Inversión inicial en infraestructura de observabilidad para cumplir NFR-06 (distributed tracing).

**Impacto en NFRs**: NFR-01 ⚠ (riesgo por overhead de red; mitigable con async + caching); NFR-02 ✓ (aislamiento de fallos por servicio); NFR-03 ✓ (escalado independiente); NFR-04 ✓ (messaging asíncrono desacopla Stripe); NFR-05 ✓/⚠ (eventual consistency OK para reporting; sagas necesarias para aggregate del pedido); NFR-06 ✓ (con inversión en tracing); NFR-07 ✓ (Strangler Fig nativo).

---

### Opción 3: Macro-services agrupados (3–4 servicios de grano grueso)

**Descripción**: Agrupar las 7 capacidades en 3–4 servicios de mayor granularidad según cohesión funcional: **(a) Order Core** (Order Taking + Order Fulfillment/Kitchen), **(b) Delivery & Notifications** (Delivery + Notifications), **(c) User Services** (Consumer Management + Restaurant Management), **(d) Billing & Accounting** como servicio independiente por cumplimiento PCI-DSS. Cada grupo tiene su propia base de datos pero expone una interfaz de servicio unificada.

**Pros**:
- Menor complejidad operativa que la Opción 2: 4 pipelines CI/CD en lugar de 7.
- Reduce los saltos de red en el flujo crítico UC-01: Order Taking y Fulfillment viven en el mismo proceso → preserva mejor NFR-01.
- Escalado independiente a nivel de grupo → cubre parcialmente NFR-03 (se puede escalar Order Core en pico sin tocar Billing).
- Billing aislado por PCI-DSS (R-08) desde el primer día.
- Más fácil de migrar incrementalmente desde el monolito: 4 extracciones en lugar de 7.

**Contras**:
- Granularidad gruesa sacrifica aislamiento de fallos intra-grupo: un bug en Order Fulfillment puede afectar Order Taking si comparten proceso → riesgo para NFR-02.
- Escalado intra-grupo sigue siendo monolítico: Order Taking y Fulfillment tienen perfiles de carga distintos (pico en Taking durante el pedido, pico en Fulfillment durante preparación) → coexistir en el mismo servicio obliga a sobreprovisionar.
- La agrupación puede perpetuar el acoplamiento original del monolito bajo una nueva capa de abstracción → riesgo de *Distributed Monolith* (anti-patrón del prompt).
- Menos alineado con los bounded contexts canónicos del Cap 2 del libro; la decisión de agrupamiento requiere justificación adicional que no está documentada en el brief.
- Migración futura de macro-service a microservicio individual tiene costo alto.

**Impacto en NFRs**: NFR-01 ✓ intra-grupo, ⚠ inter-grupo; NFR-02 ⚠ (sin aislamiento intra-grupo); NFR-03 ✓ parcial (por grupo, no por capacidad); NFR-04 ✓ (Billing aislado); NFR-07 ✓ (4 extracciones de Strangler Fig).

---

## 4. Decisión

**Se adopta la Opción 2: Microservicios de grano fino (7 servicios, uno por capacidad de negocio), implementados de forma incremental mediante el patrón Strangler Fig durante 18–24 meses.**

### Justificación

1. **Cobertura directa de los NFRs críticos**: la Opción 2 es la única que cubre NFR-02 (99.9 % mediante aislamiento de fallos) y NFR-03 (Scale Cube Y-axis independiente) simultáneamente. La Opción 1 falla en ambos. La Opción 3 cubre NFR-03 solo a nivel de grupo y deja NFR-02 en riesgo dentro de cada macro-service.

2. **Compatibilidad nativa con Strangler Fig**: la Opción 2 permite extraer cada servicio de forma independiente y activarlo con feature flag sin afectar el flujo principal, cumpliendo NFR-07. Esta es la premisa del patrón descrito en Richardson cap. 3 (*Strangler Fig Application*).

3. **Alineación con el patrón canónico del libro**: la descomposición por capacidad de negocio (*Decompose by Business Capability*, Richardson cap. 2) es el patrón recomendado para FTGO y el único documentado en el repositorio oficial [Brief §A.7]. Las 7 capacidades de Brief §A.3 son estables y están bien delimitadas, lo que minimiza el riesgo de refactorización de boundaries posterior.

4. **Resiliencia ante fallos externos (R-04)**: la separación de Billing & Accounting como servicio independiente, con mensajería asíncrona mediante eventos de dominio (`OrderCreated` → `PaymentApproved`), permite desacoplar el flujo de toma de pedidos de la disponibilidad de Stripe. Esto no es implementable limpiamente en la Opción 1.

5. **El riesgo de latencia (NFR-01) es mitigable**: el overhead de red en el flujo crítico UC-01 se controla con (a) comunicación asíncrona donde sea posible, (b) caching de datos de lectura frecuente (menú, disponibilidad de restaurante), y (c) diseño de bounded contexts que minimice los saltos sincrónicos en el path crítico. Este riesgo queda documentado como follow-up (ADR-0002).

---

## 5. Consecuencias

### Positivas

- **Escalabilidad selectiva**: Order Taking puede absorber el pico de 5× (NFR-03) sin necesidad de escalar Billing, Notifications ni Consumer Management, reduciendo el costo de infraestructura en producción.
- **Aislamiento de fallos real**: un fallo en el servicio de Notifications o Delivery no puede cascadear a Order Taking → la disponibilidad de 99.9 % del flujo de pedidos (NFR-02) es alcanzable mediante circuit breakers independientes.
- **Migración sin big-bang**: el Strangler Fig permite activar y revertir cada servicio con feature flag (NFR-07); el equipo puede validar cada extracción en producción con riesgo acotado.
- **Base para decisiones posteriores**: la separación de bounded contexts habilita la elección de mecanismos IPC adecuados por par de servicios (ADR-0002) y la adopción de Database per Service (ADR-0003).
- **Cumplimiento PCI-DSS limpio**: Billing & Accounting como servicio aislado limita el scope de PCI-DSS a un único componente (R-08).

### Negativas

- **Latencia p95 en riesgo durante el flujo UC-01**: el path Order Taking → Restaurant Management → Consumer Management implica ≥ 2 llamadas sincrónicas adicionales respecto al monolito. Si no se diseña con caching y bounded contexts mínimos, el presupuesto de latencia de 200 ms p95 (NFR-01) puede agotarse en condiciones de pico. **Acción de mitigación**: definir en ADR-0002 qué llamadas son sincrónicas vs. asíncronas en el flujo crítico.
- **Consistencia distribuida obligatoria**: las transacciones que cruzan servicios (UC-01 + UC-04: toma de pedido + pago) requieren implementar sagas con compensación (Richardson cap. 4). La complejidad de implementación es significativamente mayor que una transacción ACID local. **Acción de mitigación**: documentar el patrón de saga en el FSD (ya iniciado en UC-04 y UC-06).
- **Inversión operativa desde el día 1**: distributed tracing (NFR-06), service discovery, y 7 pipelines CI/CD son prerequisitos antes de migrar el primer servicio. El equipo debe aprovisionar esta infraestructura como parte del sprint 0 de migración.
- **Riesgo de Distributed Monolith**: si los servicios se acoplan síncronamente sin disciplina de bounded contexts (ej. Order Taking llama síncronamente a todos los demás en cada pedido), se replica el acoplamiento del monolito pero con los costos de la distribución. **Mitigación**: los anti-patrones del prompt (`prompts_enhanced/adr_enhanced.md`) deben aplicarse en code reviews.

### Follow-ups

| ADR | Decisión pendiente | Dependencia |
|---|---|---|
| ADR-0002 | Mecanismo IPC predominante (REST síncrono vs. messaging asíncrono) | Define cómo se mitiga el riesgo de NFR-01 en el flujo UC-01 |
| ADR-0003 | Estrategia de datos (Database per Service, Shared Database, CQRS) | Define cómo se garantiza NFR-05 en transacciones distribuidas |
| POC-01 | Validar latencia p95 del flujo UC-01 con ≥ 2 saltos de red sincrónicos en carga 5× | Antes de migrar Order Taking a producción |

---

## Notas de implementación

- El orden de extracción recomendado para el Strangler Fig es: **Notifications → Billing → Consumer Management → Restaurant Management → Delivery → Order Fulfillment → Order Taking**. Se extraen primero los servicios de menor criticidad para ganar experiencia operativa antes de tocar el flujo transaccional central.
- Cada extracción debe incluir: feature flag, rollback documentado, y smoke test de NFR-02 (disponibilidad) antes de promocionar a producción.
