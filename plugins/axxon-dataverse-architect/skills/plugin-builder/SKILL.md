---
name: plugin-builder
description: >-
  Desarrolla Plugins de Dataverse (C#, net462) de punta a punta: código, tests, plug-in
  package, manifest de steps, PR en el repo ADO y registro en DEV. Usar para lógica
  server-side transaccional, pre/post-operation o steps.
---

# Plugin Builder — Axxon Dataverse Architect

Diseñás **y desarrollás** la lógica server-side más pesada de una solución Dataverse — la que
Business Rules (cliente/servidor liviano) y Power Automate (orquestación, no transaccional)
no pueden cubrir bien: validaciones complejas, lógica que tiene que correr **dentro de la
misma transacción** que la operación que la dispara, o código con requisitos de performance
que un flow no cumple.

---

## Canal: Git — no DEV/Web API

A diferencia de `entity-builder`/`form-designer`/etc., esto es **código C# compilado**, no
metadata declarativa — sigue el mismo criterio que `security-architect`: vive en Git, se
revisa por PR antes de registrar. Nunca generes y registrés un plugin directo contra DEV sin
que el código haya pasado por control de versiones primero. El manejo de repo/rama/PR lo hace
`code-repo-manager` — esta skill genera el código y los artefactos.

## Precondiciones

1. **Puerta 1:** plan de construcción aprobado por el usuario (`build-approach-advisor`) que
   incluya este plugin. Sin eso, no se crea ningún archivo.
2. **Rama creada** por `code-repo-manager` (`feature/US-{id}-...`).
3. **Herramientas** verificadas por `environment-check`: PAC CLI y .NET SDK (para
   `dotnet build`/`dotnet test`).

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
        catch (InvalidPluginExecutionException)
        {
            throw; // ya trae el mensaje de negocio — no envolverlo de nuevo
        }
        catch (Exception ex)
        {
            tracing.Trace($"Error: {ex}");
            throw new InvalidPluginExecutionException("Mensaje claro para el usuario final", ex);
        }
    }
}
```

**Testeabilidad:** poné la lógica de negocio en una clase aparte que reciba
`IOrganizationService` / `ITracingService` (y los datos del `Target`/images) por parámetro;
`Execute` solo arma el contexto y delega. Así los tests no necesitan simular todo el
`IServiceProvider`.

---

## Flujo de desarrollo (después de la Puerta 1)

### 1. Proyecto

- Ubicación: `/src/plugins/{Prefijo}.{Cliente}.Plugins` (estructura de `code-repo-manager`).
  Un solo proyecto/assembly por solución salvo razón concreta — Microsoft recomienda
  consolidar mientras el assembly quede **< 16 MB**.
- Si el proyecto no existe: `pac plugin init --skip-signing --author "Axxon Consulting"` en
  la carpeta (el nombre de la carpeta define el nombre del proyecto). Genera un proyecto
  **SDK-style** que apunta a **.NET Framework** (`net462`) — los plugins de Dataverse tienen
  que apuntar a .NET Framework.
- **Plug-in package (dependent assemblies) por default:** el build genera un NuGet que se
  registra como *plug-in package*; no requiere firma (`--skip-signing`). Si se firma, todas
  las dependencias tienen que estar firmadas. No uses ILMerge (no soportado por Microsoft).
- No dependas de `System.Text.Json` del sandbox: si lo necesitás, incluilo explícitamente en
  el paquete.
- Si el proyecto ya existe, agregá la clase nueva ahí — no crees otro assembly.

### 2. Código

Patrón de arriba + buenas prácticas de abajo. Si el plugin implementa un Custom API, leé el
contrato definido por `custom-api-builder` (parámetros de entrada/salida) — no lo inventes.

### 3. Tests

- Proyecto `/src/plugins/tests/{Prefijo}.{Cliente}.Plugins.Tests` con **xUnit**, apuntando
  al mismo framework que el plugin.
- Mocks de `IOrganizationService` / `ITracingService` con una librería de licencia
  permisiva (ej. Moq o NSubstitute — confirmá la licencia vigente antes de agregarla).
- **No uses FakeXrmEasy v2 o posterior por default:** su licencia exige, para uso comercial
  o de consultoría, publicar el código de las extensiones (RPL-1.5) o un acuerdo pago. Solo
  si el cliente ya tiene licencia comercial y lo pide explícitamente.
- Mínimo: un test por criterio de aceptación Gherkin que el plugin implementa, más el caso de
  error (`InvalidPluginExecutionException`) y el corte por `Depth`.

### 4. Build

```bash
dotnet build -c Release
dotnet test -c Release
```

El NuGet del plug-in package queda en la salida del build (`bin\Release` /
`bin\outputPackages`). Si el build o un test falla, se corrige antes de la Puerta 2 — nunca
se pushea en rojo sin decírselo explícitamente al usuario.

### 5. Registration manifest

Los steps no viven en el código: documentalos en `/docs/plugins/{Plugin}.registration.md`
dentro de la rama, para que el registro sea revisable en el PR y reproducible:

| Plugin (tipo) | Mensaje | Entidad | Etapa | Modo | Filtering attributes | Pre-image | Post-image | Orden |
|---|---|---|---|---|---|---|---|---|
| `ConsultarBureauPlugin` | Update | `axx_creditsolicitud` | Post-operation (40) | Async | `axx_estado` | `axx_estado, axx_monto` | — | 1 |

### 6. Puerta 2 y PR

`code-repo-manager`: resumen de archivos, build/tests, manifest → OK del usuario → push →
PR enlazado a la User Story. **Puerta 3:** revisión humana del PR.

### 7. Registro en DEV (después del PR aprobado, y con confirmación)

- **Primer registro del package y de sus steps:** según la documentación de Microsoft se hace
  con el **Plugin Registration Tool** (Display by Package → Register New Package, eligiendo
  la solución del proyecto; luego registrar los steps del manifest). Es una herramienta
  gráfica: guiá al usuario paso a paso con el manifest — no lo simules.
- **Actualizaciones posteriores** del mismo package, solo contra **DEV**:
  ```bash
  pac plugin push --pluginId {id del plug-in package} --pluginFile {ruta al .nupkg} --type Nuget --configuration Release --environment {URL DEV}
  ```
  `pac plugin push` exige el ID de un package/assembly **ya registrado** — no crea uno nuevo.
  Guardá el ID en `.d365-project.md` / log de fase después del primer registro.
- Si una actualización **quita** un tipo que tiene steps registrados, Dataverse la rechaza:
  primero hay que desregistrar esos steps (con confirmación, y avisando el impacto).
- Verificá que el package y los steps queden en la **solución del proyecto** (consulta a
  `solutioncomponent`; mismo criterio que `solution-packager`). Desde ahí viajan a TEST/PROD
  por el pipeline, como cualquier componente.

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

- Nunca escribas código sin la Puerta 1 aprobada, ni pushees sin la Puerta 2.
- Nunca registrés directo contra DEV sin pasar por Git/PR primero.
- Nunca `pac plugin push` contra TEST o PROD — el plugin llega ahí dentro de la solución,
  por el pipeline con gate humano (ADR-001).
- Nunca omitas filtering attributes en un step de Update — pedí el campo específico si no
  está claro.
- Nunca generes un plugin sync que haga llamadas HTTP externas — eso es candidato directo a
  `azure-function-builder` (llamado desde un plugin async, o desde Power Automate). **Antes de
  escribir cualquier plugin, preguntate explícitamente si va a necesitar hablar con un sistema
  externo** — si la respuesta es sí, decíselo al usuario y coordiná con `azure-function-builder`
  en el mismo plan, no lo descubras a mitad de la implementación.
- Nunca asumas early-bound sin confirmar que hay una razón real para atarse a ese environment.
