---
name: dataverse-connect
description: >
  Setup único que verifica/instala PAC CLI (con confirmación previa), autentica contra el
  environment de Dataverse, y registra el servidor MCP oficial de Microsoft Dataverse —
  distinto del D365 Architect MCP Server propio de Axxon (ADR-002), que las otras 16 skills
  siguen usando sin cambios. Habilita consultas/lecturas y operaciones de datos que ninguna
  otra skill de este paquete cubre. Usar cuando el usuario pida conectar con Dataverse,
  "dv-connect", o pregunte por lectura/analytics/bulk data sobre registros existentes.
  Adaptado del patrón real de `dv-connect` (repo microsoft/Dataverse-skills) — NO instala el
  plugin completo de Microsoft (evita que sus 4 skills que compiten con las nuestras
  —`dv-metadata`, `dv-solution`, `dv-security`, `dv-overview`— entren en conflicto de routing).
---

# Dataverse Connect — Axxon Dataverse Architect

Habilitás el **MCP oficial de Microsoft Dataverse** — no el nuestro — específicamente para
cubrir lo que ninguna de las otras 16 skills de este paquete hace: consultas/analytics sobre
datos existentes, CRUD puntual, bulk import. No tocás nada de lo que ya funciona (`entity-builder`,
`form-designer`, etc. siguen usando el D365 Architect MCP Server propio, sin cambios).

---

## Por qué existe como skill separada, no dentro de `environment-check`

`environment-check` tiene una restricción fuerte: nunca instala nada, solo reporta. Esta skill
sí instala (con confirmación), autentica, y registra un servidor MCP — un perfil de riesgo
distinto, que merece su propio espacio en vez de debilitar la garantía de la otra.

## Qué NO es esto

No instala el plugin completo `dataverse@claude-plugins-official` de Microsoft — ese trae 7
skills más (`dv-metadata`, `dv-solution`, `dv-security`, `dv-query`, `dv-data`, `dv-admin`,
`dv-overview`), 4 de las cuales compiten directo con skills nuestras ya probadas
(`entity-builder`+`form-designer`+`view-designer`, `solution-packager`, `security-architect`,
y nuestro propio `dataverse-architect` conductor). Esta skill toma solo el patrón de
`dv-connect` — verificar, instalar lo que falte, autenticar, registrar el MCP — adaptado a
nuestro alcance específico (habilitar el MCP oficial para lo que nos falta, no reemplazar lo
que ya tenemos).

## Antes que nada — el costo real, no solo técnico

**Desde el 15 de diciembre de 2025, las tools del MCP de Dataverse están medidas por Copilot
Credits cuando las llama un agente fuera de Microsoft Copilot Studio** — esto incluye Claude
Code/Cowork. Antes de correr esta skill con un cliente, confirmá con el usuario que entiende
esta implicancia de costo — no es gratis solo porque el setup lo sea.

## Flujo

1. **Verificar prerequisitos** — PAC CLI instalado (`pac --version`). Si falta, **pedir
   confirmación explícita antes de instalar** — a diferencia del `dv-connect` original, que
   instala directo, acá seguimos nuestro propio principio de nunca instalar sin que el usuario
   lo apruebe primero.
2. **Autenticar** — `pac auth create` (o verificar con `pac auth list` si ya hay una sesión
   activa) contra el environment de Dataverse del proyecto (`.d365-project.md`, sección
   Dataverse — nunca preguntes la URL de nuevo si ya está ahí).
3. **Registrar el MCP oficial de Microsoft** — siguiendo la guía vigente del repo
   `microsoft/Dataverse-skills` (la sintaxis exacta de registro puede cambiar entre versiones
   del CLI — confirmá contra la documentación actual del repo en el momento de ejecutar esto,
   no asumas un comando fijo de memoria).
4. **Confirmar el resultado** — mostrale al usuario que el MCP quedó registrado (debería
   aparecer como `dataverse-<orgname>`), y aclarale explícitamente qué puede pedir ahora que
   antes no podía: consultas sobre datos existentes, CRUD puntual, bulk import — no las cosas
   que ya hacían `entity-builder`/`form-designer`/etc.

## Restricciones

- Nunca instalar PAC CLI (ni nada) sin confirmación explícita del usuario primero — a
  diferencia del `dv-connect` original.
- Nunca instalar el plugin completo de Microsoft como atajo — trae 4 skills que compiten con
  las nuestras, ver arriba.
- Nunca asumas la sintaxis exacta de registro del MCP de memoria — confirmala contra la
  documentación vigente del repo al momento de ejecutar, puede haber cambiado.
- Avisar siempre sobre el costo de Copilot Credits antes de correr esto con un cliente real —
  no es un detalle menor a omitir.
- No reemplaza al D365 Architect MCP Server propio (ADR-002) — las otras 16 skills de este
  paquete no cambian su comportamiento por esta skill.
