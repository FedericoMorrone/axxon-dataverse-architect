---
name: view-designer
description: >
  Crea y modifica Views (SavedQuery) de tablas en Dataverse / Dynamics 365 CE — vistas
  públicas, Quick Find, Advanced Find, Associated (subgrid) y Lookup — construyendo FetchXML y
  LayoutXML vía Web API contra DEV. Usar esta skill cuando el usuario pida crear o modificar
  una vista o view, agregar una columna a la vista, cambiar el orden de una vista, filtrar una
  vista, crear una vista de búsqueda rápida (Quick Find) o avanzada (Advanced Find), configurar
  una vista asociada o subgrid, una vista de lookup, preguntar qué vistas tiene una tabla, o
  definir la vista por defecto. Usar junto con la skill dataverse-architect.
---

# View Designer — Axxon Dataverse Architect

Skill especializada en la creación y modificación de **Views** (entidad de sistema
`savedquery`) en Dataverse / Dynamics 365 CE. Las views definen qué columnas, orden y filtros
ve el usuario al listar registros — en grids principales, subgrids, Advanced Find, y campos
de búsqueda (lookup).

---

## MCP Tools disponibles

| Tool name            | Método HTTP | Endpoint Dataverse                                                        |
|----------------------|-------------|-----------------------------------------------------------------------------|
| `list_views`         | GET         | `/api/data/v9.2/savedqueries?$filter=returnedtypecode eq 'xxx'`             |
| `get_view`            | GET         | `/api/data/v9.2/savedqueries(<savedqueryid>)`                               |
| `create_view`        | POST        | `/api/data/v9.2/savedqueries`                                               |
| `patch_view`         | PATCH       | `/api/data/v9.2/savedqueries(<savedqueryid>)`                               |
| `publish_view`       | POST        | `/api/data/v9.2/PublishXml`                                                 |
| `set_default_view`   | POST        | `/api/data/v9.2/SetIsDefault` (una sola vista pública por tabla puede ser default) |
| `add_to_solution`    | POST        | `/api/data/v9.2/AddSolutionComponent` (ComponentType: 26)                   |

---

## Tipos de vista (`querytype`)

| `querytype` | Tipo             | Dónde se usa                                                    |
|---------------|--------------------|------------------------------------------------------------------------|
| `0`         | Public (Main)     | Grid principal de la tabla, selector de vistas del usuario     |
| `1`         | Advanced Find      | Selector de columnas/resultados en Advanced Find                |
| `2`         | Associated (Grid)  | Subgrid que muestra registros relacionados dentro de otro form  |
| `4`         | Quick Find         | Motor de búsqueda por texto en el selector de registros          |
| `8`         | Lookup             | Resultados que aparecen al buscar en un campo lookup             |

> Una vista **personal** (`userquery`) es una entidad distinta — normalmente no se gestiona
> desde este agente porque pertenece al usuario final, no a la solución. Si el pedido es
> "cada usuario arma su propia vista", aclarar que eso es nativo de D365 y no requiere
> configuración vía esta skill.

---

## Arquitectura de una View

Una view combina dos XML dentro del mismo registro `savedquery`:

```
savedquery
  ├── fetchxml    → QUÉ datos trae: entidad, filtros, orden, joins
  └── layoutxml   → CÓMO se muestran: columnas, anchos, ícono
```

### FetchXML — estructura mínima

```xml
<fetch version="1.0" mapping="logical">
  <entity name="axx_creditsolicitud">
    <attribute name="axx_name" />
    <attribute name="axx_montosolicitado" />
    <attribute name="axx_estadosolicitud" />
    <attribute name="createdon" />
    <order attribute="createdon" descending="true" />
    <filter type="and">
      <condition attribute="statecode" operator="eq" value="0" />
    </filter>
    <link-entity name="contact" from="contactid" to="axx_cliente" link-type="outer" alias="cliente">
      <attribute name="fullname" />
    </link-entity>
  </entity>
</fetch>
```

