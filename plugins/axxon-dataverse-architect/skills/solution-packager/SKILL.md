---
name: solution-packager
description: >
  Gestiona el ciclo de vida ALM de soluciones en Dataverse / Dynamics 365 CE: crea publishers
  y soluciones en DEV vía Web API, agrega componentes, versiona, y promueve DEV→TEST→PROD
  exclusivamente vía pipeline de Azure DevOps con pac CLI y gates de aprobación — nunca importa
  directo a TEST/PROD. Usar esta skill cuando el usuario pida crear una solución o publisher,
  agregar un componente a la solución, versionar la solución, promover al ambiente de TEST o
  PROD, listar soluciones o componentes, consultar el estado de un pipeline o deploy, generar
  un script de pac CLI, o hable de ALM en general. Usar junto con la skill dataverse-architect.
---

# Solution Packager — Axxon Dataverse Architect

Skill especializada en la gestión del ciclo de vida de **soluciones** en Dataverse / D365 CE.
A partir de v2.0.0 opera en **dos canales completamente separados** (ver ADR-001):

1. **DEV / Web API** — crear publisher, crear solución, agregar componentes, versionar.
   Feedback inmediato en el chat.
2. **Pipeline** — todo lo que promueve la solución más allá de DEV. El agente **dispara y
   consulta** el pipeline; **nunca** ejecuta el import contra TEST/PROD por su cuenta.

---

## MCP Tools disponibles

### Canal DEV / Web API

| Tool name                  | Método HTTP | Endpoint Dataverse                                                        |
|------------------------------|-------------|---------------------------------------------------------------------------|
| `list_publishers`          | GET         | `/api/data/v9.2/publishers?$select=uniquename,friendlyname,customizationprefix` |
| `create_publisher`         | POST        | `/api/data/v9.2/publishers` (solo en DEV)                                 |
| `list_solutions`           | GET         | `/api/data/v9.2/solutions?$filter=ismanaged eq false`                     |
| `get_solution`             | GET         | `/api/data/v9.2/solutions(<solutionid>)`                                  |
| `create_solution`          | POST        | `/api/data/v9.2/solutions` (solo en DEV)                                  |
| `update_solution_version`  | PATCH       | `/api/data/v9.2/solutions(<solutionid>)`                                  |
| `list_components`          | GET         | `/api/data/v9.2/msdyn_solutioncomponentsummaries?$filter=msdyn_solutionid eq '<solutionid>'` |
| `add_component`            | POST        | `/api/data/v9.2/AddSolutionComponent`                                     |

### Canal Pipeline (Azure DevOps)

| Tool name                  | Acción                                                                     |
|------------------------------|-------------------------------------------------------------------------------|
| `export_solution_to_repo`  | Exporta la solución de DEV, la commitea (unmanaged + `pac solution unpack`) al branch de la sesión |
| `create_deployment_settings`| Ejecuta `pac solution create-settings` sobre el export y commitea el JSON resultante |
| `trigger_promotion_pipeline`| Dispara el pipeline `{Proyecto}-CD` para el ambiente target (TEST o PROD) |
| `get_pipeline_status`      | Solo lectura — consulta el estado de un run (`running`/`succeeded`/`failed`) |
| `list_missing_dependencies`| Ejecuta `RetrieveMissingDependencies` antes de habilitar la promoción      |

> Notá que ya no existen `export_solution`/`import_solution` como tools que el MCP Server
> ejecuta directo contra Dataverse — quedan reemplazados por el flujo de repo + pipeline.

---

## Flujo ALM v2.0.0

```
[DEV — Web API]                          [Azure DevOps — git]              [TEST/PROD — pipeline]
crear_publisher (axx)                          │                                    │
crear_solución                                  │                                    │
  agregar_componentes                           │                                    │
    (entity, form, business rule, etc.)         │                                    │
  versionar (1.0.0.X)                           │                                    │
       ──── export_solution_to_repo ───────────→│                                    │
                                          pac solution unpack (en el pipeline CI)     │
                                          commit a branch de la sesión                │
                                          create_deployment_settings                  │
                                          ──── Pull Request (confirmación explícita)  │
                                                 │                                    │
                                          merge → pipeline CI dispara automático      │
                                                 pac solution pack                    │
                                                 pac solution check (gate de calidad) │
                                                 ──── trigger_promotion_pipeline ─────→│
                                                                              gate de aprobación manual
                                                                              pac solution import --settings-file
                                                                              (managed, con Build Tools)
```

