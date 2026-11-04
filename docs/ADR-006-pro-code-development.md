# ADR-006 — Desarrollo profesional (pro-code): decisión, construcción y repo del proyecto

**Estado:** Aceptado
**Fecha:** 2026-11-04
**Contexto del proyecto:** Axxon Dataverse Architect — Axxon Consulting
**Depende de:** ADR-001 (modelo de ejecución), ADR-002 (un Project de ADO por cliente), ADR-005 (ADRs del proyecto)

---

## Contexto

El architect tenía skills pro-code (`plugin-builder`, `azure-function-builder`,
`custom-api-builder`), pero:

1. La decisión low-code vs. pro-code estaba repartida en criterios sueltos (análisis de
   alcance del conductor, restricciones de `plugin-builder`) — sin un criterio uniforme ni
   una justificación registrada.
2. `plugin-builder` solo diseñaba y documentaba; no producía un proyecto compilable, tests ni
   registro.
3. Ninguna skill definía *cómo* llega el código al repo del cliente (qué repo, estructura,
   rama, PR), aunque ADR-001 establece el repo como fuente de verdad.
4. La confirmación era por acción puntual, no sobre el plan de construcción completo.

## Decisión

1. **`build-approach-advisor`** (nueva): matriz de decisión por necesidad (transaccionalidad,
   integración externa, volumen/performance, orquestación, operación custom, lógica de UI,
   mantenibilidad, licenciamiento/costo, ALM). Regla: **low-code primero**; pro-code solo con
   un criterio concreto, que queda explícito y se ofrece registrar como ADR (ADR-005).
2. **Tres puertas de validación** para todo pro-code:
   - **Puerta 1 — Plan:** el usuario aprueba el Plan de construcción antes de crear la rama
     o escribir código.
   - **Puerta 2 — Resultado:** el usuario aprueba archivos, build y tests antes del push y
     antes de registrar/desplegar en DEV.
   - **Puerta 3 — PR:** una persona del equipo revisa y aprueba en Azure DevOps; el agente
     nunca aprueba ni mergea.
3. **`code-repo-manager`** (nueva): **un repo por proyecto** (solución + `/src/plugins`,
   `/src/functions`, `/src/logicapps`, `/infra`, `/pipelines`, `/docs`), una rama por User
   Story (`feature/US-{id}-{slug}`), PR enlazado al work item. Git local con las credenciales
   del usuario, o MCP de Azure DevOps (`repo_create_branch`, `repo_pull_request_write`).
4. **`plugin-builder`** pasa a desarrollar de punta a punta: `pac plugin init --skip-signing`
   (SDK-style, .NET Framework `net462`), plug-in package (dependent assemblies) por default,
   tests xUnit con mocks de licencia permisiva, `dotnet build`/`test`, registration manifest
   versionado en `/docs`, registro en DEV post-PR.
5. **`azure-function-builder`** se integra con las puertas y el repo (tests, Bicep, YAML).
6. **Logic Apps** (`logic-app-builder`, Standard por default) queda para v1.9.

### Alternativas descartadas

- **Repo de código separado del de la solución:** dos PRs por User Story y más
  coordinación; se prefirió un solo repo por proyecto.
- **FakeXrmEasy para tests de plugins:** desde v2 su licencia exige, para uso comercial o
  de consultoría, publicar el código de las extensiones (RPL-1.5) o un acuerdo pago. Se usa
  xUnit + mocks con licencia permisiva; FakeXrmEasy solo si el cliente ya tiene licencia.
- **Registrar plugins y steps vía `pac plugin push`:** el comando exige el ID de un
  package/assembly ya registrado — solo actualiza. El primer registro y los steps se hacen
  con el Plugin Registration Tool, guiado por el registration manifest.

## Consecuencias

- Toda pieza pro-code queda justificada (y opcionalmente con ADR), revisada por PR y
  trazable a su User Story.
- Más fricción deliberada: tres validaciones humanas por pieza pro-code.
- El primer registro de plugins depende de un paso manual (PRT). Las actualizaciones en DEV
  sí son automatizables (`pac plugin push`, solo DEV).
- TEST/PROD no cambian: solución por pipeline con gate (plugins); pipeline propio con
  environment de aprobación (Functions).

## Pendiente para las próximas iteraciones

- `logic-app-builder` (v1.9): Logic Apps Standard, workflows versionados, Bicep, deploy por
  pipeline. Verificar antes el mecanismo de deploy de Standard desde Azure DevOps.
- Evaluar registro de steps vía Web API (`sdkmessageprocessingstep`) contra DEV para eliminar
  el paso manual del PRT — verificar en un environment real antes de adoptarlo.
- Plantillas de pipeline YAML (CI de plugins y Functions) reutilizables entre proyectos.
