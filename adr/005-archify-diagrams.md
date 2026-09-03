# ADR-005: Archify como skill para diagramas de arquitectura verificados

**Status:** Proposed
**Fecha:** 2026-08-31
**Relacionado:** [ADR-004](004-context7-mcp.md)

## Contexto

El harness tiene documentación técnica rica (proposals, exec-plans, ADRs) pero ninguna
capa de visualización de arquitectura. Los diagramas de sistemas son útiles para:
- Incorporar nuevos miembros al equipo
- Documentar decisiones arquitectónicas en los ADRs con evidencia visual
- Revisar cambios de arquitectura antes de un merge (before/after comparisons)
- Generar diagramas de secuencia para flujos complejos (OAuth, Gmail sync, triage stream)

**Archify** (github.com/tt-a1i/archify, MIT, 20k+ ★) es un sistema de rendering y
validación para Claude Code y otros AI agents. Los agentes producen JSON tipado (IR);
Archify lo compila determinísticamente en HTML/SVG interactivo con temas dark/light,
export PNG/SVG/WebM, pan/zoom, route tracing y share cards.

Soporta 5 tipos: `architecture`, `workflow`, `sequence`, `dataflow`, `lifecycle`.
Tiene validación stricta (9 artifact checks, 0 composition errors) y un `deliver`
command que freezea el output con SHA-256.

**Por qué está en Proposed y no Accepted:**
Archify requiere instalar el package de Node.js (`npx skills add tt-a1i/archify -g`)
que no está en las dependencias del proyecto. El skill es un agente-guiado renderer, no
un MCP server — el agente genera JSON y corre comandos CLI. Requiere que el humano
instale el package antes de que el agente pueda usarlo.

## Decisión (propuesta)

Cuando el humano instale Archify globalmente (`npx skills add tt-a1i/archify -g`),
el agente puede usarlo para:

1. **ADRs con diagramas**: cuando un ADR describe una decisión arquitectónica compleja,
   generar un diagrama `architecture` o `sequence` como evidencia adjunta.
2. **Documentar el harness**: generar el diagrama `workflow` del ciclo completo del agente
   (read → plan → implement → test → decide) para la landing page.
3. **Diagramas de secuencia**: para flujos complejos como el OAuth callback o el Gmail
   sync, generar un `sequence` diagram que viva en `docs/features/`.
4. **Before/after comparisons**: al cambiar una parte del sistema, generar un diagrama
   del estado antes y después como evidencia de revisión.

## Tipo de diagrama por caso de uso en este repo

| Caso | Tipo Archify | Ejemplo |
|---|---|---|
| Mapa del harness completo | `workflow` | ciclo del agente 5 fases |
| Arquitectura del sistema | `architecture` | FastAPI + DB + Groq + Logfire |
| Flujo OAuth2 | `sequence` | `/auth/callback` step-by-step |
| Ciclo de vida de un prompt | `lifecycle` | draft → preview → published → superseded |
| Pipeline de triage | `dataflow` | email → LLM → category → triage log |

## Instalación (manual, por el humano)

```bash
npx skills add tt-a1i/archify -g
node bin/archify.mjs doctor   # verifica que todo OK
```

Después de instalar, el agente puede usar el skill directamente en la sesión de Claude Code.

## Consecuencias

### Positivas
- Diagramas verificables (no "dibujados a mano"): el JSON IR es versionable y el
  HTML es generado determinísticamente desde él.
- Integración nativa con Claude Code — el agente genera los diagramas sin salir del chat.
- 20k+ stars, MIT, mantenido activamente (v2.16 en agosto 2026).

### Negativas / Trade-offs
- Requiere instalar un package de Node.js global — no es una dependencia de `uv.lock`
- El skill tiene un proceso de authoring estricto (validate → deliver → visual-check)
  que requiere varias rondas de corrección
- No es un MCP server — es un CLI que el agente corre via Bash; requiere permisos
  `Bash(node bin/archify.mjs *)` en `settings.local.json`

### Condición de transición a Accepted
Esta ADR pasa a Accepted cuando:
1. El humano instala el package (`npx skills add tt-a1i/archify -g`)
2. Se genera y commitea el primer diagrama del harness
3. Se añade el permiso `Bash(node bin/archify.mjs *)` a `settings.local.json`
