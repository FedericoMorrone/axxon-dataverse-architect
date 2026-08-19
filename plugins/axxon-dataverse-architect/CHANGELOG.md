# Changelog — Axxon Dataverse Architect

## v1.3.0-rc1 (2026-08-19)

Gap identificado revisando la Fase 2 en detalle: extensibilidad pro-code de Dataverse
(Plugins, Custom APIs) e integración custom en Azure — ninguna de las 13 skills anteriores
lo cubría, ni siquiera la comparación contra el repo de Microsoft (ese repo apunta a
low-code, no a pro-code).

### Agregado
- **`plugin-builder`** — Plugins C#/.NET: mensaje+entidad+etapa (Pre-validation/
  Pre-operation/Post-operation), sync vs. async, filtering attributes, pre/post-images,
  patrón de código estándar (`IPlugin`, chequeo de `context.Depth`, late-bound por default,
  `InvalidPluginExecutionException` para errores esperables). Canal **Git** — código
  compilado, mismo criterio que `security-architect`.
- **`custom-api-builder`** — Definición del contrato de Custom APIs (Action vs. Function,
  binding Global/Entity/Entity Collection, parámetros tipados). Explícitamente **no** genera
  la implementación — coordina con `plugin-builder` (o `flow-builder` si la lógica va en un
  flow). Canal DEV/Web API, igual que `entity-builder`.
- **`azure-function-builder`** — Build y deploy de Azure Functions para integración externa
  (HTTP/Timer/Service Bus triggers). Lee la suscripción de Azure desde `.d365-project.md` —
  nunca la asume. **Deploy a PROD exige el mismo gate de aprobación humana que
  `solution-packager`** (decisión explícita, confirmada antes de construir) — DEV directo sí
  está permitido. Nuevo canal: **Azure CLI / Functions Core Tools**.
- `dataverse-architect` (conductora) actualizada con las 3 skills nuevas, 2 canales nuevos en
  el routing (Git para código compilado, Azure CLI/Functions Core Tools).
- **Nueva sección en `dataverse-architect`**: análisis de alcance obligatorio al recibir una
  User Story completa, antes de rutear a la primera skill obvia — detecta explícitamente la
  cadena Custom API → Plugin → Azure Function cuando la Historia la necesita, y exige mostrar
  el plan completo al usuario antes de empezar a construir. Reforzado también en
  `plugin-builder` (chequeo explícito de integración externa antes de escribir el código), para
  que la disciplina se sostenga aunque se invoque la skill directo, sin pasar por el conductor.

## v1.2.0-rc1 (2026-08-18)

