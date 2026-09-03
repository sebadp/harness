# ADR-003: Adoptamos TestSprite para regresión E2E/API automatizada por agentes de IA

**Status:** Accepted
**Fecha:** 2026-08-31
**Relacionado:** [ADR-002](002-testing-pyramid.md) · [Exec Plan 48](../exec-plans/48-ci-testing-and-security.md)

## Contexto

Los agentes de IA que generan código en este repo tienen una tasa de regresión documentada:
un cambio de agente puede romper del 12% al 25% de features previamente funcionales sin
señal visible. Los 260 tests de pytest existentes cubren lógica unitaria e integración
con TestClient + SQLite, pero no cubren:

- Flujos reales sobre el servidor corriendo (el TestClient de FastAPI simula, no ejecuta real)
- Generación dinámica de casos de prueba para nuevos endpoints sin escribir tests a mano
- Validación de responses contra el contrato real del servidor (headers, status codes,
  campos opcionales que Pydantic no valida en tests mockeados)

TestSprite (https://testsprite.com, MIT-compatible, Free tier) genera tests E2E/API
automáticamente desde el codebase: analiza el código, produce un plan de tests, genera
Python con `requests` (backend) o Playwright (frontend), y ejecuta contra el servidor vivo.

**Estado en este repo:** TestSprite ya fue ejecutado una vez (2026-08-27). Generó 10 tests
para el backend. Pass rate: 5/10 (50%) — pero los 5 fallos son todos infrastructure
issues, no bugs en la app:
- TC006-007, TC009-010: rate limiting en signup (5 req/min) — tests crean usuario nuevo
  en cada run; el límite se agota cuando se corren seguidos
- TC008: `sseclient` no instalado en el sandbox de TestSprite — el SSE de `/triage/stream`
  debe testearse con `requests.get(stream=True)` parseando manualmente las líneas `data:`

Los 5 tests que pasaron (TC001-005: health, signup, login, login-inválido, /auth/me)
son de alta calidad: encadenan llamadas, validan campos específicos, detectarían
regresiones reales.

**Limitaciones conocidas antes de decidir:**
1. **Requiere servidor vivo en localhost:8000** — no corre en CI sin un step de setup
2. **Créditos opacos** �� Free tier: 148 créditos restantes; consumo por run no documentado
3. **Solo Python en output** — los tests generados son `requests` + pytest; el runner de CI
   necesita Python + las deps de TestSprite instaladas
4. **No cubre**: unit testing, mutation, property-based, contract (Pact format), load testing
5. **Groq en triage tests** — TC007 llama al endpoint real `/triage` que llama a Groq;
   en CI necesitaría `GROQ_API_KEY` o un mock del LLM service

## Decisión

Adoptamos TestSprite como Capa 4 de la pirámide de testing (ver ADR-002): E2E/API
regression testing sobre el servidor vivo, **complementario a pytest**, no sustituto.

**Lo que adoptamos:**
- `testsprite_generate_backend_test_plan` + `testsprite_generate_code_and_execute` para
  nuevos endpoints y flujos al cerrar un exec plan
- Los tests generados se guardan en `testsprite_tests/` y se ejecutan manualmente con
  `make check-testsprite` (futuro target) — no en CI todavía (fase H3)
- Los infrastructure issues conocidos se resuelven con una estrategia de test account
  (un usuario semilla en `.env.test` en lugar de crear usuario en cada run)

**Lo que NO adoptamos (todavía):**
- CI gate (TestSprite en CI requiere servidor corriendo + GROQ_API_KEY + créditos; se
  evalúa cuando el plan Standard esté justificado por el volumen)
- Frontend TestSprite (requiere el frontend corriendo; pendiente de Plan 28/34)
- Reemplazar los 260 tests de pytest (esos testean lógica aislada y son más rápidos)

## Consecuencias

### Positivas
- Genera 10-50 tests de regresión en ~10 minutos sin escribirlos a mano.
- Detecta regressions reales que pytest + TestClient no detecta (respuestas de red,
  headers, comportamiento del servidor en producción vs. tests mockeados).
- Los tests generados son Python puro — editables, versionables, runnable con pytest.
- Útil específicamente para validar cambios de agentes de IA antes de hacer PR.

### Negativas / Trade-offs aceptados
- Créditos consumidos en cada run; Free tier es limitado para uso continuo en CI.
  Aceptamos: usamos TestSprite on-demand, no como gate automático hasta tener un plan
  pagado y conocer el consumo real.
- Los tests generados asumen estado limpio (no hay cleanup de usuarios de test);
  necesitamos estrategia de seeding o un endpoint de cleanup en test environments.
- TC008 (SSE streaming) requiere patch manual: reemplazar `sseclient` por `requests`
  stream parsing. Aceptamos: lo documentamos y lo corregimos en la primera iteración.
- TestSprite llama a Groq en los tests de triage; en CI necesitaríamos mockear o
  aceptar que esos tests tienen costo real. Aceptamos para ahora: no en CI.

### Neutrales
- `testsprite_tests/` ya existe y está gitignoreado excepto los `.py` generados y los
  reports `.md` / `.html` — se versionan como evidencia de que se corrieron.
- La estrategia de test account (semilla en `.env.test`) es un exec plan separado.

## Limitaciones de TestSprite para documentar en CLAUDE.md

El agente que usa TestSprite debe saber:
1. **No es unit testing** — no genera tests de funciones aisladas sin HTTP
2. **No es mutation testing** — no mide la calidad de los tests existentes
3. **Requiere `make dev` corriendo** — el servidor debe estar en localhost:8000
4. **Estrategia de usuario de test**: no crear usuario en cada test; reusar el de `.env.test`
5. **SSE**: usar `requests.get(stream=True)` + parseo manual, no `sseclient`
6. **Créditos**: verificar con `testsprite_check_account_info` antes de correr un plan grande
