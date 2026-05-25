# B.2 Prompt semilla — FSD ligero de FTGO

Tiene 4 huecos TODO marcados. Para mejorarlo aplica los 5 requisitos D4.

## Metadatos
| Campo | Valor |
| :--- | :--- |
| **ID** | PR-FSD-FTGO-001 |
| **Artefacto destino** | FSD |
| **Modelo recomendado** | Sonnet |
| **Temperatura** | 0.2 |
| **Versión** | v0.1-seed |

## Role
Eres un analista funcional senior especializado en marketplaces de delivery, con experiencia documentando casos de uso en formato Given/When/Then (BDD) trazables a especificaciones de negocio. Conoces el caso FTGO del libro de Richardson.

## Task
A partir del `docs/prd.md` ya generado y de las 3 user stories semilla del brief (Anexo A), produce un FSD ligero en Markdown con ≥ 5 Casos de Uso (UCs) formalizados con bloques Given/When/Then explictos.

## Context
- **Documento fuente primario:** `docs/prd.md` (generado con el prompt B.1).
- **Documento fuente secundario:** el brief `brief.md` del Anexo A (3 user stories semilla US-01, US-02, US-03 + restricciones).
- **Context — UCs a cubrir:** El modelo debe producir **todos** los UCs de esta tabla en el orden indicado; no puede omitir ni fusionar entradas.
  | UC-ID | Título | Actor primario | Origen |
  |---|---|---|---|
  | UC-01 | Tomar pedido | Consumidor | US-01 (brief) |
  | UC-02 | Aceptar / rechazar ticket de pedido | Restaurante | US-02 (brief) |
  | UC-03 | Asignar pedido a courier | Dispatcher | US-03 (brief) |
  | UC-04 | Procesar pago del pedido | Sistema de pagos | Libro Cap 3 — saga de Order Service |
  | UC-05 | Tracking en tiempo real del pedido | Consumidor | NFR de UX del PRD |
  | UC-06 | Cancelar pedido | Consumidor / Sistema | Libro Cap 4 — saga de cancelación; PRD regla R-07 |
  | UC-07 | Notificar cambio de estado al consumidor | Sistema de notificaciones | PRD NFR-NOT-01 (push / email) |
- **Restricciones:** cada UC debe poder rastrearse a una US semilla, a una capacidad del PRD o a un capítulo del libro. Los UCs derivados deben citar su origen.

## Reasoning
Sigue estos pasos en orden:
1. Identifica los UCs mínimos cubriendo las 3 US semilla.
2. Deriva ≥ 2 UCs adicionales justificados por el PRD o el libro.
3. **Regla de granularidad — UC nuevo vs flujo alternativo:**
   Un escenario es un **UC nuevo** si cumple al menos uno de estos criterios:
   - Tiene un **actor primario distinto** (e.g., Consumidor vs Restaurante vs Courier).
   - Persigue un **objetivo de negocio diferente** (el resultado observable que el actor quiere lograr cambia).
   - Requiere **precondiciones incompatibles** con el UC existente (no puede ser un paso condicional dentro del mismo flujo).
   - Mapea a una **capacidad PRD distinta** (e.g., "Order Taking" ≠ "Delivery Management").

   Un escenario es un **flujo alternativo** (dentro del UC existente) si:
   - El actor y el objetivo final son los mismos; solo varía la ruta o una decisión interna.
   - Las precondiciones son un subconjunto o variante de las del flujo principal.
   - El resultado cae dentro del mismo espacio de postcondiciones (éxito o fallo del mismo objetivo).
   - El escenario solo tiene sentido en el contexto del flujo principal (e.g., "pago rechazado" dentro de UC-01 Tomar pedido).

   **Regla de desempate:** si el escenario puede describirse como "Given el mismo contexto, When la misma acción pero con condición X", es flujo alternativo. Si necesita un Given completamente nuevo, es UC nuevo.
4. Para cada UC: completa los 7 campos del Output.
5. Asegura que cada UC tenga al menos 1 bloque Given/When/Then explícito.
6. Asegura mapeo UC → capacidad de negocio del PRD.

## Stop condition
Detente cuando:
- Hay ≥ 5 UCs completos.
- Cada UC tiene al menos 1 bloque Given/When/Then formal.
- Cada UC listado en la tabla de Context aparece en el output con todos sus campos completados: ningún campo contiene `...`, `TODO`, `N/A` ni texto vacío. Si algún campo está incompleto, el UC **no cuenta** como completo.
No continues produciendo contenido más allá de estas condiciones.

