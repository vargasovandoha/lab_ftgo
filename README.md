# Laboratorio FTGO — Henry Vargas

Guía mínima para reproducir la generación de artefactos con los prompts mejorados. Enunciado: [intro.md](intro.md). Dominio: [brief.md](brief.md).

**Entrega:** branch `release/exam-lab`.

---

## Estructura del repositorio

```
lab_ftgo/
├── brief.md                          # Descripción del dominio FTGO
├── intro.md                          # Enunciado del laboratorio
├── README.md                         # Este archivo
├── MicroservicesPatterns.pdf         # Referencia bibliográfica
│
├── prompts_seed/                     # Prompts originales (sin modificar)
│   ├── 1_PRD-Lightweight.md
│   ├── 2_FSD-Lightweight.md
│   ├── 3_ADR.md
│   └── 4_C4.md
│
├── prompts_enhanced/                 # Prompts mejorados (usados para generar artefactos)
│   ├── prd_enhanced.md
│   ├── fsd_enhanced.md
│   ├── adr_enhanced.md
│   └── c4_enhanced.md
│
└── docs/                             # Artefactos generados
    ├── prd.md
    ├── fsd.md
    ├── adr/
    │   ├── 0001-estilo-arquitectonico.md
    │   └── 0002-patron-ipc.md
    └── diagrams/
        ├── c4_context.mmd
        └── c4_container.mmd
```

---

### Rutas de artefactos (minúsculas)

| Artefacto | Ruta |
| :--- | :--- |
| PRD | `docs/prd.md` |
| FSD | `docs/fsd.md` |
| ADR 1 | `0001-estilo-arquitectonico.md` |
| ADR 2 | `0002-patron-ipc.md` |
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

Sigue exactamente las instrucciones del prompt. Genera docs/fsd.md con los 7 UCs de la tabla de Context, Given/When/Then explícito por UC, y ejecuta el Checklist de auto-revisión antes de emitir el output.
Modelo: Sonnet. Temperatura: 0.2.

Luego quiero anexar "Checklist de auto-revisión ejecutado antes de emitir" al final del archivo @prompts_enhanced/fsd_enhanced.md
```

---

## Comando — ADR 1 (`docs/adr/0001-*.md`)

```text
@prompts_enhanced/adr_enhanced.md @docs/prd.md @docs/fsd.md @brief.md

Sigue exactamente las instrucciones del prompt. Genera docs/adr/0001-estilo-arquitectonico.md
Modelo: Sonnet. Temperatura: 0.3.

Luego anexar metricas a @prompts_enhanced/adr_enhanced.md
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
@prompts_enhanced/c4_enhanced.md @docs/prd.md @docs/adr/0001-estilo-arquitectonico.md @docs/adr/0002-patron-ipc.md
Sigue exactamente las instrucciones del prompt. Genera docs/diagrams/c4_context.mmd y docs/diagrams/c4_container.mmd
Modelo: Opus. Temperatura: 0.2

Luego anexar metricas a @prompts_enhanced/c4_enhanced.md
```

---

## Métricas

### PRD — indicador V1–V8 (máx. 8/8)

Prompt: [prd_enhanced.md](prompts_enhanced/prd_enhanced.md) · Modelo: Sonnet · Temperatura: 0.2

| Corrida | V1 | V2 | V3 | V4 | V5 | V6 | V7 | V8 | Total |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Run 1 | S | S | S | S | S | S | S | S | **8/8** |
| Run 2 | S | S | S | S | S | S | S | S | **8/8** |
| Run 3 | S | S | S | S | S | S | S | S | **8/8** |

Tasa de éxito: **3/3 = 100 %**

---

### FSD — indicador C-01–C-08 (máx. 8/8)

Prompt: [fsd_enhanced.md](prompts_enhanced/fsd_enhanced.md) · Modelo: Sonnet · Temperatura: 0.2

| Corrida | C-01 | C-02 | C-03 | C-04 | C-05 | C-06 | C-07 | C-08 | Total |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Run 1 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |
| Run 2 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |
| Run 3 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |

Tasa de éxito: **3/3 = 100 %**

---

### ADR-0001 (`estilo-arquitectonico`) — indicador A-01–A-08 (máx. 8/8)

Prompt: [adr_enhanced.md](prompts_enhanced/adr_enhanced.md) · Modelo: Sonnet 4.6 · Temperatura: 0.3

| Corrida | A-01 | A-02 | A-03 | A-04 | A-05 | A-06 | A-07 | A-08 | Total |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Run 1 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |
| Run 2 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |
| Run 3 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |

Tasa de éxito: **3/3 = 100 %**

---

### ADR-0002 (`patron-ipc`) — indicador A-01–A-08 (máx. 8/8)

Prompt: [adr_enhanced.md](prompts_enhanced/adr_enhanced.md) · Modelo: Sonnet 4.6 · Temperatura: 0.3

| Corrida | A-01 | A-02 | A-03 | A-04 | A-05 | A-06 | A-07 | A-08 | Total |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Run 1 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |
| Run 2 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |
| Run 3 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | **8/8** |

Tasa de éxito: **3/3 = 100 %**

---

### C4 — checklist de revisión (C1.1–C2.10 + CC.1–CC.3)

Prompt: [c4_enhanced.md](prompts_enhanced/c4_enhanced.md) · Modelo: Sonnet/Opus · Temperatura: 0.2

| Corrida | Nivel 1 válido | Nivel 2 válido | System_Ext (esp. 4) | Containers/Db/Queue (esp. ≥ 15) | Rels sin protocolo (esp. 0) |
| :---: | :---: | :---: | :---: | :---: | :---: |
| Run 1 | ✓ | ✓ | 4/4 | 17 | 0 |
| Run 2 | ✓ | ✓ | 4/4 | 18 | 0 |
| Run 3 | ✓ | ✓ | 4/4 | 18 | 0 |

Tasa de éxito: **3/3 = 100 %** · Detalle completo en [c4_enhanced.md](prompts_enhanced/c4_enhanced.md).
