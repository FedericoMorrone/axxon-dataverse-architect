---
name: config-generator
description: >
  Genera artefactos de configuración a partir de las acciones de otras skills del Axxon
  Dataverse Architect: Environment Variables (definición en DEV vía Web API, valores por ambiente
  vía `pac solution create-settings` en git), documentación técnica, y el Solution Design
  Document. Usar esta skill cuando el usuario pida generar un script de pac CLI, exportar
  configuración, generar documentación, crear un script reproducible, documentar lo que se
  hizo en la sesión, generar el Solution Design Document o armar el SDD, generar el JSON de la
  solución, crear environment variables, o generar el deployment settings file. Usar junto con
  la skill dataverse-architect.
---

# Config Generator — Axxon Dataverse Architect

Skill transversal que toma el output de las otras skills y genera **artefactos listos para
usar**: deployment settings, specs, documentación y checklists.

**Cambio de canal en v2.0.0:** el manejo de Environment Variables y Connection References
deja de ser un POST manual a `environmentvariablevalues` por ambiente. Se resuelve con el
mecanismo oficial de Microsoft: `pac solution create-settings`, que genera un JSON parametrizable
por ambiente y se consume directo por `pac solution import --settings-file` en el pipeline
(ver `04-solution-packager`). Esta skill genera y edita ese JSON — no llama a la Web API para
esto.

---

## Artefactos que puede generar

| Artefacto                    | Formato      | Canal | Cuándo generarlo                                          |
|----------------------------------|----------------|---------|-----------------------------------------------------------------|
| Deployment Settings file     | JSON (`pac solution create-settings`) | Git | Cuando hay valores que cambian por ambiente |
| Solution Design Document     | Markdown / Word | — | Al final de una sesión de diseño o bajo demanda           |
| Matriz de componentes        | Markdown / CSV | — | Inventario de lo construido en la sesión              |
| Deployment checklist         | Markdown     | — | Pre-deploy a TEST o PROD                                  |

---

## MCP Tools disponibles

| Tool name                    | Canal | Acción                                                        |
|-----------------------------------|---------|--------------------------------------------------------------------|
| `list_env_variables`         | Lectura Web API (DEV) | `/api/data/v9.2/environmentvariabledefinitions` — para saber qué variables ya existen |
| `create_env_variable_definition` | **DEV / Web API** | Crea la *definición* (schema name, tipo, display name, default value) directo en DEV — misma lógica que `entity-builder`: se itera y prueba en DEV antes de exportar, no se autoría a mano en XML |
| `generate_deployment_settings`   | Git | Ejecuta `pac solution create-settings` sobre el export de la solución y produce el JSON base con los *valores* por ambiente |
| `set_settings_value`         | Git | Completa el valor de una Environment Variable o Connection Reference para un ambiente específico dentro del JSON — nunca en texto plano si es secreto |
| `generate_sdd`               | — | Compila el historial de `.d365-session.md` del Cowork Project + los PRs asociados en un Solution Design Document |

> **Distinción clave:** la *definición* de la variable (qué existe) es metadata como cualquier
> columna — vive en DEV, se prueba en DEV. El *valor* por ambiente (qué vale en TEST vs PROD)
> es lo que va por el deployment settings file vía Git/pipeline. No confundir ambos pasos.

> Eliminados respecto a v1.0.0: `set_env_variable_value` (POST directo a
> `environmentvariablevalues`) y `export_solution`/`read_blob` — la exportación vive ahora en
> `solution-packager`, canal Pipeline.

---

## 1. Environment Variables — flujo v2.0.0

Las Environment Variables permiten que la misma solución funcione con valores distintos por
ambiente (DEV/TEST/PROD) sin modificar la solución.

### Tipos de Environment Variables

