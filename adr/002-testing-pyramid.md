# ADR-002: Estrategia de testing — pirámide de siete capas

**Status:** Accepted
**Fecha:** 2026-08-31
**Relacionado:** [Exec Plan 48](../exec-plans/48-ci-testing-and-security.md) · [Exec Plan 49](../exec-plans/49-frontend-ci-testing.md) · [ADR-003](003-testsprite-e2e-regression.md)

## Contexto

El repo tiene 260 tests de pytest (154 unit / 106 e2e), CI de seguridad, pre-commit con
ruff + pyright, y a partir del Plan 50 un sensor de diseño (Habit Hooks). Sin embargo,
no existe una decisión explícita sobre la estrategia de testing como sistema: qué capa
cubre qué riesgo, qué se defiere conscientemente y por qué, y cómo se relacionan las
capas entre sí.

Sin ese mapa, los agentes de IA que trabajan en el repo no tienen un criterio claro
para decidir qué tipo de test escribir cuando implementan una feature.

Riesgos concretos sin estrategia documentada:
- Un agente escribe solo tests unitarios para código que solo tiene riesgo de integración.
- Un agente genera tests con TestSprite para código que ya tiene cobertura de pytest, gastando créditos.
- Coverage gaps críticos (mutation, contrato) permanecen indefinidamente porque nadie los priorizó.
- La calidad de los tests no se mide (¿matan los tests bugs reales, o solo pasan?).

## Decisión

Adoptamos una pirámide de siete capas explícita. Cada capa tiene un owner, una
herramienta, y un criterio de cuándo agregar tests en ella. Las capas "deferidas"
están documentadas como tal, con la razón y el criterio de desbloqueo.

```
Capa 7 ── Mutation testing         [DEFERIDO — ver §Deferred]
Capa 6 ── Property-based testing   [DEFERIDO]
Capa 5 ── Contract testing         [DEFERIDO]
Capa 4 ── TestSprite E2E/API       [ACTIVO — ver ADR-003]
Capa 3 ── pytest e2e (TestClient)  [ACTIVO — 106 tests]
Capa 2 ── pytest unit              [ACTIVO — 154 tests]
Capa 1 ── Tipos + estilo           [ACTIVO — ruff + pyright]
```

### Capa 1 — Tipos y estilo (ruff + pyright strict)
- **Qué cubre:** errores de tipo, imports, formato, reglas de diseño estructural (C901, PLR09xx)
- **Cuándo:** en cada commit (pre-commit) y en cada turno del agente (Habit Hooks Stop hook)
- **No cubre:** comportamiento en runtime, lógica de negocio
- **Gate:** bloquea el commit; el Stop hook bloquea el turno del agente (exit 2)

### Capa 2 — pytest unit (154 tests, ~7s)
- **Qué cubre:** lógica pura aislada: schemas Pydantic, helpers, validadores, compilador de prompts, métricas de evals, baggage OTel
- **Criterio:** un test es unit si no usa fixtures `*client`, `*session`, `*engine`
- **No cubre:** integración con FastAPI, DB, red
- **Gate:** CI job `unit`, requerido en PR

### Capa 3 — pytest e2e (106 tests, ~19s)
- **Qué cubre:** endpoints FastAPI completos, auth flows, workspace CRUD, prompt publish/rollback, Gmail sync — todo contra TestClient + SQLite in-memory
- **Criterio:** usa fixtures `client`, `session`, `engine`
- **Limitación conocida y aceptada:** SQLite en lugar de Postgres; diferencias en `RETURNING`, CTEs y aislamiento de transacciones no se detectan aquí (→ futura capa de integración real)
- **Gate:** CI job `e2e`, requerido en PR

### Capa 4 — TestSprite E2E/API (tests contra servidor vivo)
- **Qué cubre:** flujos reales de extremo a extremo contra el servidor corriendo en localhost:8000; generación automática de casos desde el codebase
- **Cuándo agregar:** al cerrar un exec plan con nuevos endpoints o flujos UI; como validación adicional para cambios de agente de IA
- **No cubre:** unit logic, mutation, properties, contrato formal
- **Gate:** no bloquea PR en CI actualmente (H3 del harness); se corre manualmente con `make check-testsprite` (futuro)
- **Ver:** ADR-003 para la decisión completa sobre TestSprite

### Capa 5 — Contract testing [DEFERIDO]
- **Qué cubriría:** compatibilidad de schema entre frontend y backend; que un cambio en el API no rompa el cliente React sin una versión explícita
- **Por qué diferido:** el frontend y backend están en el mismo repo (monorepo); el riesgo de drift silencioso es bajo con pyright strict + Zod en el frontend. Se desbloquea cuando el API tenga consumidores externos o se separe en microservicios.
- **Herramienta candidata:** Pact (consumer-driven) o validación automática del OpenAPI schema contra los responses reales

### Capa 6 — Property-based testing [DEFERIDO]
- **Qué cubriría:** invariantes sobre distribuciones de input: "la confianza siempre está en [0, 1]", "un slug siempre es alfanumérico", "el prompt compilado siempre incluye todas las categorías activas"
- **Por qué diferido:** los invariantes más críticos ya están cubiertos por pyright (tipos) y por validación Pydantic (schemas). El ROI incremental es bajo hasta que tengamos mutation testing que demuestre que los tests actuales son insuficientes.
- **Herramienta candidata:** `hypothesis` + `pytest-hypothesis`; no requiere servidor

### Capa 7 — Mutation testing [DEFERIDO]
- **Qué cubriría:** medir la calidad de los tests existentes: ¿matarían un bug intencional? Kill rate > 80% como gate.
- **Por qué diferido:** requiere ejecutar el test suite N veces por cada mutante (caro en tiempo de CI). Se prioriza cuando coverage gate esté activo y los tests lleguen a ~80% de cobertura.
- **Herramienta candidata:** `mutmut` sobre `email_triage/services/` y `email_triage/auth/` primero
- **Criterio de desbloqueo:** coverage ≥ 80% activo como gate en CI

### Coverage gate [PENDIENTE — prioridad alta]
- **Estado actual:** sin `--cov-fail-under`; nadie sabe qué porcentaje del codebase está cubierto
- **Decisión pendiente:** activar `pytest-cov` con umbral 80% antes de activar mutation testing
- **Esto NO es parte de esta ADR** — requiere exec plan propio

## Consecuencias

### Positivas
- Los agentes tienen un criterio explícito para decidir qué tipo de test escribir.
- Las capas deferidas tienen criterios de desbloqueo — no son "never", son "when X".
- El harness es coherente: cada capa cubre un riesgo que la anterior no puede cubrir.

### Negativas / Trade-offs aceptados
- SQLite en Capa 3 puede dar falsos verdes para código Postgres-específico.
- Sin mutation testing, no sabemos si los 260 tests son "buenos" o solo tienen cobertura superficial.
- Contract testing ausente: si el API cambia de forma breaking, el frontend puede romperse silenciosamente.

### Neutrales
- El orden de las capas no implica que se ejecuten en ese orden en CI; implica el nivel de confianza que aportan.
- Un mismo test no puede estar en dos capas; la clasificación es exhaustiva y mutuamente excluyente.
