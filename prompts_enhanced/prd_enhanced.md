# B.1 Prompt semilla — PRD ligero de FTGO

Tiene 4 huecos TODO marcados. Para mejorarlo aplica los 5 requisitos D4 de la sección "Requisitos para los prompts mejorados".

## Metadatos
| Campo | Valor |
| :--- | :--- |
| **ID** | PR-PRD-FTGO-001 |
| **Artefacto destino** | PRD |
| **Modelo recomendado** | Sonnet / Opus |
| **Temperatura** | 0.2 |
| **Versión** | v1.0-enhanced |

## Role
Eres un arquitecto de software senior con 10+ años en marketplaces de delivery. Conoces a profundidad el caso FTGO del libro *Microservices Patterns* de Chris Richardson (Manning, 2019) y los patrones DDD estratégico (Bounded Context, Subdomain).

## Task
Genera un PRD ligero para FTGO en formato Markdown, partiendo del brief `brief.md` del Anexo A de este documento. El PRD debe tener entre 2 y 4 páginas equivalentes y servir como entrada para un FSD y para 2 ADRs arquitectónicos.

## Context
- **Documento fuente principal:** el brief del Anexo A `brief.md` (contexto de negocio, stakeholders, capacidades, NFRs, 3 user stories semilla).
- **Documento fuente secundario:** el PDF *Microservices Patterns* `MicroservicesPatterns.pdf` (caps 1-2 obligatorios).
- **Context — Stakeholders:** Consumidor, Restaurante, Courier, Empleado FTGO, Equipo de arquitectura y Sistemas externos.
- **Context — Capacidades:** Consumer Management, Restaurant Management, Order Taking, Order Fulfillment, Delivery, Billing & Accounting, Notifications.
- **Restricciones de dominio:** no inventar stakeholders, capacidades o NFRs fuera del brief. Cada NFR del PRD debe poder rastrearse a una entrada de la sección A.4 del brief `brief.md`.
- **Restricción del laboratorio:** PRD ligero. No exhaustivo. Cubre lo esencial para que el FSD y los ADRs puedan derivarse.

## Reasoning
Sigue estos pasos en orden:
1. Lee el brief y extrae stakeholders, capacidades y restricciones.
2. Estructura el PRD con secciones estándar (ver Output).
3. Asegura trazabilidad explícita de cada NFR al brief (formato `[Brief §A.4]`).
4. Identifica y declara el alcance del laboratorio: qué entra, qué queda fuera (Strangler Fig, monolito legacy, etc.).
5. NO incluyas el razonamiento interno en el output final.

## Stop condition
Detente cuando:
- El PRD tenga las 5 secciones obligatorias declaradas en Output.
- Cada NFR cite su origen en el brief.
- Detente cuando el PRD cumpla **todo** lo siguiente (cuenta explícita):
| Métrica | Mínimo / rango | Verificación |
| :--- | :--- | :--- |
| Stakeholders listados | ≥ 6 | Uno por fila de [Brief §A.2] |
| Capacidades documentadas | = 7 | Una subsección H3 por capacidad [Brief §A.3] |
| Trazabilidad NFR | 100% | 0 NFRs sin tag `[Brief §A.4` |
| Alcance — In scope | ≥ 4 bullets | Incluye migración Strangler Fig |
| Alcance — Out of scope | ≥ 4 bullets | Incluye monolito legacy / implementación |
No continues produciendo contenido más allá de estas condiciones.

## Output
Formato: Markdown.

Secciones obligatorias del PRD:
1. Contexto y objetivos (1-2 párrafos).
2. Stakeholders (lista con rol y necesidad principal).
3. Capacidades de negocio (las 7 del Cap 2, cada una con 1 párrafo).
4. Requisitos no funcionales (≥ 5, cada uno con métrica y origen `[Brief §A.4]`).
5. Alcance (qué entra, qué queda fuera).

