# Errores, formato de confirmación y output

## Formato del pedido de confirmación

```
⚠️ Estás por ejecutar:
**Acción:** Crear table `axx_creditsolicitud`
**Canal:** DEV / Web API
**Environment:** https://org123-dev.crm2.dynamics.com
**Solución:** ClienteXCreditOnboarding

¿Confirmás?
```

## Manejo de errores

| Error | Causa probable | Alternativa |
|---|---|---|
| `403 Forbidden` (Web API) | Service Principal sin permisos suficientes en DEV | Verificar permisos en Azure AD del cliente |
| `404 Entity not found` | Logical name incorrecto o table no existe | Listar tables con `get_metadata` |
| `DuplicateDetected` | Ya existe un componente con ese nombre | Verificar con `get_metadata` antes de crear (precondición) |
| Pipeline run failed | `pac solution check` falló, import rechazado, gate no aprobado | Resumir el log del pipeline, dar el link al run completo |
| Conector MCP sin responder | El MCP Server del cliente no está accesible desde esta sesión (revisar egress si es sesión en la nube) | Sugerir verificar el conector en Customize → Connectors, o reintentar en sesión local |
| Git push rechazado | Branch desactualizado o conflicto | Traer cambios del branch base, reintentar |

## Output

Sin capa de Adaptive Cards ni formato Copilot Studio — el output es el nativo de Cowork: texto
en el chat para confirmaciones y resúmenes breves, archivos (Markdown/Word/Excel) para
documentación extensa como el SDD, y los links de PR/pipeline en texto plano. No depender de
la skill `output-formatter` — está diseñada para agentes Copilot Studio (Adaptive Cards) y no
aplica en este contexto.
