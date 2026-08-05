# ADR-001 — Modelo de ejecución del D365 Architect Agent

**Estado:** Aceptado
**Fecha:** 2026-08-05
**Contexto del proyecto:** D365 Architect Agent — Axxon Consulting

> **Nota de actualización (ver ADR-003, RC1):** las referencias a `axx_designsession` en este
> documento reflejan el diseño original bajo Copilot Studio. Esa tabla fue retirada por
> completo — el contexto conversacional vive ahora en Cowork Projects (`.d365-session.md`).
> El razonamiento de fondo de este ADR (canales DEV/Git/Pipeline) no cambia.

---

## Contexto

La v1.0.0 del agente ejecutaba **todas** las operaciones (metadata, forms, business rules,
seguridad, promoción de soluciones) vía llamadas directas del MCP Server a la Dataverse Web
API / Metadata API, contra cualquier environment, incluido PROD. Esto generó tres problemas:

1. **Contradicción de gobierno:** el orchestrator prohibía tocar PROD, pero
   `solution-packager` permitía `import_solution` managed contra PROD sin ningún guard técnico.
2. **Sin audit trail estructurado:** el único registro de "qué se hizo" era el campo Memo
   `axx_actionlog` en la tabla de sesión — no es revisable por pares, no es reproducible,
   no sobrevive a la eliminación de la sesión.
3. **MCP Server sobrecargado:** una única Azure Function concentraba autenticación, lógica de
   negocio de 6 skills distintas, y ejecución directa contra Dataverse — alta superficie de
   riesgo para un cliente FSI.

## Decisión

Se adopta un **modelo de ejecución híbrido**, distinto según la naturaleza de la operación:

| Tipo de operación | Canal | Justificación |
|---|---|---|
| Creación/iteración de metadata, forms, business rules | **Web API, contra DEV únicamente** | Necesitan feedback conversacional inmediato — es el modo de trabajo natural del agente |
| Seguridad (roles, BU, teams, field security) | **XML de solución, versionado en git desde el origen** | Baja frecuencia de iteración, alta sensibilidad — se beneficia más de PR review que de velocidad |
| Environment Variables / Connection References | **`pac solution create-settings` → JSON por ambiente** | Mecanismo oficial de Microsoft, reemplaza el POST manual que teníamos |
| Promoción DEV → TEST → PROD | **Azure DevOps + Power Platform Build Tools, con gates de aprobación** | Reutiliza el estándar ALM que Axxon ya opera (mismo patrón de `ado_client.py`); el agente nunca dispara un import a PROD directamente |

Se descartan como alternativas para este proyecto:
- **PAC CLI como reemplazo de la Web API para creación de metadata** — no existe un
  `pac table create` ni equivalente; PAC CLI gestiona el ciclo de vida de la *solución*, no la
  autoría de metadata desde cero.
- **Power Platform Pipelines nativas** — más simples, pero introducirían una segunda
  herramienta de ALM en paralelo a Azure DevOps, que ya es el estándar de facto de Axxon.
- **ALM Accelerator** — opinionado sobre estructura de repos/pipelines; se evalúa solo como
  acelerador de setup inicial, no como reemplazo del criterio propio de Axxon.

## Consecuencias

- El MCP Server deja de ser un proxy general de la Dataverse Web API. Se divide en dos
  responsabilidades: **(1)** cliente de lectura/escritura Web API acotado a DEV, y
  **(2)** cliente de Azure DevOps (Git API + Pipelines API) para todo lo que promueve más
  allá de DEV.
- La tabla `axx_designsession` deja de ser la fuente de verdad de "qué existe" — pasa a ser
  únicamente contexto conversacional. La fuente de verdad de lo construido es el branch/PR de
  Azure DevOps Repos.
- El agente **nunca** ejecuta un `import_solution` contra TEST o PROD. Genera el artefacto y
  dispara (o deja disparar por policy del pipeline) la promoción; el resultado se reporta de
  vuelta al usuario por polling de estado del pipeline, no por respuesta síncrona.
- Prefijo de publisher y de tablas unificado a `axx_` en todos los archivos (corrige la
  inconsistencia `axx_`/`axxon_` que tenía la v1.0.0 entre skills — el README usaba `axxon_`,
  el resto usaba `axx_`; se confirma `axx_` como estándar del proyecto). Tabla de sesión
  renombrada de forma consistente a `axx_designsession` en todos los archivos.

## Pendiente para las próximas iteraciones

- Definir si Column Security / Business Rules complejas también migran a autoría directa en
  XML (hoy quedan en Web API → DEV, revisar si la frecuencia de cambio lo justifica).
- Definir retención y ownership del repo de Azure DevOps (¿uno por cliente? ¿uno por Axxon con
  carpetas por proyecto?).