**Estructura mínima detallada:** Sigue exactamente este esqueleto (rellena contenido; no omitas encabezados ni campos):

# PRD ligero — FTGO
## 1. Contexto y objetivos
[Párrafo 1 — 80–120 palabras: negocio FTGO, monolito, decisión de migrar a microservicios. Cita implícita al brief §A.1.]
[Párrafo 2 — 60–100 palabras: objetivo del PRD como entrada para FSD y 2 ADRs; alcance del laboratorio.]

## 2. Stakeholders
| Rol | Necesidad principal | Origen |
| :--- | :--- | :--- |
| Consumidor | UX rápida, estado del pedido, tracking | [Brief §A.2] |
| Restaurante | Tickets, carga de cocina, dashboard | [Brief §A.2] |
| … | … | [Brief §A.2] |
*(6 filas obligatorias; sin roles inventados.)*
## 3. Capacidades de negocio
### 3.1 Consumer Management
[1 párrafo — 40–80 palabras: responsabilidad según brief §A.3; sin decidir aún microservicio vs módulo.]
### 3.2 Restaurant Management
[1 párrafo — 40–80 palabras.]
### 3.3 Order Taking
…
### 3.7 Notifications
[1 párrafo — 40–80 palabras.]
*(7 subsecciones H3 obligatorias, numeradas 3.1–3.7.)*
## 4. Requisitos no funcionales
### NFR-01: <nombre corto del brief>
- **Métrica:** <valor medible, ej. ≤ 200 ms p95>
- **Origen:** [Brief §A.4 <categoría exacta>]
- **Justificación:** <1 línea ligada al negocio FTGO>
### NFR-02: …
*(≥ 5 bloques NFR-NN; cada Métrica debe ser cuantificable o binaria verificable.)*
## 5. Alcance
### In scope
- Documentar arquitectura objetivo para migración incremental (Strangler Fig, 18–24 meses).
- …
*(≥ 4 bullets)*
### Out of scope
- Implementación o refactor del monolito legacy en este entregable.
- …
*(≥ 4 bullets)*

**Mini-ejemplos por sección (referencia de tono y extensión):**
| Sección | Ejemplo (1–3 líneas) |
| :--- | :--- |
| 1. Contexto | FTGO opera como monolito Java/WAR con síntomas de infierno monolítico. Este PRD define capacidades y NFRs para derivar FSD y ADRs de la arquitectura objetivo. |
| 2. Stakeholder | `### Rol Courier` Asignaciones cercanas, rutas, pago confiable [Brief §A.2] |
| 3. Capacidad | `### 3.5 Delivery` — Asignación de couriers, rutas y tracking en tiempo real según el flujo de fulfillment del libro (Cap 2). |
| 4. NFR | `### NFR-01: Latencia UX` — **Métrica:** ≤ 200 ms p95 en acciones del consumidor. **Origen:** [Brief §A.4 Latencia UX]. **Justificación:** experiencia móvil en horarios pico. |
| 5. Alcance | In scope: definir bounded contexts candidatos para ADR-01. Out of scope: despliegue productivo y SLOs operativos finales. |

*Sin un esqueleto explícito el output tiende a ser inconsistente entre corridas.*

## Verification
Antes de entregar el PRD, valida cada ítem. Si alguno falla, corrige el documento y vuelve a verificar; no declares el PRD completo.
| # | Criterio | Cumple (S/N) |
| :---: | :--- | :---: |
| V1 | Existen las 5 secciones H2 del Output (§1–§5) | |
| V2 | Tabla Stakeholders con **6** filas y origen `[Brief §A.2]` en cada una | |
| V3 | Capacidades **3.1–3.7** presentes (7 H3, nombres del brief §A.3) | |
| V4 | **≥ 5** NFRs; cada uno con **Métrica**, **Origen** `[Brief §A.4 …]` y **Justificación** | |
| V5 | **0** NFRs sin tag `[Brief §A.4` (trazabilidad 100 %) | |
| V6 | Alcance: **≥ 4** bullets *In scope* (incluye Strangler Fig / migración incremental) | |
| V7 | Alcance: **≥ 4** bullets *Out of scope* (incluye monolito legacy o implementación) | |
| V8 | No hay stakeholders, capacidades ni NFRs que no estén en `brief.md` | |
**Regla:** el PRD solo se considera válido si **V1–V8 = S**. Si falla V4, V5 o V8, trata como `E_INCOMPLETE_NFR` o `E_INVENTED_DOMAIN` y reintenta.