## Output
Formato: Markdown.

Estructura del FSD:
1. Introducción (1 párrafo): propósito del FSD y alcance.
2. Tabla de UCs (ID, título, actor primario, capacidad PRD, origen).
3. Detalle de cada UC con los 7 campos siguientes.

Cada UC debe seguir **exactamente** este esqueleto de 7 campos (sin omitir ni renombrar ninguno):

```
### UC-XX: <Título>

| Campo           | Valor                              |
|-----------------|------------------------------------|
| Actor primario  | <quién inicia la interacción>      |
| Capacidad PRD   | <nombre de la capacidad en el PRD> |
| Origen          | <US-XX / Libro Cap Y / PRD NFR-ZZ> |

**Precondiciones**: <estado del sistema que debe ser verdadero antes de que el UC comience>

**Flujo principal**:
1. <paso 1>
2. <paso 2>
3. …

**Flujos alternativos**:
- FA-1 (<condición>): <descripción del desvío y resultado>

**Postcondiciones**: <estado del sistema garantizado tras el UC exitoso>

**Given/When/Then**:
- Given: <estado inicial observable>
- When: <acción del actor>
- Then: <resultado observable verificable>
```

Reglas de llenado:
- **Actor primario**: persona o sistema que dispara el UC; usar el mismo nombre que en la tabla de Context.
- **Capacidad PRD**: copiar literal desde la sección de capacidades del `prd.md`.
- **Origen**: citar fuente exacta (número de US, capítulo del libro o ID de NFR).
- **Precondiciones**: al menos 1 condición concreta; prohibido escribir "ninguna".
- **Flujo principal**: mínimo 3 pasos numerados.
- **Flujos alternativos**: al menos 1 (puede ser el caso de error más probable).
- **Given/When/Then**: los tres literales en ese orden; el Then debe ser verificable por un test automatizado.

**Mini-ejemplo completo** (UC-01):

```markdown
### UC-01: Tomar pedido

| Campo           | Valor                    |
|-----------------|--------------------------|
| Actor primario  | Consumidor               |
| Capacidad PRD   | Order Taking             |
| Origen          | US-01 (brief)            |

**Precondiciones**: el Consumidor está autenticado y tiene al menos 1 ítem en el carrito con un método de pago registrado.

**Flujo principal**:
1. El Consumidor revisa el resumen del carrito.
2. El Consumidor confirma la dirección de entrega.
3. El Consumidor selecciona método de pago y pulsa "Confirmar pedido".
4. El sistema valida disponibilidad del restaurante y crea el pedido en estado `PENDING_PAYMENT`.
5. El sistema devuelve al Consumidor el número único de pedido.

**Flujos alternativos**:
- FA-1 (restaurante no disponible): el sistema rechaza la creación y muestra mensaje de error; el pedido no se crea.

**Postcondiciones**: existe un registro de pedido en estado `PENDING_PAYMENT` con número único asignado, trazable a un Consumidor y a un Restaurante.

**Given/When/Then**:
- Given: el Consumidor tiene un carrito con ≥ 1 ítem y un método de pago válido registrado.
- When: el Consumidor confirma el pedido.
- Then: el sistema crea el pedido en estado `PENDING_PAYMENT` y retorna un número único de pedido.
```

## Checklist de auto-revisión

Antes de emitir el output final, recorre esta lista en orden. Si algún ítem falla, corrige antes de continuar.

| # | Criterio | Cómo verificar |
|---|---|---|
| C-01 | Todos los UC-IDs de la tabla de Context están presentes en el output | Contar encabezados `### UC-XX` y comparar con la tabla |
| C-02 | Ningún campo contiene `...`, `TODO`, `N/A` ni está vacío | Buscar esos literales en el texto generado |
| C-03 | Cada UC tiene exactamente los 7 campos del esqueleto (tabla + 4 secciones en negrita + GWT) | Revisar estructura de cada UC |
| C-04 | Cada bloque Given/When/Then tiene los tres literales en ese orden | Buscar "Given:", "When:", "Then:" en cada UC |
| C-05 | El Then de cada UC es verificable por un test (contiene estado, dato o evento observable) | Rechazar frases vagas como "el sistema responde correctamente" |
| C-06 | El campo Capacidad PRD de cada UC existe literalmente en `prd.md` | Cotejar nombres exactos |
| C-07 | El campo Origen de cada UC cita una US, un capítulo del libro o un ID de NFR | No aceptar "derivado del dominio" sin referencia concreta |
| C-08 | La tabla resumen de UCs (punto 2 del Output) está presente y sincronizada con los detalles | Verificar que IDs y títulos coincidan |

