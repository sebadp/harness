# Architecture Decision Records

Decisiones arquitectónicas significativas capturadas junto al código.
Un ADR = una decisión. Son permanentes: no se borran, se deprecan o superseden.

> "One of the hardest things to track during the life of a project is the motivation
> behind certain decisions." — Michael Nygard, 2011

## Cuándo escribir un ADR

**Requerido** cuando la decisión:
- Cambia el stack o adopta/depreca una herramienta principal
- Introduce un patrón arquitectónico que afecta múltiples features futuras
- Rechaza explícitamente una alternativa que parece obvia (para futuros lectores)
- Tiene consecuencias negativas conocidas que se aceptan deliberadamente
- Es cross-cutting: afecta más de un dominio del sistema

**No requerido** para:
- Features rutinarias (esas van en exec-plans)
- Cambios de configuración sin impacto arquitectónico
- Bug fixes y refactors locales

## Relación con otros artefactos

```
Proposal (PRD)          →  estrategia multi-fase, múltiples decisiones
Exec Plan (PRP)         →  plan de implementación concreto, scope + checklist
ADR                     →  una sola decisión, permanente, con contexto + consecuencias
```

Un exec plan puede referenciar un ADR que ya existe, o disparar la creación de uno nuevo
al cerrarse. Los ADRs no reemplazan los exec plans — los complementan.

## Statuses

| Status | Significado |
|---|---|
| `Proposed` | Draft, en discusión |
| `Accepted` | Decisión tomada, en vigor |
| `Deprecated` | Ya no aplica, sin sucesor directo |
| `Superseded` | Reemplazado por otro ADR (con link) |

## Índice

| # | ADR | Status | Fecha | Área |
|---|---|---|---|---|
| 001 | [Adoptar Habit Hooks en vez de script propio](001-adopt-habit-hooks.md) | Accepted | 2026-08-31 | Tooling / DX |
| 002 | [Estrategia de testing — pirámide de siete capas](002-testing-pyramid.md) | Accepted | 2026-08-31 | Testing / CI |
| 003 | [TestSprite para regresión E2E/API automatizada por agentes de IA](003-testsprite-e2e-regression.md) | Accepted | 2026-08-31 | Testing / CI |
| 004 | [Context7 MCP para documentación de librerías siempre actualizada](004-context7-mcp.md) | Accepted | 2026-08-31 | Tooling / DX |
| 005 | [Archify como skill para diagramas de arquitectura verificados](005-archify-diagrams.md) | Proposed | 2026-08-31 | Tooling / Docs |

## Convenciones

- **Nombre:** `NNN-kebab-case.md` con prefijo numérico (`001-`, `002-`, …)
- **Tamaño:** 1–2 páginas. Si necesita más, probablemente son dos decisiones.
- **Voz activa:** "Adoptamos X" / "Usaremos Y", no "Se decidió usar Y".
- **Template:** [TEMPLATE.md](TEMPLATE.md)
