# ADR-001: Adoptamos Habit Hooks en vez de un script propio

**Status:** Accepted
**Fecha:** 2026-08-31
**Relacionado:** [Exec Plan 50](../exec-plans/50-habit-hooks.md) | [Propuesta 004](../proposals/004-habit-hooks.md)

## Contexto

El Plan 50 implementó un `scripts/habit_hook.py` propio: un script Python que ejecuta
`ruff check --output-format=json`, filtra las violaciones `C9xx`/`PLR09xx`, y mapea
cada código de regla a una guía de refactorización escrita a mano. El script sale con
código `2` para que el Stop hook de Claude Code lo reinyecte al contexto del agente.

Inmediatamente después de implementarlo, se descubrió que
[Habit Hooks](https://github.com/habit-hooks/habit-hooks) (MIT, en PyPI) es un producto
open-source que hace exactamente lo mismo pero con tres ventajas estructurales:

1. **Guías mantenidas por la comunidad**: no tenemos que escribir ni actualizar las
   coaching guides nosotros.
2. **Normalización multi-linter**: mapea `PLR0913`, `max-params`, `max-arguments`
   (ruff/pylint/ESLint) al mismo smell key `too-many-parameters`, lo que lo hace
   útil si el stack crece (TypeScript en el frontend, por ejemplo).
3. **Plugin system desacoplado**: sensor y mapper son independientes; cada uno
   puede ser testeado, reemplazado, o extendido sin tocar el otro.

La alternativa de mantener el script propio implicaba: escribir y actualizar todas las
guías a mano, reinventar la normalización si se agrega un segundo linter, y tener cero
beneficio de la mejora comunitaria del producto.

## Decisión

Deprecamos `scripts/habit_hook.py` y adoptamos el paquete `habit-hooks` como la
implementación del design quality sensor. El Stop hook pasa a ejecutar
`habit-hooks run` en lugar del script propio. Los umbrales del linter
(`pyproject.toml`: `max-complexity`, `max-branches`, etc.) se mantienen en el repo
porque Habit Hooks usa el linter existente — no lo bypasea.

## Consecuencias

### Positivas
- Cero mantenimiento de las coaching guides (responsabilidad del proyecto upstream).
- Lista para multi-linter sin cambios de arquitectura.
- El patrón "sensor + guía en el mismo payload" — el insight central del Plan 50 —
  se preserva; solo cambia quién implementa la guía.

### Negativas / Trade-offs aceptados
- Nueva dependencia de dev (`habit-hooks`). Acepto: es MIT, en PyPI, sin dependencias
  pesadas, y el valor supera el costo de gestión.
- Las guías son genéricas (en inglés, orientadas a cualquier codebase). Si en el futuro
  necesitamos guías muy específicas a las convenciones de este repo (p.ej. "usa Pydantic
  AI para este patrón"), deberemos extender con un plugin propio o mantener
  adicionalmente las guías custom.
- El comportamiento exacto del output depende de la versión de `habit-hooks`. Mitigación:
  pinear la versión en `pyproject.toml` y revisar en upgrades.

### Neutrales
- El planning gate (`require-plan.sh`) no se ve afectado — es un sistema separado.
- Los umbrales de ruff (`max-complexity = 10`, etc.) permanecen en `pyproject.toml`
  porque son configuración del linter, no de Habit Hooks.
- Los `# noqa: C901,PLR09xx` en las funciones pre-existentes también permanecen.