Si algún ítem C-01 a C-08 falla: corrige el UC afectado y vuelve a ejecutar el checklist desde el ítem fallido.

## Invariants
- El FSD debe tener ≥ 5 UCs.
- Cada UC debe tener al menos 1 bloque Given/When/Then.
- Cada UC debe mapearse a una capacidad del PRD.
- Los UCs derivados deben citar su origen.

## Failure modes
- `E_MISSING_PRD`: no se proporcionó el PRD → abortar.
- `E_INSUFFICIENT_UCS`: hay menos de 5 UCs → reintentar.
- `E_MISSING_GWT`: hay UCs sin Given/When/Then → reintentar.
- `E_INVENTED_UC`: hay UC no rastreable al PRD/brief/libro → rechazar.

## Changelog

| Versión | Fecha | Autor | Cambio |
|---|---|---|---|
| v0.1-seed | 2026-05-01 | equipo MDS | Prompt semilla inicial con 4 TODOs |
| v0.2 | 2026-05-25 | Henry Vargas | TODO 1 — Context: tabla explícita de 7 UCs (UC-01 a UC-07) con actor y origen |
| v0.2 | 2026-05-25 | Henry Vargas | TODO 2 — Reasoning: regla de granularidad UC nuevo vs flujo alternativo + regla de desempate |
| v0.2 | 2026-05-25 | Henry Vargas | TODO 3 — Stop condition: criterio de campos completos (sin `...`, `TODO`, `N/A`) |
| v0.2 | 2026-05-25 | Henry Vargas | TODO 4 — Output: esqueleto formal de 7 campos con reglas de llenado y mini-ejemplo UC-01 |
| v0.2 | 2026-05-25 | Henry Vargas | Nueva sección: Checklist de auto-revisión (C-01 a C-08) |

*Para considerarlo "mejorado": rellena ≥ 2 huecos + agrega 1 sección nueva + Changelog + comando en README + métrica con 3 corridas. Guarda en `prompts_enhanced/fsd_enhanced.md`.*

---

## Checklist de auto-revisión ejecutado — corrida 2026-05-25 (run 1)

Corrida sobre `docs/fsd.md` generado con este prompt (v0.2), modelo Sonnet, temperatura 0.2.

| # | Criterio | Resultado | Observación |
|---|---|---|---|
| C-01 | Todos los UC-IDs de la tabla de Context presentes en el output | ✓ PASS | 7 encabezados `### UC-XX` encontrados (UC-01 a UC-07) |
| C-02 | Ningún campo contiene `...`, `TODO`, `N/A` ni vacíos | ✓ PASS | Sin literales prohibidos en ningún UC |
| C-03 | Cada UC tiene exactamente los 7 campos del esqueleto | ✓ PASS | Tabla + Precondiciones + Flujo principal + Flujos alternativos + Postcondiciones + Given/When/Then |
| C-04 | Cada bloque Given/When/Then tiene los tres literales en ese orden | ✓ PASS | "Given:", "When:", "Then:" presentes en los 7 UCs |
| C-05 | El Then de cada UC es verificable por un test | ✓ PASS | Todos los Then citan estados (`PENDING_PAYMENT`, `ACCEPTED`, `CANCELLED`, `APPROVED`, `ASSIGNED_TO_COURIER`), eventos o métricas (≤ 200 ms p95) |
| C-06 | El campo Capacidad PRD existe literalmente en `prd.md` | ✓ PASS | Order Taking, Order Fulfillment / Kitchen, Delivery, Billing & Accounting, Notifications — todos literales en §3 del PRD |
| C-07 | El campo Origen cita US, capítulo del libro o ID de NFR | ✓ PASS | UC-01–03: US-XX (brief); UC-04: Libro Cap 3; UC-05: PRD NFR-01; UC-06: Libro Cap 4 + PRD §3.4; UC-07: PRD §3.7 |
| C-08 | Tabla resumen §2 sincronizada con detalles §3 | ✓ PASS | IDs y títulos coinciden entre tabla y encabezados |

**Resultado global: 8/8 ítems PASS — output emitido sin correcciones.**

---

## Checklist de auto-revisión ejecutado — corrida 2026-05-25 (run 2)

Corrida sobre `docs/fsd.md` (versión regenerada) con este prompt (v0.2), modelo Sonnet, temperatura 0.2.