**Diferencia clave con v1.0.0:** ya no hay un solo salto DEV→TEST→PROD ejecutado por API. Hay
un circuito de PR + pipeline con al menos un gate humano antes de TEST y otro, obligatorio,
antes de PROD.

---

## Crear publisher (sin cambios de canal — sigue siendo Web API en DEV)

### Inputs requeridos

| Input                   | Descripción                                        | Ejemplo          |
|---------------------------|--------------------------------------------------------|--------------------|
| `uniqueName`            | Nombre único del publisher (sin espacios)          | `axx`             |
| `friendlyName`          | Nombre legible                                     | `Axxon Consulting`|
| `customizationPrefix`   | Prefijo de 2-8 letras minúsculas sin espacios      | `axx`             |
| `customizationOptionValuePrefix` | Prefijo numérico para option values     | `10000`            |

### Payload `create_publisher`

```json
{
  "uniquename": "axx",
  "friendlyname": "Axxon Consulting",
  "customizationprefix": "axx",
  "customizationoptionvalueprefix": 10000,
  "description": "Publisher de Axxon Consulting para implementaciones D365 CE",
  "address1_city": "Buenos Aires",
  "address1_country": "Argentina"
}
```

---

## Crear solución (sin cambios de canal)

### Convención de nombres de soluciones

```
{ClienteAbreviado}{Proyecto}{Módulo}
Ejemplos:
  ClienteXCreditOnboarding
  BGaliciaCRMSales
  SecuritasColombiaContactCenter
```

### Payload `create_solution`

```json
{
  "uniquename": "ClienteXCreditOnboarding",
  "friendlyname": "Cliente X — Credit Onboarding",
  "description": "Solución de onboarding KYC/AML para el proyecto Credit Onboarding",
  "version": "1.0.0.1",
  "publisherid@odata.bind": "/publishers(<publisher-guid>)"
}
```

### Convención de versionado

```
Major.Minor.Build.Revision
1.0.0.1   → Primera build de desarrollo
1.0.1.0   → Release a TEST (vía pipeline)
1.1.0.0   → Nueva funcionalidad menor
2.0.0.0   → Cambio breaking (nueva arquitectura de modelo)
```

---

## Agregar componentes a la solución (sin cambios de canal — Web API, DEV)

Referencia de `ComponentType` values:

| Componente                    | `ComponentType` | Notas                                      |
|----------------------------------|------------------|-----------------------------------------------|
| Entity / Table                | 1               | Incluye automáticamente columnas y vistas   |
| Attribute / Column            | 2               | Solo si no se incluye la entity completa    |
| Relationship                  | 10              |                                             |
| Global Option Set             | 9               |                                             |
| Form                          | 24              |                                             |
| View (SavedQuery)             | 26              |                                             |
| Business Rule (Workflow cat.2)| 29              |                                             |
| Security Role                 | 14              | Ver `security-architect` — se agrega vía canal Git, no vía este tool |
| Web Resource                  | 61              |                                             |
| Plugin Assembly               | 91              |                                             |
| SDK Message Processing Step   | 92              |                                             |
| Connection Role               | 63              |                                             |
| Canvas App                    | 300             |                                             |
| Custom API                    | 10145           |                                             |

> ⚠️ **`Security Role` (14) sin verificar** contra fuente oficial vigente — heredado de la
> v1.0.0 sin re-chequeo. **`Alternate Key` y `Duplicate Detection Rule` deliberadamente no
> tienen un `ComponentType` documentado acá**: se encontró un caso real reportado donde
> `ComponentType=44` para `DuplicateRule` fue rechazado por Dataverse
> ("Invalid component type provided"). Antes de scriptear un `add_component` para cualquiera
> de estos dos, confirmar el código contra `$metadata` o la documentación oficial vigente en
> el momento de la implementación — no asumir los valores de esta tabla.

```json
{
  "ComponentId": "<GUID del componente>",
  "ComponentType": 1,
  "SolutionUniqueName": "ClienteXCreditOnboarding",
  "AddRequiredComponents": true
}
```

