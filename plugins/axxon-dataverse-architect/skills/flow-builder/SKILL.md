---
name: flow-builder
description: >-
  Crea Power Automate cloud flows vía el MCP FlowAgent de Microsoft. Usar para
  automatizaciones, notificaciones o integraciones con connectors estándar.
---

# Flow Builder — Axxon Dataverse Architect

Construís y gestionás **Power Automate cloud flows** — la vía estándar de Microsoft para
automatizar procesos vía connectors (Dataverse, Teams, SharePoint, Excel, Office 365, y
cientos más), sin escribir una Azure Function custom.

## Dependencia distinta al resto del paquete — importante

Esta skill usa el **FlowAgent MCP server** — un bundle oficial de Microsoft (Node.js 18+,
`server/mcp.mjs`, stdio, 50+ tools), **no** el D365 Architect MCP Server que Axxon hostea por
cliente (ver ADR-002). Verificá que el usuario tenga el connector `flowagent` agregado en
Cowork (Customize → Connectors) antes de asumir que las tools están disponibles — si no está,
decíselo explícitamente, no intentes emularlo con Web API directa.

Autenticación: `az login` local, mismo tenant/identity que el `pac auth list` activo del
proyecto — mismo criterio que `genpage-builder`/`code-app-builder`.

## Qué cubrís

**Flows** — `list`, `get`, `create`, `edit` (ediciones quirúrgicas a nivel de acción
individual, no reescribir el flow entero para un cambio chico), `copy` (dentro o entre
environments), `update`, `publish`/`disable`, `delete`.

**Runs** — historial, detalle, acciones, drill-down de iteraciones de loop, cancelar
(individual o todos), reenviar (`resubmit`), diagnosticar.

**Build autónomo**: dada una descripción de lo que el flow tiene que hacer, descubrís el
environment y las conexiones disponibles, generás la definición completa del flow, lo creás,
y opcionalmente lo publicás — siempre mostrando el resumen antes de publicar, nunca en
silencio.

## Cómo encaja en el modelo de canales de Axxon

Es su propio canal — MCP directo contra el environment (como DEV/Web API en espíritu: rápido,
iterativo), pero vía el servidor de Microsoft, no el nuestro. **Nunca publiques un flow
directo contra TEST/PROD** — mismo principio que el resto del paquete: iterás y probás en
DEV, la promoción a TEST/PROD sigue el canal Pipeline de `solution-packager` una vez que el
flow queda empaquetado en la solución.

## Restricciones

- Nunca publiques (`publish`) un flow sin mostrarle antes al usuario qué acciones tiene y
  pedirle confirmación explícita — un flow publicado empieza a correr en producción real.
- Nunca ediciones un flow reescribiéndolo entero cuando el pedido es un cambio quirúrgico a
  una sola acción — usá la edición a nivel de acción, preserva el resto intacto.
- Nunca asumas que el connector `flowagent` está disponible — confirmalo primero.
- Nunca publiques directo contra TEST/PROD — ver "Cómo encaja" arriba.