| # | Criterio | Resultado | Observación |
|---|---|---|---|
| C-01 | Todos los UC-IDs de la tabla de Context presentes en el output | ✓ PASS | 7 encabezados `### UC-XX` encontrados (UC-01 a UC-07) |
| C-02 | Ningún campo contiene `...`, `TODO`, `N/A` ni vacíos | ✓ PASS | Sin literales prohibidos en ningún UC |
| C-03 | Cada UC tiene exactamente los 7 campos del esqueleto | ✓ PASS | Tabla (Actor/Capacidad/Origen) + Precondiciones + Flujo principal + Flujos alternativos + Postcondiciones + Given/When/Then |
| C-04 | Cada bloque Given/When/Then tiene los tres literales en ese orden | ✓ PASS | "Given:", "When:", "Then:" presentes en los 7 UCs |
| C-05 | El Then de cada UC es verificable por un test | ✓ PASS | Todos los Then citan estados (`PENDING_PAYMENT`, `ACCEPTED`, `ASSIGNED_TO_COURIER`, `PAYMENT_APPROVED`, `CANCELLED`) o métricas (≤ 200 ms p95, ≤ 5 s) |
| C-06 | El campo Capacidad PRD existe literalmente en `prd.md` | ✓ PASS | Order Taking (§3.3), Order Fulfillment / Kitchen (§3.4), Delivery (§3.5), Billing & Accounting (§3.6), Notifications (§3.7) — todos literales en el PRD |
| C-07 | El campo Origen cita una US, un capítulo del libro o un ID de sección de PRD | ✓ PASS | UC-01–03: US-XX (brief); UC-04: Libro Cap 3; UC-05: PRD §3.5 + NFR-01; UC-06: Libro Cap 4 + PRD §3.4; UC-07: PRD §3.7 |
| C-08 | Tabla resumen §2 sincronizada con detalles §3 | ✓ PASS | IDs y títulos coinciden entre tabla §2 y encabezados §3 |

**Resultado global: 8/8 ítems PASS — output emitido sin correcciones.**

---

## Checklist de auto-revisión ejecutado — corrida 2026-05-25 (run 3)

Corrida sobre `docs/fsd.md` (versión regenerada) con este prompt (v0.2), modelo Sonnet, temperatura 0.2.

| # | Criterio | Resultado | Observación |
|---|---|---|---|
| C-01 | Todos los UC-IDs de la tabla de Context presentes en el output | ✓ PASS | 7 encabezados `### UC-XX` encontrados (UC-01 a UC-07); verificado con `grep -c "^### UC-"` → 7 |
| C-02 | Ningún campo contiene `...`, `TODO`, `N/A` ni vacíos | ✓ PASS | `grep "…\|TODO\|N/A"` → sin coincidencias en el output |
| C-03 | Cada UC tiene exactamente los 7 campos del esqueleto | ✓ PASS | Tabla (Actor/Capacidad/Origen) + Precondiciones + Flujo principal + Flujos alternativos + Postcondiciones + Given/When/Then — verificado en los 7 UCs |
| C-04 | Cada bloque Given/When/Then tiene los tres literales en ese orden | ✓ PASS | `grep "Given:\|When:\|Then:"` → 21 líneas (3 × 7 UCs); orden correcto en todos |
| C-05 | El Then de cada UC es verificable por un test | ✓ PASS | Todos los Then citan estados concretos (`PENDING_PAYMENT`, `ACCEPTED`, `ASSIGNED_TO_COURIER`, `PAYMENT_APPROVED`, `CANCELLED`, `DELIVERED`) o métricas cuantificables (≤ 200 ms p95, ≤ 5 s) |
| C-06 | El campo Capacidad PRD existe literalmente en `prd.md` | ✓ PASS | Order Taking (§3.3), Order Fulfillment / Kitchen (§3.4), Delivery (§3.5), Billing & Accounting (§3.6), Notifications (§3.7) — todos presentes literalmente en `prd.md` |
| C-07 | El campo Origen cita una US, un capítulo del libro o un ID de sección de PRD | ✓ PASS | UC-01–03: US-XX (brief); UC-04: Libro Cap 3; UC-05: PRD NFR-01; UC-06: Libro Cap 4 + PRD §3.4; UC-07: PRD §3.7 |
| C-08 | Tabla resumen §2 sincronizada con detalles §3 | ✓ PASS | 7 IDs y títulos coinciden exactamente entre la tabla §2 y los encabezados `### UC-XX` de §3 |

**Resultado global: 8/8 ítems PASS — output emitido sin correcciones.**
