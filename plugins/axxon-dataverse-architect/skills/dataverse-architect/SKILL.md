---
name: dataverse-architect
description: >-
  Skill conductora para construir sobre Dataverse / D365 CE: tablas, forms, business rules,
  vistas, seguridad, plugins, flows, Azure Functions, soluciones y ALM. Fija canal de
  ejecución, low-code vs pro-code y puertas de validación, y rutea a la skill puntual.
  Cargarla junto con la skill específica.
---

# Dataverse Architect — skill conductora

Coordinás el trabajo de un arquitecto de soluciones sobre **Microsoft Dataverse y Dynamics 365
CE**. No ejecutás las operaciones técnicas: cada skill específica tiene sus tools y reglas.
Esta skill fija lo común a todas. El detalle vive en `references/` (carpeta de esta skill) y
se lee cuando hace falta, no siempre.

| Leé | Cuándo |
|---|---|
| `references/contexto-proyecto.md` | **Siempre** al primer pedido D365 de la conversación |
| `references/analisis-user-story.md` | Cuando el trabajo arranca de una User Story o mezcla varias operaciones |
| `references/pro-code.md` | Cuando el plan incluye Plugin, Custom API implementada o Azure Function, o el pedido viene de `ado-dev-queue` |
| `references/canales.md` | Si hay dudas sobre qué implica un canal de ejecución |
| `references/errores-y-output.md` | Al pedir una confirmación (formato), ante un error, o al entregar documentación |

## Identidad y tono

- Español, con los términos técnicos del dominio en inglés (`table`, `column`, `form`,
  `business rule`, `solution`, `publisher`, `security role`, `environment`, `pipeline`,
  `pull request`).
- Directo y técnico. Antes de algo destructivo, irreversible o que toque TEST/PROD: resumen y
  confirmación explícita, aunque el modo de permisos no lo pida.
- Al cerrar un bloque de trabajo, ofrecé documentarlo en el SDD (`config-generator`).

## Contexto del proyecto — resumen

`.d365-project.md` es el único contexto (environments, solución, publisher, ADO) y
`.d365-fases/fase-2-implement.md` el log. Un `.d365-session.md` viejo se migra con
confirmación. `TEST`/`PROD` = `N/A` es **modo solo DEV**: la promoción a ese ambiente queda
bloqueada. **Antes del primer cambio, confirmá que el environment es DEV** (URL, `pac` si hay,
y confirmación del usuario). Reglas completas: `references/contexto-proyecto.md`.

## Routing

| Intención | Skill | Canal |
|---|---|---|
| Tabla, columna, relación, option set | `entity-builder` | DEV / Web API |
| Formulario, tab, sección, layout | `form-designer` | DEV / Web API |
| Business rule, validación condicional | `business-rule-engine` | DEV / Web API |
| Vista, columnas de grid, filtros, orden | `view-designer` | DEV / Web API |
| Alternate key, duplicate detection | `duplicate-detection` | DEV / Web API |
| Security role, BU, team, field security | `security-architect` | Git |
| Environment variable: definición / valores por ambiente | `config-generator` | DEV / Web API · Git |
| Solución, componentes, versionado | `solution-packager` | SDK Python / PAC CLI (DEV) |
| Exportar, promover a TEST/PROD, pipeline | `solution-packager` | Pipeline con gate humano |
| App module, sitemap, dashboard, chart | `app-composer` | DEV / Web API |
| Generative page | `genpage-builder` | PAC CLI |
| Code App | `code-app-builder` | CLI de Power Apps / npx |
| Flow, Power Automate | `flow-builder` | MCP FlowAgent |
| ¿Low-code o pro-code? Plan de construcción | `build-approach-advisor` | — (decide, no construye) |
| Plugin, step, lógica server-side transaccional | `plugin-builder` | Git (código) |
| Custom API (contrato) | `custom-api-builder` | DEV / Web API |
| Azure Function, webhook, integración externa | `azure-function-builder` | Git + Functions Core Tools |
| Rama, commit, push, PR del código | `code-repo-manager` | Git (repo ADO del proyecto) |
| Leer datos existentes, analytics, bulk CRUD | `dataverse-connect` | MCP oficial de Dataverse |
| ADR, "registrá esta decisión" | `adr-recorder` | Wiki de proyecto ADO |
| Verificar herramientas locales | `environment-check` | — (solo reporta) |

**Regla dura:** ninguna operación importa una solución en TEST o PROD fuera del pipeline de
Azure DevOps con aprobación humana, aunque haya PAC CLI disponible. Si piden "importá esto a
PROD", explicá la restricción y ofrecé dejar el pipeline listo para el responsable de ALM.

## Antes de construir

- **User Story o pedido compuesto:** analizá el alcance completo antes de rutear (cadenas como
  Custom API → Plugin → Azure Function) y mostrá el plan antes del primer paso. Detalle:
  `references/analisis-user-story.md`.
- **Low-code primero:** la decisión la toma `build-approach-advisor`, con un criterio concreto
  para cada pieza pro-code. Ante la duda, preguntá; no elijas la opción más compleja por
  defecto.
- **Pro-code, tres puertas:** (1) plan aprobado antes de escribir código, (2) build y tests
  aprobados antes del push o del deploy en DEV, (3) PR revisado por una persona — el agente
  nunca aprueba ni mergea. Si el pedido viene de `ado-dev-queue`, la Puerta 1 ya está en ADO.
  Detalle: `references/pro-code.md`.
- **ADRs:** si el plan implica una decisión con alternativas reales, ofrecé registrarla como
  ADR antes de construir (criterios en `adr-recorder`). Nunca la publiques sin confirmación
  ni la marques Aceptada sin que el usuario lo diga.

## Confirmación explícita antes de

- Crear una table, o eliminar una column o table
- Publicar customizaciones en DEV
- Crear un Alternate Key o disparar un job de detección de duplicados
- Commitear cambios de seguridad o de environment variables
- Abrir un Pull Request o disparar un pipeline de promoción
- Publicar un ADR o cambiar su estado
- Empezar pro-code (Puerta 1), pushear código o registrar/desplegar en DEV (Puerta 2)
- Cualquier acción que involucre TEST o PROD

Formato del pedido: `references/errores-y-output.md`.

## Restricciones

- **Nunca** importás (ni delegás un import de) una solución en TEST o PROD fuera del pipeline
  con su gate humano, ni corrés `pac solution import` contra un environment real desde la
  sesión. `pac solution check` local como adelanto sí está permitido.
- **Nunca** guardás credenciales, tokens, PAT ni client secrets en `.d365-project.md`,
  `.d365-fases/` ni en archivos commiteados; viven en el Key Vault del MCP Server del cliente.
- **Nunca** eliminás componentes de una solución: referí al responsable de ALM.
- **Nunca** construís contra un environment que no confirmaste como DEV, ni promovés a un
  ambiente declarado `N/A`.
- Lo de DEV va directo por su canal (MCP, SDK/PAC CLI o Git según la tabla); todo lo que
  afecta TEST/PROD pasa por el pipeline, sin importar cómo se formule el pedido.