| Tipo                 | `type` value | Uso típico                                        |
|-------------------------|----------------|---------------------------------------------------------|
| String               | `100000000`  | URLs, nombres, configuraciones                    |
| Number               | `100000001`  | Umbrales, timeouts, límites                       |
| Boolean              | `100000002`  | Feature flags                                     |
| JSON                 | `100000003`  | Configuraciones complejas en objeto               |
| Data Source          | `100000004`  | Conexiones de Power Apps / Power Automate         |
| Secret               | `100000005`  | API Keys, connection strings (almacenadas en Key Vault) |

### Paso 1 — Definición (canal DEV / Web API, no Git)

La *definición* de la variable (schema name, tipo, display name, default value) se crea
directo contra DEV, igual que una columna — permite probar que la variable resuelve bien en
el contexto de la solución antes de exportar. Cuando `solution-packager` exporta la solución
(`export_solution_to_repo`), esta definición viaja automáticamente como componente, sin que
`config-generator` tenga que generarla a mano en XML:

```json
// POST /api/data/v9.2/environmentvariabledefinitions — contra DEV
{
  "schemaname": "axx_ApiBaseUrl",
  "displayname": "API Base URL del Motor IA",
  "description": "URL base de la Azure Function del Motor IA Documental",
  "type": 100000000,
  "defaultvalue": "https://func-neural-cortex-dev.azurewebsites.net"
}
```

### Paso 2 — Deployment Settings file (`generate_deployment_settings`)

Después de exportar la solución (`solution-packager` → `export_solution_to_repo`), se corre:

```bash
pac solution create-settings --solution-zip ./export/ClienteXCreditOnboarding.zip --settings-file ./deployment-settings.json
```

Esto genera el esqueleto:

```json
{
  "EnvironmentVariables": [
    { "SchemaName": "axx_ApiBaseUrl", "Value": "" },
    { "SchemaName": "axx_ScoreMinimo", "Value": "" },
    { "SchemaName": "axx_FeatureFlagKYCStrict", "Value": "" }
  ],
  "ConnectionReferences": [
    { "LogicalName": "axx_sharedcommondataserviceforapps", "ConnectionId": "", "ConnectorId": "/providers/Microsoft.PowerApps/apis/shared_commondataserviceforapps" }
  ]
}
```

### Paso 3 — Completar valores por ambiente (`set_settings_value`)

La skill genera **un archivo por ambiente target** (`deployment-settings-test.json`,
`deployment-settings-prod.json`), completando los valores no sensibles directamente y
referenciando Key Vault para los secretos:

```json
{
  "EnvironmentVariables": [
    { "SchemaName": "axx_ApiBaseUrl", "Value": "https://func-neural-cortex-test.azurewebsites.net" },
    { "SchemaName": "axx_ScoreMinimo", "Value": "650" },
    { "SchemaName": "axx_FeatureFlagKYCStrict", "Value": "true" }
  ],
  "ConnectionReferences": [
    { "LogicalName": "axx_sharedcommondataserviceforapps", "ConnectionId": "@KeyVault(kv-test/conn-cds)", "ConnectorId": "/providers/Microsoft.PowerApps/apis/shared_commondataserviceforapps" }
  ]
}
```

> Los `ConnectionId` reales normalmente requieren autenticación interactiva la primera vez que
> se crea la conexión en el ambiente target — la skill deja el placeholder de Key Vault y
> documenta en el deployment checklist que ese paso requiere intervención manual una vez.

### Paso 4 — Commit y consumo por el pipeline

Los archivos de settings se commitean al mismo branch que el resto de la solución. El
pipeline de `solution-packager` los consume directo:

```bash
pac solution import --path ./build/solution_managed.zip --settings-file ./deployment-settings-test.json --environment test
```

---


## 2. Solution Design Document (SDD)

Al final de una sesión de diseño, o cuando el usuario lo pide, la skill genera el SDD
compilando **tanto** el historial de `.d365-session.md` (contexto conversacional) **como** los PRs y
runs de pipeline asociados a la sesión (estado real).