---

## Exportar y promover (canal Pipeline — reemplaza completo a v1.0.0)

### Paso 1 — `export_solution_to_repo`

```json
{
  "solutionUniqueName": "ClienteXCreditOnboarding",
  "repoId": "<GUID del repo de Azure DevOps>",
  "branch": "feature/onboarding-kyc",
  "environmentUrl": "https://orgXXX-dev.crm2.dynamics.com"
}
```

Internamente, el pipeline de build (no el MCP Server) ejecuta:
```bash
pac solution export --name ClienteXCreditOnboarding --path ./export --managed false
pac solution unpack --zipfile ./export/ClienteXCreditOnboarding.zip --folder ./src --packagetype Both
```
y el resultado se commitea al branch indicado.

### Paso 2 — `create_deployment_settings`

Genera el archivo de Environment Variables / Connection References por ambiente (ver también
`06-config-generator`):

```bash
pac solution create-settings --solution-zip ./export/ClienteXCreditOnboarding.zip --settings-file ./deployment-settings-test.json
```

El JSON resultante se commitea junto al resto — los valores reales por ambiente los completa
`config-generator`, nunca se dejan hardcodeados en el repo en texto plano si son secretos
(referencian Key Vault).

### Paso 3 — Confirmación explícita y Pull Request

El orchestrator **siempre** pide confirmación antes de abrir el PR (acción de "explicit
permission required" — ver reglas de seguridad del agente). El PR es lo que dispara la
revisión humana antes de que cualquier pipeline de CD corra.

### Paso 4 — `trigger_promotion_pipeline`

Solo se invoca **después** de que el PR fue mergeado (verificado, no asumido) y el usuario
confirmó explícitamente el ambiente target:

```json
{
  "pipelineId": 42,
  "targetEnvironment": "TEST",
  "solutionVersion": "1.0.1.0",
  "settingsFile": "./deployment-settings-test.json"
}
```

El pipeline internamente corre:
```bash
pac solution pack --folder ./src --zipfile ./build/solution_managed.zip --packagetype Managed
pac solution check --path ./build/solution_managed.zip   # gate de calidad — bloquea si hay errores críticos
pac solution import --path ./build/solution_managed.zip --settings-file ./deployment-settings-test.json --environment test
```

Para PROD, el mismo pipeline pero con un **gate de aprobación manual adicional** configurado
en Azure DevOps Environments — nunca disparado automáticamente, ni siquiera si TEST pasó
todos los checks.

### Paso 5 — `get_pipeline_status` (polling)

```
Estado del pipeline ClienteXCreditOnboarding-CD:
🟡 Running — pac solution check en curso (paso 2 de 4)
Link: https://dev.azure.com/axxon/.../runs/1234
```

El agente reporta el estado cuando el usuario pregunta, o proactivamente cuando el run
termina si la conversación sigue activa. No bloquea el chat esperando.

---

## Gestión de dependencias de solución

Antes de habilitar la promoción, verificar dependencias faltantes:

```
list_missing_dependencies → RetrieveMissingDependencies
{ "SolutionUniqueName": "ClienteXCreditOnboarding" }
```

Si hay dependencias faltantes, mostrar la lista y guiar al usuario para que las resuelva
antes de proceder con el PR.

---

## Restricciones

- **Nunca** el MCP Server ejecuta `import_solution` (ni su equivalente) contra TEST o PROD.
  Esa operación vive únicamente dentro del pipeline de Azure DevOps.
- **Nunca** se abre un Pull Request sin confirmación explícita del usuario en el chat.
- **Nunca** se dispara `trigger_promotion_pipeline` hacia PROD sin que el usuario haya
  confirmado explícitamente el ambiente y la versión — y aun así, el pipeline exige su propio
  gate de aprobación humana, independiente de esta confirmación conversacional.
- Si el environment target ya tiene la solución en versión superior a la que se quiere
  importar, `pac solution check`/`import` rechazará con `VersionMismatch` — se reporta al
  usuario, no se reintenta automáticamente con otra versión.
- **Nunca** eliminar componentes de una solución administrada — crear una patch solution
  o una nueva versión.
- Siempre incrementar la versión antes de exportar para ambientes distintos a DEV.
