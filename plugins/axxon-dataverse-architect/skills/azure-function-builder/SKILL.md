---
name: azure-function-builder
description: >
  Crea y despliega Azure Functions como parte de la capa de integración de una solución
  Dataverse — webhooks, middleware, procesamiento async pesado llamado desde plugins o Power
  Automate — contra la suscripción de Azure definida en `.d365-project.md`. Usar cuando el
  usuario pida una integración con un sistema externo que necesite código custom en Azure, un
  endpoint HTTP que reciba llamadas de Dataverse, un job programado, o mencione "Azure
  Function", "webhook", "middleware de integración". El deploy a producción pasa por el mismo
  gate de aprobación humana que `solution-packager` — nunca directo. No usar para lógica
  dentro de Dataverse (eso es `plugin-builder`) ni para flows de Power Automate (eso es
  `flow-builder`).
---

# Azure Function Builder — Axxon Dataverse Architect

Construís la pieza de integración que vive **fuera** de Dataverse — en Azure — cuando un
plugin o un flow no alcanza: llamadas a sistemas externos que necesitan reintentos y manejo
de errores robusto, procesamiento pesado que no puede correr sync dentro de un plugin, o un
endpoint HTTP que otro sistema necesita llamar.

---

## Antes de arrancar (0) — Puerta 1 y rama

- Tiene que existir un **Plan de construcción aprobado** por el usuario
  (`build-approach-advisor`) que incluya esta Function. Sin eso, no se crea ningún archivo.
- La rama `feature/US-{id}-...` la crea `code-repo-manager`. El código va en
  `/src/functions/{nombre}`, los tests en `/src/functions/tests/{nombre}.Tests`, la infra en
  `/infra` y el pipeline en `/pipelines` (estructura estándar de `code-repo-manager`).

## Antes de arrancar (1) — verificar el entorno

Corré `environment-check` primero (Azure Functions Core Tools, Azure CLI, .NET SDK) — no
arranques `func init` con una herramienta faltante.

## Antes de arrancar (2) — leer `.d365-project.md`

La suscripción de Azure destino ya está definida ahí (sección "Azure — suscripciones") — no
le preguntes al usuario de nuevo ni asumas una suscripción por default. Si esa sección está
vacía o no existe el archivo, es el mismo gate bloqueante que ya rige para el resto de Fase 2
(ver `axxon-orchestrator`) — parás y lo pedís, no improvisás.

## Triggers relevantes para integración con D365

| Trigger | Cuándo usarlo |
|---|---|
| **HTTP** | Webhook que un plugin (async, `plugin-builder`) o un flow (`flow-builder`) llama; o un endpoint que un sistema externo consume |
| **Timer** | Jobs programados — sincronizaciones periódicas, limpieza, reportes batch |
| **Service Bus / Queue** | Procesamiento asíncrono desacoplado, picos de carga, garantía de entrega |

## Flujo — desarrollo local, luego deploy

1. **Confirmar la suscripción/resource group** contra `.d365-project.md` antes de cualquier
   otra cosa.
2. **Desarrollo local**: `func init` (Azure Functions Core Tools) para el scaffolding, `func
   new` para agregar un trigger, `func start` para correr y probar localmente antes de tocar
   Azure. `local.settings.json` **nunca** se commitea con valores reales (`.gitignore`).
3. **Tests**: proyecto xUnit en `/src/functions/tests`, con la lógica separada del trigger
   (el trigger solo adapta la entrada/salida). Mínimo un test por criterio de aceptación que
   la Function implementa, más el caso de error del sistema externo. `dotnet test` en verde
   antes de la Puerta 2.
4. **Infra y pipeline**: Bicep en `/infra` (ver abajo) y YAML en `/pipelines` (build + deploy
   a DEV; el stage de PROD con environment de Azure DevOps con aprobación manual).
5. **Puerta 2 y PR**: `code-repo-manager` muestra el resumen (archivos, build, tests) y pide
   OK antes del push; luego abre el PR enlazado a la User Story. **Puerta 3:** revisión
   humana del PR.
6. **Deploy a DEV** — con confirmación explícita, idealmente después del PR aprobado:
   `func azure functionapp publish {nombre-app}` — esto sí podés correrlo vos directo contra
   el environment DEV de desarrollo (mismo criterio que el canal DEV del resto del paquete).
7. **Deploy a PROD**: **nunca** `func azure functionapp publish` apuntando a producción
   directo. Sigue el mismo canal Pipeline con gate de aprobación humana que
   `solution-packager` — un pipeline de Azure DevOps con el task `AzureFunctionApp`, revisado
   y aprobado antes de ejecutar.

## Infraestructura como código — preferir sobre creación manual

Cuando haga falta crear el Function App en sí (no solo el código), preferí Bicep/ARM
templates versionados en el repo del proyecto, sobre `az functionapp create` manual desde la
Terminal — la infra queda documentada y repetible entre environments, no depende de que
alguien recuerde los parámetros exactos que usó una vez.

## Restricciones

- Nunca escribas código sin la Puerta 1 aprobada, ni pushees sin la Puerta 2.
- Nunca deploy a producción sin pasar por el canal Pipeline con gate humano — sin excepción,
  mismo criterio que `solution-packager`.
- Nunca asumas la suscripción/resource group — siempre `.d365-project.md` primero.
- Nunca pongas secretos (connection strings, API keys) en el código — Application Settings de
  la Function App, o Azure Key Vault referenciado desde ahí, nunca hardcodeado.
- Preferí HTTP triggers con autenticación explícita (Function key como mínimo, Azure AD/Entra
  para integraciones internas) — nunca `authLevel: anonymous` en un endpoint que toca datos
  del cliente.
