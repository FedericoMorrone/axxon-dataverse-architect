---
name: custom-api-builder
description: >
  Define Custom APIs de Dataverse — mensajes custom expuestos vía Web API (Actions o
  Functions), con parámetros de request/response tipados, vía MCP tools contra DEV. Usar
  cuando el usuario pida exponer una operación custom callable desde código externo o Power
  Automate, un endpoint tipado que no sea CRUD estándar, o mencione "Custom API",
  "custom action", o "función custom de Dataverse". No usar para la implementación en sí
  (casi siempre un plugin — coordiná con `plugin-builder`) ni para Business Rules (eso es
  `business-rule-engine`).
---

# Custom API Builder — Axxon Dataverse Architect

Definís el **contrato tipado** de una operación custom de Dataverse — qué parámetros recibe,
qué devuelve, si es global o atada a una tabla. La implementación real de esa lógica casi
siempre es un plugin (`plugin-builder`) — esta skill define el "qué", no el "cómo".

---

## MCP Tools disponibles

| Tool name | Método HTTP | Endpoint Dataverse |
|---|---|---|
| `create_customapi` | POST | `/api/data/v9.2/customapis` |
| `get_customapi` | GET | `/api/data/v9.2/customapis(<customapiid>)` |
| `create_customapi_request_parameter` | POST | `/api/data/v9.2/customapirequestparameters` |
| `create_customapi_response_property` | POST | `/api/data/v9.2/customapiresponseproperties` |
| `list_customapis` | GET | `/api/data/v9.2/customapis` |

## Decisiones que confirmar con el usuario antes de crear uno

- **Action o Function**: Function = solo lectura, sin efectos secundarios, se llama por GET.
  Action = puede modificar datos, se llama por POST. Si la operación escribe algo, es Action
  — no lo definas como Function aunque el usuario lo pida así, corregilo y explicá por qué.
- **Binding**: Global (no atado a nada, ej. `axx_CalcularComision`), Entity (atado a un
  registro específico, ej. sobre `axx_poliza`), o Entity Collection (opera sobre un set).
- **Parámetros de request/response**: nombre, tipo (String, Integer, Boolean, EntityReference,
  Entity, etc.), si son opcionales. No inventes parámetros que el usuario no pidió — si falta
  claridad sobre qué necesita recibir/devolver, preguntá antes de crear.

## Quién implementa la lógica — nunca la asumas vos

Un Custom API sin nada detrás no hace nada — necesita un **plugin type** registrado contra su
mensaje (el `uniquename` del Custom API funciona como el mensaje que dispara el plugin). Al
crear un Custom API:

1. Definí el contrato acá (esta skill).
2. Coordiná con `plugin-builder` para la implementación real — nunca generes vos mismo el
   código C# desde esta skill, ni asumas que "ya está resuelto" sin ese paso.
3. Si el usuario prefiere que la lógica viva en un Power Automate flow en vez de un plugin
   (es una opción válida, menos común), coordiná con `flow-builder` en su lugar.

## Restricciones

- Nunca definas un Custom API como Function si va a modificar datos — es Action.
- Nunca inventes parámetros no pedidos — confirmá el contrato exacto con el usuario.
- Nunca generes la implementación (plugin o flow) desde acá — coordiná con la skill
  correspondiente, esta skill solo define metadata.
- No confundir con Custom Actions clásicas (mecanismo anterior, basado en workflows) —
  Custom API es el mecanismo moderno recomendado por Microsoft, usar siempre esto salvo pedido
  explícito de mantener compatibilidad con algo legado.
