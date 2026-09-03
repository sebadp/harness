# ADR-004: Adoptamos Context7 MCP para documentación de librerías siempre actualizada

**Status:** Accepted
**Fecha:** 2026-08-31
**Relacionado:** [ADR-002](002-testing-pyramid.md) · [ADR-003](003-testsprite-e2e-regression.md)

## Contexto

El stack de este repo usa librerías que evolucionan rápido y tienen APIs no triviales:
`pydantic-ai` (>=1.106, <2), `SQLAlchemy 2.x` async, `FastAPI 0.136+`, `logfire 4.x`,
`alembic`, `pytest-asyncio`, entre otras. El modelo de Claude tiene un knowledge cutoff
de agosto 2025; las versiones de estas librerías cambian cada pocas semanas.

Sin acceso a documentación actualizada, un agente que trabaja en este repo puede:
- Usar APIs deprecadas o renombradas (e.g., `pydantic-ai` tuvo breaking changes entre 0.x y 1.x)
- Inventar parámetros que no existen en la versión pinada (`uv.lock`)
- Generar código que pasa pyright pero falla en runtime por cambios de semántica

**Context7** (`@upstash/context7-mcp`, upstash.com) es un servidor MCP que resuelve este
problema: expone dos herramientas (`resolve-library-id` + `query-docs`) que inyectan
documentación versionada y ejemplos actuales directamente en el contexto del agente.
No requiere API key en el modo gratuito. Se ejecuta localmente vía `npx`.

## Decisión

Añadimos Context7 como MCP server en `.claude/settings.json` del proyecto. El agente
puede usarlo cuando trabaja con código que depende de APIs específicas de librerías —
llamando `resolve-library-id` para encontrar el ID de la librería y `query-docs` para
obtener la documentación actualizada para esa versión.

```json
// .mcp.json (project-level MCP config, committed to repo)
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"],
      "type": "stdio"
    }
  }
}
```

El server se auto-aprueba via `"enabledMcpjsonServers": ["context7"]` en `.claude/settings.json`.

## Cuándo usar Context7

El agente DEBE consultar Context7 cuando:
- Escribe código que usa una API de `pydantic-ai`, `SQLAlchemy async`, `FastAPI`, `logfire`, o `alembic`
- No está seguro si un parámetro o método existe en la versión actual (`uv.lock`)
- Encuentra un error de tipo que puede ser un cambio de API entre versiones
- Implementa un patrón nuevo (p.ej. structured output de pydantic-ai, lifespan de FastAPI)

El agente NO necesita Context7 para:
- Python stdlib
- Código que no depende de librerías específicas (lógica pura, schemas Pydantic simples)
- Cuando ya tiene el código actual en contexto (Edit de un archivo existente)

## Consecuencias

### Positivas
- Elimina alucinaciones de API — el agente usa la versión real, no su entrenamiento
- Acelera el trabajo en librerías como `pydantic-ai` donde los cambios entre versiones son frecuentes
- Cero configuración: `npx` descarga y ejecuta automáticamente; no requiere instalación global

### Negativas / Trade-offs aceptados
- Primera invocación por sesión tiene latencia de ~2s (descarga + arranque del server)
- `npx` requiere Node.js instalado — presente en la mayoría de entornos de desarrollo
- Sin garantía de disponibilidad offline — requiere conexión para el primer `npx` download

### Neutrales
- No reemplaza leer el código fuente del proyecto; Context7 es para APIs de dependencias externas
- Los MCP servers en `settings.json` arrancan lazy (solo cuando se llama una herramienta)
