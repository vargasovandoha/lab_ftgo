# Laboratorio FTGO — Henry Vargas

Guía mínima para reproducir la generación de artefactos con los prompts mejorados. Enunciado: [intro.md](intro.md). Dominio: [brief.md](brief.md).

**Entrega:** branch `release/exam-lab`.

### Rutas de artefactos (minúsculas)

| Artefacto | Ruta |
| :--- | :--- |
| PRD | `docs/prd.md` |
| FSD | `docs/fsd.md` |
| ADR 1 | `docs/adr/0001-*.md` |
| ADR 2 | `docs/adr/0002-*.md` |
| C4 contexto | `docs/diagrams/c4_context.mmd` |
| C4 contenedores | `docs/diagrams/c4_container.mmd` |

---

## Cómo “correr” un prompt

No hay script en terminal. En **Cursor o VSC**  (o IDE con `@` de archivos):

1. Abre un chat nuevo.
2. Pega el bloque del artefacto (incluye las líneas `@ruta/al/archivo`).
3. Adjunta o confirma que el modelo ve esos archivos.
4. Pide guardar la salida en la ruta indicada (`docs/...`).
5. Usa **temperatura 0.2** y el modelo del metadato de cada prompt.

El `@archivo.md` carga el prompt o el contexto en la conversación; el texto debajo es la orden de generación.

---

## Comando — PRD (`docs/prd.md`)

Prompt mejorado: [prompts_enhanced/prd_enhanced.md](prompts_enhanced/prd_enhanced.md).

```text
@prompts_enhanced/prd_enhanced.md @brief.md
Genera el PRD ligero de FTGO en Markdown y guárdalo como docs/prd.md
Cumple la Stop condition y el checklist Verification (V1–V8) del prompt
Modelo: Sonnet u Opus. Temperatura: 0.2
```

---

## Comando — FSD (`docs/fsd.md`)

Requiere `docs/prd.md` ya generado. Prompt: [prompts_enhanced/fsd_enhanced.md](prompts_enhanced/fsd_enhanced.md).

```text
@prompts_enhanced/fsd_enhanced.md @docs/prd.md @brief.md
Genera docs/fsd.md con ≥ 5 casos de uso y Given/When/Then explícito por UC.
Modelo: Sonnet. Temperatura: 0.2.
```

---

## Comando — ADR 1 (`docs/adr/0001-*.md`)

```text
@prompts_enhanced/adr_enhanced.md @docs/prd.md @docs/fsd.md @brief.md
Genera docs/adr/0001-estilo-arquitectonico.md (≥ 3 opciones, pros/contras, decisión formal).
Modelo: Sonnet u Opus. Temperatura: 0.2.
```

---

## Comando — ADR 2 (`docs/adr/0002-*.md`)

```text
@prompts_enhanced/adr_enhanced.md @docs/prd.md @docs/fsd.md @brief.md
Genera docs/adr/0002-patron-datos-ipc.md (segunda decisión clave: p. ej. Saga, CQRS o IPC).
Modelo: Sonnet u Opus. Temperatura: 0.2.
```

---

## Comando — C4 (`docs/diagrams/`)

```text
@prompts_enhanced/c4_enhanced.md @docs/prd.md @docs/adr/0001-estilo-arquitectonico.md @docs/adr/0002-patron-datos-ipc.md
Genera docs/diagrams/c4_context.mmd y docs/diagrams/c4_container.mmd (Mermaid C4; protocolos en relaciones del nivel 2).
Modelo: Sonnet. Temperatura: 0.2.
```

---

## Métricas

Indicador PRD: ítems **V1–V8** cumplidos (máx. 8/8). Detalle y tabla de 3+3 corridas: sección **## Métrica** en [prd_enhanced.md](prompts_enhanced/prd_enhanced.md) (cuando la completes).
