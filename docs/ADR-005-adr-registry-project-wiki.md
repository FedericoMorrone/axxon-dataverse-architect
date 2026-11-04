# ADR-005 — Registro de ADRs del proyecto en la wiki de proyecto de Azure DevOps

**Estado:** Aceptado
**Fecha:** 2026-11-04
**Contexto del proyecto:** Axxon Dataverse Architect — Axxon Consulting
**Depende de:** ADR-001 (modelo de ejecución), ADR-002 (un Project de ADO por cliente)

---

## Contexto

El architect deja trazabilidad de *qué* se construyó (repo/PRs vía ADR-001, log de fase,
Solution Design Document), pero no de *por qué* se eligió una alternativa sobre otra cuando la
decisión tenía alternativas reales: Business Rule vs. flow vs. plugin, la cadena
Custom API → Plugin → Azure Function, integración sync vs. async, modelo de seguridad, GAPs
resueltos con customización. Ese razonamiento quedaba en el chat y se perdía.

Se necesita que, durante la ejecución de un proyecto, el agente registre esas decisiones de
forma estructurada en el Azure DevOps del cliente, donde el equipo y el cliente ya trabajan.

## Decisión

1. **Skill nueva `adr-recorder`** en el architect. La dispara el conductor
   `dataverse-architect` (regla transversal), no cada skill puntual — el conductor es quien ve
   el alcance completo de la User Story antes de rutear.
2. **Destino: wiki de proyecto** de Azure DevOps (no code wiki), vía el servidor MCP oficial
   (`wiki`, `wiki_upsert_page`, `wit_work_item_link_write` con `add_artifact_link` /
   `linkType: "Wiki"`).
3. **Estructura:** página índice en una ruta raíz configurable (`/Arquitectura/ADRs` por
   defecto) + una página por ADR (`ADR-NNNN-slug`), plantilla alineada con el formato de ADRs
   de Axxon más campos de proyecto (User Story, componentes, skills, decisores).
4. **Ciclo de vida:** Propuesto → Aceptado → Reemplazado / Deprecado. Un ADR aceptado es
   inmutable; si la decisión cambia se registra uno nuevo que lo reemplaza.
5. **Configuración** en `.d365-project.md`, sección opcional `## Wiki de ADRs` (validada por
   `validate_project_context.py` solo si está presente).

### Alternativa descartada: code wiki (carpeta del repo publicada como wiki)

Daba un gate natural (aprobar el PR = aceptar el ADR) y coherencia total con ADR-001 (Git
como fuente de verdad). Se descartó por decisión del usuario a favor de la wiki de proyecto:
menos fricción (sin branch/PR por ADR) y funciona aunque el cliente todavía no tenga repo de
solución.

## Consecuencias

- **No hay gate técnico de aprobación**: la wiki de proyecto no tiene PR. El gate es
  conversacional — el agente publica solo como *Propuesto* tras confirmación, y pasa a
  *Aceptado* únicamente cuando el usuario lo indica explícitamente (queda registrado quién y
  cuándo). Es un gate más débil que un PR; si un cliente FSI exige aprobación auditable, se
  revisa esta decisión (code wiki).
- **Riesgo de sobrescritura**: `wiki_upsert_page` hace *update* si la página ya existe. La
  skill verifica con `get_page` antes de crear y calcula el número justo antes de escribir,
  para tolerar consultores en paralelo.
- **Dependencia del MCP de Azure DevOps**: sin él, la skill trabaja en modo offline
  (`.d365-adr/pendiente-*.md`) y publica en la sesión siguiente. Hereda el pendiente de
  conectividad del MCP (config remota OAuth vs. local stdio).
- **Sin creación de wiki por MCP**: el servidor no expone una acción para crear la wiki de
  proyecto; se crea una vez desde la UI de ADO.
- El SDD (`config-generator`) incorpora el índice de ADRs como sección propia.

## Pendiente para las próximas iteraciones

- Verificar contra un ADO real el formato del artifact link de wiki (`vstfs:///Wiki/WikiPage/...`)
  que arma `add_artifact_link`, y si `get_page` devuelve el `pageId`/ETag necesarios.
- Evaluar extender el registro a otras fases (decisiones de Fase 1 en `fdd-builder` /
  `blueprint-builder`, de Fase 3 en `deploy-manager`).
- Numeración de ADRs de producto: existen dos ADR-004 distintos (`axxon-dataverse-architect`
  y `axxon-dataverse-agent`) — definir un esquema que no colisione entre repos.