Se agregan 2 skills más, siguiendo con la comparación contra
[`microsoft/power-platform-skills`](https://github.com/microsoft/power-platform-skills) — esta
vez a pedido explícito, cubriendo 2 de los otros 6 plugins del repo que originalmente se
habían descartado por estar "fuera del alcance de Dataverse model-driven".

### Agregado
- **`flow-builder`** — Power Automate cloud flows: crear, editar (quirúrgico, a nivel de
  acción), copiar, publicar/deshabilitar, y gestión de runs (historial, cancelar, reenviar,
  diagnosticar). Usa el **FlowAgent MCP server oficial de Microsoft** — distinto del D365
  Architect MCP Server propio de Axxon (ADR-002) — la skill verifica explícitamente que el
  connector esté disponible antes de asumir que las tools existen.
- **`code-app-builder`** — Power Apps Code Apps (React + Vite + TypeScript, 1500+ connectors,
  autenticación Entra). Incluye una regla dura real de Microsoft (scaffolding siempre con
  `npx degit`, nunca `git clone` ni archivos a mano) y una nota honesta: las fuentes sobre
  estado GA vs. Preview de esta feature eran **contradictorias** al momento de escribir la
  skill — se documentó la discrepancia en vez de elegir una al azar.
- 2 canales nuevos en el routing de `dataverse-architect`: **PAC CLI / npx directo**
  (`code-app-builder`) y **MCP externo** (`flow-builder`, FlowAgent) — ambos explícitamente
  distintos de nuestro propio MCP Server por cliente.

### Documentado (sin cambio funcional todavía)
- **`ADR-004-mcp-apps-adoption.md` (Aceptado)** — al comparar el modelo de ejecución contra
  el repo de Microsoft, se encontró el protocolo MCP Apps (vistas previas interactivas en
  tool results, no una alternativa de performance). Se diseñó un MVP de 3 widgets
  (`form-designer`, `view-designer`, `app-composer`), stack resuelto: **HTML/CSS/JS vanilla,
  sin React** — el Azure Function no tiene pipeline de frontend hoy, y el contenido
  (layouts, grillas, árboles de sitemap) no necesita el modelo de componentes de React.
  Implementación todavía pendiente.

## v1.1.0-rc1 (2026-08-18)

Se agregan 2 skills nuevas tras comparar las 9 existentes contra el repo oficial de Microsoft
[`microsoft/power-platform-skills`](https://github.com/microsoft/power-platform-skills) (plugin
`model-apps`) — encontramos 2 capacidades reales que no cubríamos.

### Agregado
- **`app-composer`** — App module, Sitemap (Area/Group/SubArea), Dashboards, y Charts. Canal
  DEV/Web API, mismo patrón que el resto del paquete (MCP tools, publish explícito al final).
- **`genpage-builder`** — Generative Pages (React 17 + TypeScript + Fluent UI V9, corren
  nativas dentro del shell de la app sin iframe). Adaptado del enfoque oficial de Microsoft
  (`/genpage` en `model-apps`), incluyendo un gotcha técnico real documentado por Microsoft
  (caché de módulo que se resetea en cada navegación — fix con `window.__pp<Entity>Cache`).
  **Es Preview de Microsoft, no GA** — la skill exige confirmar esto con el cliente antes de
  usarla, no lo asume.
- Nuevo canal en el routing de `dataverse-architect`: **PAC CLI directo** (solo
  `genpage-builder`) — distinto de DEV/Web API (que usa MCP tools del servidor por cliente).

### No se tocó (evaluado y descartado como gap)
- `solution-packager`, `duplicate-detection`, y el modelo de 3 canales con gates humanos —
  Microsoft no los cubre con la misma profundidad ALM; nuestro paquete queda más maduro ahí,
  no hacía falta adaptar nada de ese lado.

## v1.0.0-rc1 (2026-08-05)

Primer release candidate del paquete migrado a Claude Cowork. Consolida el trabajo de
revisión arquitectónica documentado en `docs/ADR-001` a `docs/ADR-003`.

### Cerrado en este RC
- Migración completa de las 9 skills al formato real de Claude Skills (carpeta + `SKILL.md`,
  frontmatter `name`+`description`).
- Pivot de plataforma: Copilot Studio → Claude Cowork (ver ADR-003).
- Modelo de ejecución por canal: DEV/Web API, Git, Pipeline (ver ADR-001).
- Topología de MCP Server: una instancia por cliente (ver ADR-002).
- Prefijo de publisher unificado a `axx_` en todo el paquete.
- Skills nuevas: `view-designer`, `duplicate-detection`.
- Retirada la tabla `axx_designsession` — el contexto conversacional vive en Cowork Projects
  (`.d365-session.md`).
- Descartada la dependencia de la skill `output-formatter` (Copilot Studio / Adaptive Cards).

### Conocido / pendiente (no bloqueante para RC1)
- Verificar contra `$metadata` de un environment real los códigos de `ComponentType` para
  `Security Role`, `Alternate Key` y `Duplicate Detection Rule` (ver `docs/ADR-002`, nota de
  `04-solution-packager`/`solution-packager`).
- Diseñar el pipeline "meta" que propague actualizaciones de la plantilla del MCP Server a
  todas las instancias de clientes activos (ver `docs/ADR-002`).
- Confirmar restricciones de red de Cowork para sesiones en la nube contra el MCP Server (ver
  `docs/ADR-003`).

---

## Historial previo (pre-repo)

Registrado acá por trazabilidad; no corresponde a commits reales de este repositorio.

- **v2.2.0** — Topología del MCP Server (ADR-002): una instancia por cliente, Azure DevOps
  Project dedicado por cliente.
- **v2.1.0** — Correcciones de una validación end-to-end (canal de Environment Variables,
  confirm-before-execute para skills nuevas, advertencia sobre ComponentType sin verificar).
- **v2.0.0** — Pivot de ejecución (ADR-001): de Web API contra cualquier environment a modelo
  híbrido DEV/Git/Pipeline. Prefijo unificado a `axx_`. Skills nuevas: `view-designer`,
  `duplicate-detection`.
- **v1.0.0** — Paquete original sobre Copilot Studio + Azure Function MCP Server con Web API
  directa contra todos los environments, incluido PROD.