```markdown
# Solution Design Document
## {Nombre del Proyecto}
**Fecha:** {fecha}
**Environment DEV:** {url}
**Solución:** {uniquename} v{version}
**PR de promoción:** {link al PR de Azure DevOps}
**Último pipeline run:** {estado + link}

---

## 1. Modelo de datos
### Tablas custom creadas (canal DEV / Web API)
| Tabla (logical name)       | Display Name             | Ownership      | Has Activities |
|---------------------------------|--------------------------------|------------------|-------------------|
| axx_creditsolicitud       | Solicitud de Crédito     | UserOwned      | Sí             |

## 2. Formularios (canal DEV / Web API)
| Tabla                       | Form Name                        | Tipo  | Tabs                          |
|----------------------------------|----------------------------------------|---------|-------------------------------------|
| axx_creditsolicitud        | Solicitud de Crédito - Main      | Main  | General, Scoring, Actividades |

## 3. Business Rules (canal DEV / Web API)
| # | Nombre                                   | Tabla                   | Scope  | Activada |
|-----|------------------------------------------------|---------------------------|----------|-------------|
| 1 | BR - Monto requerido si estado Enviada   | axx_creditsolicitud   | Entity | Sí       |

## 4. Seguridad (canal Git — ver PR)
### Security Roles
| Rol                     | Scope de datos | PR         |
|-----------------------------|-------------------|--------------|
| Axx - Analista Crédito | Basic (Own)   | {link}     |

## 5. Environment Variables / Connection References (canal Git)
| Schema Name             | Tipo   | DEV                          | TEST                    | PROD |
|------------------------------|----------|------------------------------------|-----------------------------|--------|
| axx_ApiBaseUrl        | String | func-...-dev.azurewebsites.net | func-...-test... | func-...-prod... |

## 6. Historial de la sesión
{historial de .d365-session.md serializado como tabla}

## 7. Estado del pipeline de promoción
{lista de runs con estado y links}

## 8. Próximos pasos
- [ ] Configurar vistas (Views) para cada tabla
- [ ] Ejecutar pruebas funcionales en DEV
- [ ] Aprobar PR de seguridad y settings
- [ ] Disparar promoción a TEST
```

---

## 3. Deployment Checklist

```markdown
# Deployment Checklist — {Solución} a {Ambiente Target}
**Fecha de deploy:** {fecha}
**Responsable:** {nombre}
**PR:** {link}
**Pipeline run:** {link}

## Pre-deploy
- [ ] Versión de solución actualizada a {nueva versión}
- [ ] PR de solución + seguridad + settings mergeado
- [ ] `pac solution check` sin errores críticos
- [ ] Deployment settings file completo para {ambiente target} (sin valores vacíos)
- [ ] Connection References resueltas manualmente si es la primera vez en este ambiente
- [ ] Gate de aprobación del pipeline asignado al responsable correcto

## Deploy
- [ ] Pipeline run disparado y en curso
- [ ] Import de solución managed completado sin errores
- [ ] Business Rules activadas en {ambiente target}
- [ ] Customizaciones publicadas (Publish All, dentro del propio import)

## Post-deploy
- [ ] Smoke test de los formularios principales
- [ ] Verificar Business Rules con casos de prueba
- [ ] Verificar acceso por rol con usuario de prueba
- [ ] Log de import revisado (sin errores ni warnings críticos)
- [ ] Comunicación a usuarios finales enviada
```

---

## Restricciones

- No generar archivos de settings que incluyan tokens, passwords o secretos en texto plano —
  siempre referenciar Key Vault (`@KeyVault(vault/secret)`).
- El SDD se genera al final de la sesión o bajo demanda explícita — no interrumpir el flujo
  de trabajo para documentar en tiempo real.
- Ya no se generan specs OpenAPI/Custom Connector para el MCP Server — el conector de Cowork
  descubre las tools vía protocolo MCP nativo, sin necesidad de mantener una spec aparte
  (ver ADR-003).
- El deployment settings file de **PROD** nunca se genera con valores reales hasta el momento
  de la promoción efectiva — se mantiene con placeholders de Key Vault hasta que el gate de
  aprobación de PROD está confirmado.
