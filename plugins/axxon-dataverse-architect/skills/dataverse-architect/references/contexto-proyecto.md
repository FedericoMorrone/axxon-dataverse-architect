# Contexto del proyecto (Cowork Project — ver ADR-003)

Leer al **primer pedido D365 de cada conversación**.

Cada cliente/proyecto es su propio **Cowork Project**. El contexto vive en **un solo
esquema**, el mismo que usa el supraagente (`axxon-orchestrator`), con o sin él:

| Archivo | Qué guarda |
|---|---|
| `.d365-project.md` | Contexto **estático**: environments (DEV/TEST/PROD), solución, publisher prefix, Azure DevOps (org, project, repo, proceso), suscripciones de Azure, equipo, wiki de ADRs |
| `.d365-fases/fase-2-implement.md` | **Log de acciones** de construcción (tabla de historial) y **caché técnico** (ej. GUIDs de privilegios de `security-architect`) |

## Al primer pedido D365 de una conversación

1. Buscá `.d365-project.md` en el Project actual y usalo como contexto (DEV, solución en
   `## Solución Dataverse`, publisher, repo).
2. Si no existe pero hay un **`.d365-session.md` de versiones anteriores**: ofrecé migrarlo —
   copiar sus datos al esquema de `.d365-project.md` y su historial a
   `.d365-fases/fase-2-implement.md` — y, con confirmación, dejar el viejo como
   `.d365-session.md.bak` (no lo borres). Mientras no se migre, podés **leerlo**, pero no
   escribas más en él.
3. Si no existe ninguno, pedí los datos mínimos y creá `.d365-project.md` con el esquema de
   `axxon-orchestrator` (Dataverse, Azure DevOps, Publisher prefix, Solución Dataverse). Si
   el plugin del supraagente está instalado, validalo con su `validate_project_context.py`.

## Modo solo DEV

Para sandbox, prueba de concepto o el entorno de un consultor: `TEST: N/A` y/o `PROD: N/A`
en `.d365-project.md` es válido — nunca cargues URLs ficticias para "pasar" una validación.
En ese modo todo lo de DEV funciona igual, y cualquier promoción a un ambiente `N/A` (canal
Pipeline de `solution-packager`, `deploy-manager`) queda **bloqueada**: explicalo y no lo
intentes.

## Confirmar que se trabaja contra DEV

Antes del primer cambio de la conversación. El MCP no expone `WhoAmI` ni el tipo de
environment, así que no se puede verificar solo con él. La regla es:

1. Mostrá la URL contra la que vas a operar y la URL de DEV de `.d365-project.md`; si no
   coinciden, pará.
2. Si hay PAC CLI autenticado, verificá además: `pac env who` (environment activo) y
   `pac admin list --environment {URL}` (columna **Type**: Sandbox, Developer, Production…).
   `pac admin list` puede requerir permisos de administrador de Power Platform — si falla por
   permisos, seguí con el paso 3. Si el Type es **Production**, pará: no se construye contra
   un environment productivo.
3. Pedí la **confirmación explícita** del usuario de que es su environment de desarrollo.

## Log

Actualizá el log de fase después de cada acción exitosa. Es información del proyecto, no un
log de auditoría regulatorio — si el proyecto necesita eso, es una decisión aparte (ver
ADR-003, pendiente).