## Invariants
- El PRD debe citar al brief en cada NFR.
- El PRD debe cubrir las 7 capacidades de negocio del Cap 2.
- El PRD no debe inventar stakeholders fuera del brief.
- El PRD no debe exceder 4 páginas equivalentes.

## Failure modes
- `E_MISSING_BRIEF`: no se proporcionó el brief → abortar.
- `E_INVENTED_DOMAIN`: el output contiene stakeholders o capacidades fuera del brief → rechazar.
- `E_INCOMPLETE_NFR`: hay NFRs sin métrica o sin origen → reintentar.

*Para considerarlo "mejorado": rellena ≥ 2 huecos + agrega 1 sección nueva + Changelog + comando en README + métrica con 3 corridas. Guarda en `prompts_enhanced/prd_enhanced.md`.*

## Changelog
**Base:** `prompts_seed/1_PRD-Lightweight.md` (v0.1-seed, ID PR-PRD-FTGO-001)  
**Versión de este prompt:** v1.0-enhanced  
**Autor / laboratorio:** Henry Vargas — Examen FTGO individual  
### Resumen
Se rellenaron los cuatro huecos TODO del Anexo B y se añadió la sección **Verification** exigida por D4. El objetivo es que el PRD generado sea estructuralmente completo, trazable al `brief.md` y comparable entre corridas (entrada para FSD y ADRs).
---
### Cambios por hueco TODO
| Hueco | Qué se cambió | Por qué |
| :--- | :--- | :--- |
| **TODO 1 — Context (stakeholders)** | Lista fija de 6 roles del brief §A.2 (Consumidor, Restaurante, Courier, Empleado FTGO, Equipo de arquitectura, Sistemas externos). | Evita que el modelo omita o invente actores al no re-derivarlos del brief en cada corrida. |
| **TODO 2 — Context (capacidades)** | Lista de las 7 capacidades del Cap 2 (Consumer Management … Notifications). | Fija el alcance funcional del PRD y alinea con los invariantes y el FSD posterior. |
| **TODO 3 — Stop condition** | Tabla con mínimos numéricos: ≥6 stakeholders, =7 capacidades H3, 100% NFRs con `[Brief §A.4]`, ≥4 bullets in/out de alcance (Strangler Fig y monolito legacy). | Sustituye el criterio cualitativo vago (“cuando esté listo”) por conteos verificables antes de detener la generación. |
| **TODO 4 — Output** | Esqueleto Markdown completo (§1–§5) con rangos de extensión, tabla de stakeholders de ejemplo, numeración 3.1–3.7 y formato NFR (Métrica / Origen / Justificación); tabla de mini-ejemplos por sección. | Reduce variación de formato entre corridas; el semilla solo tenía un ejemplo parcial de un NFR. |
---
### Sección nueva (requisito D4)
| Sección | Qué incluye | Por qué |
| :--- | :--- | :--- |
| **Verification** | Checklist V1–V8 (secciones H2, 6 stakeholders, 7 capacidades, ≥5 NFRs, trazabilidad, alcance, anti-invención de dominio). Regla 8/8 = S y vínculo a `E_INCOMPLETE_NFR` / `E_INVENTED_DOMAIN`. | Cierra el ciclo Stop condition → validación explícita; base para la sección **Métrica** (completitud V1–V8 en 3 corridas antes/después). |
---
### Otros ajustes menores
- **Context:** referencia explícita a `brief.md` en restricciones de dominio (además del “Anexo A”).
- **Task:** mismo alcance 2–4 páginas; el esqueleto acota longitud por párrafo sin contradecir el invariante de ≤4 páginas.
---
### Pendiente (fuera de este archivo, entrega del lab)
- [ ] Sección **## Métrica** con 3 corridas semilla vs 3 corridas con este prompt (indicador sugerido: ítems V1–V8 cumplidos, máx. 8/8).
- [ ] **README.md** con comando invocable (ej. `@prompts_enhanced/prd_enhanced.md` + `brief.md`).
- [ ] Copia formal de entrega en `prompts_enhanced/prd_enhanced.md` (ruta exigida en `intro.md`).
- [ ] Actualizar **Metadatos** a `v1.0-enhanced` y quitar el párrafo introductorio de “4 huecos TODO” en la copia de entrega.
---
### Procedencia
Prompt elaborado a partir del material del examen-laboratorio FTGO (Anexo B). [Opcional: si usaste material del producto grupal, indica qué tomaste y qué modificaste para cumplir antiplagio del `intro.md`.]

