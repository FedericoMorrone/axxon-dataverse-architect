---
name: app-composer
description: >
  Configura el módulo de App (App Module), su Sitemap (navegación: Area > Group > SubArea), y
  Dashboards/Charts en Dataverse / Dynamics 365 CE vía Web API contra DEV. Usar cuando el
  usuario pida crear o modificar una app model-driven, agregar una tabla/página al sitemap,
  reorganizar la navegación de la app, crear un dashboard o un chart, o preguntar qué apps
  existen en el environment. No usar para el contenido de tablas/forms/vistas individuales
  (eso ya lo cubren `entity-builder`/`form-designer`/`view-designer`) ni para páginas custom
  con código (eso es `genpage-builder`). Usar junto con la skill dataverse-architect.
---

# App Composer — Axxon Dataverse Architect

Configurás la **capa de app completa** — qué tablas/páginas ve el usuario, cómo navega entre
ellas, y qué dashboards/charts tiene disponibles. A diferencia de `entity-builder`/
`form-designer`/`view-designer` (que trabajan tabla por tabla), esta skill trabaja a nivel de
**la app model-driven como un todo**.

---

## MCP Tools disponibles

| Tool name | Método HTTP | Endpoint Dataverse |
|---|---|---|
| `list_apps` | GET | `/api/data/v9.2/appmodules` |
| `get_app` | GET | `/api/data/v9.2/appmodules(<appmoduleid>)` |
| `create_app` | POST | `/api/data/v9.2/appmodules` |
| `patch_app` | PATCH | `/api/data/v9.2/appmodules(<appmoduleid>)` |
| `add_component_to_app` | POST | `/api/data/v9.2/AddAppComponents` |
| `get_sitemap` | GET | `/api/data/v9.2/sitemaps(<sitemapid>)` |
| `update_sitemap` | PATCH | `/api/data/v9.2/sitemaps(<sitemapid>)` (campo `sitemapxml`) |
| `publish_app` | POST | `/api/data/v9.2/PublishXml` |
| `create_dashboard` | POST | `/api/data/v9.2/systemforms` (`type` = Dashboard) |
| `create_chart` | POST | `/api/data/v9.2/savedqueryvisualizations` |

---

## App Module

Una app model-driven es un `appmodule` — el contenedor de todo lo demás. Al crear una app
nueva, confirmá con el usuario: nombre, descripción, y si arranca vacía o clonando la
estructura de otra app existente. Nunca la crees sin confirmar el nombre — es lo primero que
ve el cliente al abrir Dynamics 365.

Los componentes (tablas, dashboards, forms, vistas) se agregan a la app vía
`add_component_to_app` — una tabla no aparece en la app solo por existir en el environment,
tiene que agregarse explícitamente.

## Sitemap — Area > Group > SubArea

El sitemap define la navegación. Estructura de 3 niveles:

- **Area** — el nivel más alto (ej. "Ventas", "Atención al Cliente").
- **Group** — agrupa SubAreas dentro de un Area (ej. "Mis Casos").
- **SubArea** — el link real a una tabla, dashboard, o URL externa.

Se edita como XML (`sitemapxml`) — nunca reescribas el sitemap completo si el pedido es
agregar una SubArea puntual; traé el XML actual con `get_sitemap`, modificalo, y hacé
`update_sitemap` con el resultado. Reescribir todo el árbol arriesga perder entradas que el
usuario no mencionó.

```xml
<SiteMap>
  <Area Id="area_atencion_cliente" ResourceId="Area_AtencionCliente" ShowGroups="true">
    <Group Id="group_casos" ResourceId="Group_MisCasos">
      <SubArea Id="subarea_casos" Entity="incident" />
    </Group>
  </Area>
</SiteMap>
```

## Dashboards

Un dashboard es un `systemform` con `type` = Dashboard (11) — se define con layout XML propio
(no confundir con Form XML de una tabla individual). Antes de crear uno, confirmá qué
componentes va a tener (charts, listas, iframes) — no generes un dashboard vacío esperando
que el usuario lo complete después sin saber qué iba adentro.

## Charts

Un chart es un `savedqueryvisualization`, vinculado a una vista (`savedquery`) existente de
la tabla — necesita que la vista ya exista (coordiná con `view-designer` si falta). El XML de
visualización (`visualizationxml`) define tipo de gráfico (barras, torta, línea, etc.), eje, y
agrupación.

## Después de cualquier cambio — publicar

`create_app`/`update_sitemap`/`create_dashboard`/`create_chart` no se ven reflejados para el
usuario hasta correr `publish_app` — no des el trabajo por terminado sin este paso, y
avisale al usuario que puede tardar unos segundos en propagarse.

## Restricciones

- Nunca reescribas el sitemap completo para un cambio puntual — traé el XML actual primero.
- Nunca crees una app sin confirmar el nombre con el usuario.
- Nunca agregues una tabla a la app asumiendo que "obviamente" tiene que estar — confirmá el
  alcance contra el Kickoff Brief/backlog de Fase 1 si hay dudas.
- Nunca olvidés `publish_app` al final — es un paso real, no cosmético.
- Un chart necesita una vista existente — no la inventes, coordiná con `view-designer`.
