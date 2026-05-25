# Examen-Laboratorio Práctico Individual — Caso FTGO

| Campo | Valor |
| :--- | :--- |
| **Modalidad** | Individual — cada maestrante entrega su propio repositorio (público en GitHub, o privado dando acceso al usuario eterceros) |
| **Caso de estudio** | FTGO (Food To Go) del libro *Microservices Patterns* de Chris Richardson (Manning, 2019) |
| **Liberación** | 19 de mayo de 2026 |
| **Deadline** | Viernes 22 de mayo de 2026, 18:59 hora local |
| **Esfuerzo estimado** | ~4 horas |
| **Peso** | 10 % de la nota global |

## Objetivo
Demostrar individualmente la capacidad de:
1. Documentar la arquitectura de un sistema de ejemplo (FTGO) atravesando PRD → FSD → ADRs → diagramas C4 con trazabilidad explícita.
2. Mejorar prompts existentes del módulo (no partir de cero): rellenar huecos TODO, agregar secciones nuevas (anti-patterns / verification / examples), declarar métricas antes/después con evidencia.

## Productos esperados
Cada maestrante entrega en un repositorio individual con branch `release/exam-lab`:

| # | Artefacto | Ruta esperada | Mínimo aceptable |
| :---: | :--- | :--- | :--- |
| **1** | PRD ligero de FTGO | `docs/prd.md` | Secciones contexto, stakeholders, capacidades, NFRs, alcance |
| **2** | FSD ligero | `docs/fsd.md` | $\ge5$ UCs con Given/When/Then explícito |
| **3** | ADR 1 (estilo arquitectónico) | `docs/adr/0001-*.md` | Opciones consideradas ($\ge3$), pros/contras, decisión formal |
| **4** | ADR 2 (patrón de datos / IPC) | `docs/adr/0002-*.md` | Segunda decisión clave (ej. Saga vs 2PC, CQRS, etc.) |
| **5** | Diagrama C4 nivel 1 (Context) | `docs/diagrams/c4_context.mmd` | Mermaid C4Context válido y Contenedores |
| **6** | Diagrama C4 nivel 2 (Container) | `docs/diagrams/c4_container.mmd` | Mermaid C4Container válido |
| **7** | >= 2 Prompts mejorados | `prompts_enahnced/` | Archivos `.md` con cambios aplicados sobre los semilla del Anexo B |
| **8** | README explicativo | `README.md` en la raiz | Guía de ejecución de prompts, comandos exactos y métricas |

## Materiales provistos
Los materiales del examen están incluidos en los Anexos A y B de este documento:
* **Anexo A — Brief del caso FTGO:** contexto, stakeholders, capacidades, NFRs base, 3 user stories semilla.
* **Anexo B — 4 prompts semilla:** (uno por cada artefacto: PRD, FSD, ADR, C4), cada uno con huecos TODO que debes mejorar.

Recursos adicionales compartidos en el Classroom del módulo 4:
* PDF *Microservices Patterns* (Richardson, Manning 2019).
* Plantillas del módulo (ADR, PRD, FSD, PROMPT).

---

## Requisitos para los prompts mejorados (D4)
Elige al menos 2 de los 4 prompts semilla del Anexo B para mejorar. Cada prompt mejorado se guarda en `prompts_enhanced/<nombre>.md` y debe cumplir los siguientes 5 requisitos:

1. **Al menos 2 huecos TODO críticos rellenados con valor real** (los semilla traen 4 huecos cada uno).
2. **1 sección nueva agregada** entre: Anti-patterns, Verification, Examples (input/output).
3. **Changelog** explicando el qué y el por qué de cada cambio (sección `## Changelog`).
4. **Comando invocable documentado en README.md** (ej. `@prompts_enhanced/prd_enhanced.md` genera PRD para FTGO).
5. **Métrica de calidad antes/después con evidencia de 3 corridas** (sección `## Métrica` que reporte cómo cambia un indicador concreto: completitud del output, % de secciones cubiertas, reducción de iteraciones para llegar al resultado, etc.).

---

## Rúbrica (sobre 100 pts, equivale al 10 % de la nota global)

| Criterio | Peso | Excelente | Aceptable | Insuficiente |
| :--- | :---: | :--- | :--- | :--- |
| **Coherencia brief → PRD → FSD → ADR → C4** | 25 % | trazabilidad explícita en cada doc | parcial | rota |
| **Calidad del PRD + FSD** | 20 % | ≥ 5 UCs con Given/When/Then completos + NFRs trazables | parcial | < 5 UCs o sin GWT |
| **Calidad de los 2 ADRs** | 20 % | opciones + trade-offs + consecuencias positivas/negativas | sin trade-offs | sin opciones |
| **Calidad de los diagramas C4** | 10 % | nivel 1 + 2 con tecnología/protocolo en cada relación | parcial | solo 1 nivel o sin protocolos |
| **Mejora de los 2 prompts** | 15 % | 5 requisitos D4 cumplidos en ambos | parcial | mejoras cosméticas |
| **README y self-check** | 10 % | comandos reproducibles + métricas reportadas | parcial | ausente |

### Distribución de puntos:
* **Verificación estructural de artefactos** (estructura del repo, archivos presentes, secciones mínimas): 60 pts.
* **Evaluación cualitativa** (trazabilidad, coherencia con FTGO, calidad de prompts, razonamiento de ADRs, README): 40 pts.

---

## Reglas de entrega y antiplagio
* **Entrega:** branch `release/exam-lab` en el repositorio individual del maestrante.
* **Tardío no se acepta:** la entrega es el branch en la fecha de corte (viernes 22 de mayo, 18:59). Commits posteriores no se evalúan.
* **Individual estricto:** cualquier indicio de copia entre maestrantes o reutilización literal del producto grupal anula el 10 %. Los prompts mejorados deben ser propios; está permitido inspirarse en los del producto grupal pero el changelog debe documentar las diferencias.
* **Trazabilidad obligatoria:** cada decisión arquitectónica debe poder rastrearse a un capítulo del libro Richardson, una restricción del brief o una user story semilla. Inventar dominio fuera de FTGO penaliza.