### Métrica de tres corridas
Check	Resultado
V1 — 5 secciones H2	S
V2 — 6 stakeholders con [Brief §A.2]	S
V3 — Capacidades 3.1–3.7	S
V4 — ≥5 NFRs con Métrica/Origen/Justificación	S (6 NFRs)
V5 — 0 NFRs sin tag [Brief §A.4	S
V6 — ≥4 bullets In scope (incluye Strangler Fig)	S (5 bullets)
V7 — ≥4 bullets Out of scope (incluye monolito legacy)	S (5 bullets)
V8 — Sin dominio inventado fuera del brief	S
V1–V8 = 8/8. Este es tu corrida 1 para la tabla de métricas. Puedes ejecutar el mismo comando dos veces más en chats nuevos para completar las 3 corridas requeridas.
---
#	Criterio	Resultado
V1	≥ 5 secciones H2 (§1–§5)	S — 6 (incluye Verification)
V2	6 filas de stakeholders con [Brief §A.2]	S — 7 marcas
V3	Capacidades 3.1–3.7 (7 H3)	S — 7
V4	≥ 5 NFRs con Métrica / Origen / Justificación	S — 7 NFRs
V5	0 NFRs sin tag [Brief §A.4	S — vacío
V6	≥ 4 bullets In scope (incluye Strangler Fig)	S — 5
V7	≥ 4 bullets Out of scope (incluye monolito legacy)	S — 6
V8	Sin stakeholders/capacidades/NFRs inventados	S
V1–V8 = S. PRD válido. El documento cubre los 7 bounded contexts (§3.1–3.7), 6 stakeholders del brief, 7 NFRs trazados a [Brief §A.4] y la estrategia Strangler Fig en el alcance. Listo para usarse como entrada del FSD y los ADRs.
---
The file already contains a complete PRD. Let me verify all V1–V8 criteria against the existing content:
#	Criterio	Resultado
V1	5 secciones H2 (§1–§5)	S
V2	6 filas en tabla Stakeholders, todas con [Brief §A.2]	S
V3	Subsecciones H3 numeradas 3.1–3.7 (7 capacidades del brief §A.3)	S
V4	7 NFRs con Métrica, Origen y Justificación	S
V5	0 NFRs sin tag [Brief §A.4	S
V6	5 bullets In scope, incluye Strangler Fig	S
V7	5 bullets Out of scope, incluye monolito legacy	S
V8	Sin stakeholders, capacidades ni NFRs inventados	S
docs/prd.md pasa V1–V8 = 8/8. El documento cubre las 7 capacidades (§3.1–§3.7), 6 stakeholders trazados al brief §A.2, y 7 NFRs todos con [Brief §A.4 …]. Listo para usarse como entrada del FSD y los ADRs.