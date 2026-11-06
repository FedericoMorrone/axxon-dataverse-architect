# Canales de ejecución — detalle (ADR-001)

La tabla compacta de routing está en `SKILL.md`. Acá, lo que cada canal implica.

| Canal | Cuándo aplica | Skill(s) |
|---|---|---|
| **DEV / Web API** | Crear o iterar: tablas, columnas, relaciones, forms, business rules, vistas, alternate keys, duplicate detection rules, definición de environment variables, App module/Sitemap/Dashboards/Charts, definición del contrato de un Custom API (metadata, no la implementación) — siempre contra el environment DEV del proyecto activo, vía el D365 Architect MCP Server (Connector MCP en Cowork, agregado en Customize → Connectors) | `entity-builder`, `form-designer`, `business-rule-engine`, `view-designer`, `duplicate-detection`, `config-generator` (definición), `app-composer`, `custom-api-builder` |
| **Git** | Seguridad (roles, BU, teams, field security), valores de Environment Variables / Connection References por ambiente | `security-architect`, `config-generator` (settings) |
| **SDK Python / PAC CLI directo** | Crear publisher/solución (SDK), agregar componentes, export/unpack/pack de baseline local — **no pasa por nuestro MCP Server propio**, adaptado del enfoque de `microsoft/Dataverse-skills` (`dv-solution`) | `solution-packager` (parte DEV) |
| **Pipeline** | Promoción DEV→TEST, TEST→PROD — **gate humano obligatorio, nunca reemplazado por el canal SDK/PAC CLI** | `solution-packager` (parte Pipeline) |
| **PAC CLI directo** | Generative Pages (React/TS/Fluent) — Preview de Microsoft, deploy directo al environment; la promoción posterior a TEST/PROD pasa por el canal Pipeline | `genpage-builder` |
| **PAC CLI / npx directo** | Power Apps Code Apps — verificar estado GA/Preview y la CLI vigente antes de comprometerse con el cliente | `code-app-builder` |
| **MCP externo (FlowAgent)** | Power Automate cloud flows — servidor MCP oficial de Microsoft, distinto del MCP Server propio de Axxon; verificar que el connector `flowagent` esté agregado antes de asumir disponibilidad | `flow-builder` |
| **Git (código compilado)** | Plugins C#/.NET y código de Azure Functions — en el repo ADO del proyecto, rama por User Story, PR review antes de registrar/desplegar, nunca directo contra DEV | `plugin-builder`, `azure-function-builder`, vía `code-repo-manager` |
| **Azure CLI / Functions Core Tools** | Build y deploy de Azure Functions — DEV directo permitido, PROD **siempre** vía Pipeline con gate humano, contra la suscripción de `.d365-project.md` | `azure-function-builder` |
| **MCP oficial de Microsoft Dataverse** | Consultas/analytics y CRUD sobre datos existentes — distinto del MCP Server propio; instalar solo con confirmación, avisar el costo de Copilot Credits antes de usarlo con un cliente real | `dataverse-connect` |
| **ADO Wiki (MCP oficial de Azure DevOps)** | Registro de ADRs del proyecto en la wiki de proyecto del cliente — documentación, no construcción | `adr-recorder` |