### LayoutXML — estructura mínima (debe reflejar exactamente los atributos del FetchXML)

```xml
<grid name="resultset" object="1" jump="axx_name" select="1" icon="1" preview="1">
  <row name="result" id="axx_creditsolicitudid">
    <cell name="axx_name" width="200" />
    <cell name="axx_montosolicitado" width="120" />
    <cell name="axx_estadosolicitud" width="150" />
    <cell name="cliente.fullname" width="180" />
    <cell name="createdon" width="120" />
  </row>
</grid>
```

> **Regla crítica:** todo atributo que aparece en `layoutxml` como `<cell>` **debe** existir
> como `<attribute>` en el `fetchxml` correspondiente (directo o vía `link-entity` con alias).
> Si no coinciden, Dataverse rechaza el `create_view`/`patch_view` con un error de validación
> de columnas — es la causa más común de fallos en esta skill.

---

## Crear una vista pública (`create_view`)

### Inputs requeridos

| Input            | Descripción                                    | Ejemplo                          |
|---------------------|------------------------------------------------------|---------------------------------------|
| `tableName`       | Logical name de la tabla                       | `axx_creditsolicitud`           |
| `viewName`        | Nombre de la vista                              | `Solicitudes Activas`            |
| `queryType`        | Ver tabla de tipos arriba                        | `0` (Public)                      |
| `columns[]`        | Lista de columnas a mostrar, con ancho sugerido | Ver LayoutXML arriba              |
| `filters[]`        | Condiciones en lenguaje natural, se traducen a `<filter>` | "estado activo"          |
| `sortBy`            | Columna y dirección de orden por defecto         | `createdon desc`                  |
| `isDefault`        | Si reemplaza a la vista pública por defecto      | `false`                            |

### Payload

```json
{
  "name": "Solicitudes Activas",
  "returnedtypecode": "axx_creditsolicitud",
  "querytype": 0,
  "fetchxml": "<fetch>...</fetch>",
  "layoutxml": "<grid>...</grid>",
  "isdefault": false
}
```

Después de crear, siempre:
```json
// add_to_solution
{ "ComponentId": "<savedqueryid>", "ComponentType": 26, "SolutionUniqueName": "ClienteXCreditOnboarding" }
// publish_view
{ "ParameterXml": "<importexportxml><entities><entity>axx_creditsolicitud</entity></entities></importexportxml>" }
```

---

## Vista Associated (subgrid) — caso frecuente en FSI

Cuando `form-designer` configura un subgrid (ver `02-form-designer.md`), esa subgrid puede
apuntar a una vista `querytype: 2` específica en vez de la vista pública por defecto —
típicamente para mostrar menos columnas o un filtro distinto en el contexto del form padre:

```json
{
  "name": "Garantías Asociadas - Vista de Solicitud",
  "returnedtypecode": "axx_garantia",
  "querytype": 2,
  "fetchxml": "<fetch><entity name='axx_garantia'><attribute name='axx_name'/><attribute name='axx_valorestimado'/><order attribute='axx_valorestimado' descending='true'/></entity></fetch>",
  "layoutxml": "<grid name='resultset' object='1' jump='axx_name' select='1' icon='1' preview='1'><row name='result' id='axx_garantiaid'><cell name='axx_name' width='200'/><cell name='axx_valorestimado' width='150'/></row></grid>"
}
```

El `ViewId` de esta vista es el que se referencia en el `<parameters><ViewId>` del control
subgrid en el FormXml.

---

## Modificación de vista existente (`patch_view`)

1. `get_view` para obtener el `fetchxml`/`layoutxml` actual.
2. Parsear, agregar/quitar `<attribute>` (fetch) y su `<cell>` correspondiente (layout) en el
   mismo paso — nunca uno sin el otro.
3. `patch_view` con ambos XML actualizados.
4. `publish_view` — sin esto los cambios no son visibles.

