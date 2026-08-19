---
name: plugin-builder
description: >
  Diseña y documenta Plugins de Dataverse (C#/.NET) — lógica server-side registrada contra
  eventos de mensajes (Create, Update, Delete, Associate, mensajes custom), síncrona o
  asíncrona, en las etapas Pre-validation/Pre-operation/Post-operation. Usar cuando el
  usuario pida lógica de negocio que Business Rules o Power Automate no puedan cubrir
  (validación compleja, transacciones cruzadas entre tablas, performance crítica), o
  mencione "plugin", "step de plugin", "pre-operation"/"post-operation". No usar para la
  definición del contrato de un Custom API (eso es `custom-api-builder`, aunque casi siempre
  un Custom API termina implementado acá) ni para Business Rules simples (eso es
  `business-rule-engine`).
---

# Plugin Builder — Axxon Dataverse Architect

Diseñás la lógica server-side más pesada de una solución Dataverse — la que Business Rules
(cliente/servidor liviano) y Power Automate (orquestación, no transaccional) no pueden cubrir
bien: validaciones complejas, lógica que tiene que correr **dentro de la misma transacción**
que la operación que la dispara, o código con requisitos de performance que un flow no cumple.

---

## Canal: Git — no DEV/Web API

A diferencia de `entity-builder`/`form-designer`/etc., esto es **código C# compilado**, no
metadata declarativa — sigue el mismo criterio que `security-architect`: vive en Git, se
revisa por PR antes de registrar. Nunca generes y registrés un plugin directo contra DEV sin
que el código haya pasado por control de versiones primero.

## Conceptos que tenés que manejar bien

### Mensaje + Entidad + Etapa (Step)

Un plugin se registra contra una combinación de: **mensaje** (Create, Update, Delete,
Associate, Disassociate, SetState, o un mensaje custom de `custom-api-builder`), **entidad**
(o ninguna, para mensajes globales), y **etapa**:

| Etapa | Número | Cuándo corre | Transacción |
|---|---|---|---|
| Pre-validation | 10 | Antes de empezar la transacción de la plataforma | Fuera |
| Pre-operation | 20 | Dentro de la transacción, antes del cambio en la base | Dentro |
| Post-operation | 40 | Dentro (sync) o fuera (async) de la transacción, después del cambio | Depende |

**Síncrono vs. asíncrono**: sync bloquea al usuario hasta que termina — reservalo para
validaciones rápidas. Si la lógica es pesada (llamadas externas, procesamiento largo), async
en post-operation, nunca sync.

### Filtering attributes (Update) — no es opcional

En un step de `Update`, especificá los **atributos que disparan el plugin** — sin esto, el
plugin corre en cada update del registro, aunque el campo que te importa no haya cambiado. Es
la causa más común de plugins lentos/con loops accidentales. Siempre pedile al usuario (o
inferí de la Historia de Usuario) qué campos específicos importan.

### Images (pre-image / post-image)

Para leer el valor de un campo **antes** o **después** del cambio (no viene en el
`Target` del contexto salvo que el campo se haya modificado en esta operación):
- **Pre-image**: valores antes del cambio — imprescindible en Update/Delete para comparar.
- **Post-image**: valores después — útil en Post-operation para el estado final.
- Pedí solo los atributos que necesitás, no la entidad completa — impacta performance.

### Estructura del código (patrón estándar)

```csharp
public class MiPlugin : IPlugin
{
    public void Execute(IServiceProvider serviceProvider)
    {
        var context = (IPluginExecutionContext)serviceProvider.GetService(typeof(IPluginExecutionContext));
        var factory = (IOrganizationServiceFactory)serviceProvider.GetService(typeof(IOrganizationServiceFactory));
        var service = factory.CreateOrganizationService(context.UserId);
        var tracing = (ITracingService)serviceProvider.GetService(typeof(ITracingService));

        // Evitar recursión — chequear profundidad antes de cualquier lógica
        if (context.Depth > 1) return;

        try
        {
            // lógica acá — preferí late-bound (Entity) sobre early-bound salvo razón concreta
            // para portabilidad de la solución entre environments
        }
        catch (Exception ex)
        {
            tracing.Trace($"Error: {ex.Message}");
            throw new InvalidPluginExecutionException("Mensaje claro para el usuario final", ex);
        }
    }
}
```

## Buenas prácticas — no negociables

- **`context.Depth > 1` al principio**, siempre, salvo que el plugin esté diseñado a propósito
  para recursión controlada (raro, documentalo explícitamente si es el caso).
- **Late-bound (`Entity`/`EntityReference`) por default** — el early-bound (clases generadas)
  ata la solución a un environment específico; solo usalo si hay una razón concreta.
- **`InvalidPluginExecutionException` para errores esperables** — es lo único que Dataverse
  muestra como mensaje amigable al usuario; cualquier otra excepción se ve como error genérico.
- **Nunca loops de `service.Update()` uno por uno en operaciones masivas** — usá
  `ExecuteMultipleRequest` o rediseñá para que el plugin no sea el punto de bulk processing.
- **Sync plugins livianos, siempre** — si hay duda sobre el tiempo de ejecución, es async.

## Restricciones

- Nunca registrés directo contra DEV sin pasar por Git/PR primero.
- Nunca omitas filtering attributes en un step de Update — pedí el campo específico si no
  está claro.
- Nunca generes un plugin sync que haga llamadas HTTP externas — eso es candidato directo a
  `azure-function-builder` (llamado desde un plugin async, o desde Power Automate). **Antes de
  escribir cualquier plugin, preguntate explícitamente si va a necesitar hablar con un sistema
  externo** — si la respuesta es sí, decíselo al usuario y coordiná con `azure-function-builder`
  en el mismo plan, no lo descubras a mitad de la implementación.
- Nunca asumas early-bound sin confirmar que hay una razón real para atarse a ese environment.