### Operaciones frecuentes

| Pedido del usuario                        | Qué modifica la skill                                               |
|-----------------------------------------------|---------------------------------------------------------------------------|
| "Agregá la columna de estado a la vista"    | `<attribute>` en fetchxml + `<cell>` en layoutxml, mismo orden relativo |
| "Ordená por monto descendente"              | `<order attribute="axx_montosolicitado" descending="true" />`         |
| "Mostrame solo las solicitudes del último mes" | `<filter>` con `<condition operator="last-x-months" value="1" />`  |
| "Traé el nombre del cliente en la vista"    | `<link-entity>` a `contact` con alias + `<attribute>` + `<cell name="alias.campo">` |
| "Hacé esta vista la vista por defecto"      | `set_default_view` — solo una vista pública puede ser default por tabla |

---

## Operadores de filtro FetchXML frecuentes

| Operador          | `operator` value | Uso típico                                  |
|----------------------|---------------------|---------------------------------------------------|
| Igual               | `eq`               | Estado, tipo, opciones                      |
| Distinto            | `ne`               | Exclusiones                                  |
| Mayor / menor que    | `gt` / `lt`        | Montos, fechas                              |
| Contiene datos       | `not-null`          | Campos completados                          |
| No contiene datos    | `null`              | Campos vacíos — útil para detectar incompletos |
| Últimos X días/meses | `last-x-days` / `last-x-months` | Reportes de actividad reciente |
| En (lista de valores)| `in`                | Múltiples estados a la vez                   |

---

## Buenas prácticas de diseño de vistas

| Práctica                                                | Razón                                                   |
|--------------------------------------------------------------|-----------------------------------------------------------|
| No más de 8-10 columnas en una vista pública            | Legibilidad en pantallas estándar                        |
| Ordenar por fecha de creación/modificación por defecto  | Comportamiento esperado por el usuario de negocio         |
| Vistas Associated con menos columnas que la vista pública | El subgrid tiene menos espacio horizontal                |
| Usar Quick Find solo con columnas indexadas/buscables     | Rendimiento — evitar Quick Find sobre Memo largos          |
| Una sola vista pública `isdefault: true` por tabla        | Más de una genera comportamiento errático                 |

---

## Confirmar la solución activa antes de agregar el componente

Antes de llamar `add_to_solution`, confirmá cuál es la solución correcta — nunca asumas el
nombre de un ejemplo de esta documentación ni tomes la primera que aparezca:

1. Buscá el campo `Solución` en `.d365-session.md` (el conductor `dataverse-architect` lo
   mantiene) — si está seteado, usalo directo, no vuelvas a preguntar.
2. Si el archivo no existe, el campo está vacío, o hay motivo para pensar que puede haber más
   de una solución candidata en este environment, llamá `list_solutions` (de
   `solution-packager`) y preguntale al usuario explícitamente cuál corresponde — nunca
   elijas por tu cuenta ni asumas la primera de la lista.
3. Los nombres de solución en los ejemplos de este documento (`ClienteXCreditOnboarding`,
   `AxxonClienteXCreditOnboarding`) son ilustrativos — nunca los uses como si fueran reales.

---

## Restricciones

- **Nunca** modificar vistas de entidades OOB (Account, Contact, Lead, etc.) sin verificar que
  la vista es un componente no administrado de la solución del proyecto.
- **Siempre** mantener sincronizados `fetchxml` y `layoutxml` — un `<cell>` sin su
  `<attribute>` correspondiente rompe la vista al renderizar.
- **Siempre** publicar (`publish_view`) después de cada modificación.
- No usar `link-entity` para atravesar más de 2 niveles de relación en vistas públicas — impacta
  performance de forma significativa en tablas con volumen alto (típico en banca: miles de
  solicitudes). Para reportes más profundos, derivar a Power BI / dataflows en vez de vistas.