---
name: form-designer
description: >
  Diseña y modifica formularios (Main, Quick View, Quick Create, Card) de tablas en Dataverse /
  Dynamics 365 CE, construyendo el FormXml vía Web API contra DEV. Usar esta skill cuando el
  usuario pida crear o diseñar un formulario o form, agregar una pestaña o tab, agregar una
  sección, agregar un campo al formulario, ocultar o mostrar un campo, mover un campo en el
  form, configurar visibilidad, crear un quick view form o quick create form, modificar el
  layout de un formulario, o pregunte por el formulario de una tabla específica. Usar junto
  con la skill dataverse-architect.
---

# Form Designer — Axxon Dataverse Architect

Skill especializada en el diseño y modificación de **formularios** en Dataverse / Dynamics 365 CE.
Los formularios se almacenan como XML (`FormXml`) en la tabla de sistema `systemform`.
Esta skill construye y modifica ese XML de forma programática.
Siempre usa el MCP Dataverse Disponible en el agente.

---

## MCP Tools disponibles

| Tool name            | Método HTTP | Endpoint Dataverse                                                        |
|----------------------|-------------|---------------------------------------------------------------------------|
| `list_forms`         | GET         | `/api/data/v9.2/systemforms?$filter=objecttypecode eq 'xxx'`              |
| `get_form`           | GET         | `/api/data/v9.2/systemforms(<formid>)`                                    |
| `create_form`        | POST        | `/api/data/v9.2/systemforms`                                              |
| `patch_form`         | PATCH       | `/api/data/v9.2/systemforms(<formid>)`                                    |
| `publish_form`       | POST        | `/api/data/v9.2/PublishXml` (con `ParameterXml` apuntando al form)        |
| `add_to_solution`    | POST        | `/api/data/v9.2/AddSolutionComponent` (ComponentType: 24)                 |

---

## Tipos de formularios

| `type` value | Tipo           | Cuándo usar                                              |
|--------------|----------------|----------------------------------------------------------|
| `2`          | Main           | Formulario principal de la tabla — el más completo       |
| `6`          | Quick View     | Vista embebida de un registro relacionado en otro form   |
| `7`          | Quick Create   | Form simplificado para creación rápida (lateral)         |
| `11`         | Card           | Vista en tablero Kanban                                  |

---

## Arquitectura del FormXml

Un formulario de D365 CE tiene la siguiente jerarquía:

```
<form>
  └── <tabs>
        └── <tab> (una o más pestañas)
              ├── <labels>
              └── <columns>
                    └── <column>
                          └── <sections>
                                └── <section>
                                      ├── <labels>
                                      └── <rows>
                                            └── <row>
                                                  └── <cell>
                                                        └── <control> (campo, subgrid, etc.)
```

### Atributos críticos del FormXml

| Elemento   | Atributo clave     | Descripción                                               |
|------------|--------------------|-----------------------------------------------------------|
| `<form>`   | `name`             | Nombre del form                                           |
| `<tab>`    | `id`, `name`, `visible` | GUID, nombre interno, visibilidad inicial           |
| `<section>`| `id`, `name`, `columns`  | GUID, nombre, número de columnas de layout (1-3)   |
| `<cell>`   | `id`               | GUID único por celda                                      |
| `<control>`| `datafieldname`, `classid` | Logical name del campo y classid del control type |

---

## Class IDs de controles frecuentes

| Tipo de control          | `classid`                                      |
|--------------------------|------------------------------------------------|
| Text / Number / DateTime | `{4273B735-4538-468D-838D-C2D56C689AD5}`       |
| OptionSet / Choice       | `{3EF39988-22BB-4F0B-BBBE-64B5A3748AEE}`       |
| Lookup                   | `{270BD3DB-D9AF-4782-9025-509E298DEC0A}`        |
| Boolean / Two Options    | `{67FAC785-CD58-4F9F-ABB3-4B7DDC6ED5ED}`       |
| Memo / Text Area         | `{E0DECE4B-6FC8-4a8f-A065-082708572369}`        |
| Subgrid                  | `{E7A81278-8635-4d9e-8D4D-59480B391C5B}`        |
| Quick View Form          | `{3EF39988-22BB-4F0B-BBBE-64B5A3748AEE}`       |
| Web Resource             | `{9FDF5F91-88B1-47f4-AD53-C11EFC01A01D}`        |
| Timeline (Activities)    | `{5C5600E0-1D6E-4205-A272-BE80DA87FD42}`        |

---

## Flujo para crear un formulario nuevo

### Paso 1 — Inputs del usuario

Solicitá al usuario:

| Input                  | Descripción                                           | Ejemplo                      |
|------------------------|-------------------------------------------------------|------------------------------|
| `tableName`            | Logical name de la tabla                              | `axx_creditsolicitud`      |
| `formName`             | Nombre del formulario                                 | `Solicitud de Crédito - Main`|
| `formType`             | Tipo de form (Main, Quick Create, etc.)               | `Main`                       |
| `tabs[]`               | Lista de tabs con nombre y secciones                  | Ver estructura abajo         |

### Paso 2 — Construir el FormXml

Construís el XML del formulario. Ejemplo mínimo con un tab y dos secciones:

```xml
<form>
  <tabs>
    <tab name="tab_general" id="{TAB-GUID-1}" visible="true" expanded="true">
      <labels>
        <label description="General" languagecode="3082"/>
      </labels>
      <columns>
        <column width="100%">
          <sections>
            <section name="sec_datos_principales" id="{SEC-GUID-1}" visible="true" showlabel="false" columns="2" layout="varwidth">
              <labels>
                <label description="Datos Principales" languagecode="3082"/>
              </labels>
              <rows>
                <row>
                  <cell id="{CELL-GUID-1}" showlabel="true" rowspan="1" colspan="1">
                    <labels><label description="Número de Solicitud" languagecode="3082"/></labels>
                    <control id="axx_name" classid="{4273B735-4538-468D-838D-C2D56C689AD5}" datafieldname="axx_name" disabled="false"/>
                  </cell>
                  <cell id="{CELL-GUID-2}" showlabel="true" rowspan="1" colspan="1">
                    <labels><label description="Estado de Solicitud" languagecode="3082"/></labels>
                    <control id="axx_estadosolicitud" classid="{3EF39988-22BB-4F0B-BBBE-64B5A3748AEE}" datafieldname="axx_estadosolicitud" disabled="false"/>
                  </cell>
                </row>
                <row>
                  <cell id="{CELL-GUID-3}" showlabel="true" rowspan="1" colspan="2">
                    <labels><label description="Monto Solicitado" languagecode="3082"/></labels>
                    <control id="axx_montosolicitado" classid="{4273B735-4538-468D-838D-C2D56C689AD5}" datafieldname="axx_montosolicitado" disabled="false"/>
                  </cell>
                </row>
              </rows>
            </section>
            <section name="sec_timeline" id="{SEC-GUID-2}" visible="true" showlabel="false" columns="1">
              <labels>
                <label description="Actividades" languagecode="3082"/>
              </labels>
              <rows>
                <row>
                  <cell id="{CELL-GUID-4}" showlabel="false" rowspan="10" colspan="1">
                    <control id="timeline" classid="{5C5600E0-1D6E-4205-A272-BE80DA87FD42}" disabled="false" isrequired="false"/>
                  </cell>
                </row>
              </rows>
            </section>
          </sections>
        </column>
      </columns>
    </tab>
  </tabs>
  <DisplayConditions>
    <role/>
  </DisplayConditions>
  <formLibraries/>
  <events/>
  <clientresources/>
  <header>
    <rows>
      <row>
        <cell id="{HEADER-CELL-1}" showlabel="true" rowspan="1" colspan="1">
          <labels><label description="Número de Solicitud" languagecode="3082"/></labels>
          <control id="axx_name" classid="{4273B735-4538-468D-838D-C2D56C689AD5}" datafieldname="axx_name" disabled="false"/>
        </cell>
        <cell id="{HEADER-CELL-2}" showlabel="true" rowspan="1" colspan="1">
          <labels><label description="Estado" languagecode="3082"/></labels>
          <control id="axx_estadosolicitud" classid="{3EF39988-22BB-4F0B-BBBE-64B5A3748AEE}" datafieldname="axx_estadosolicitud" disabled="false"/>
        </cell>
      </row>
    </rows>
  </header>
  <footer>
    <rows>
      <row/>
    </rows>
  </footer>
</form>
```

> **GUIDs:** Todos los `id` de tabs, secciones y celdas deben ser GUIDs únicos generados por
> la skill. Usá el formato `{xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}` en mayúsculas y con llaves.

### Paso 3 — Payload para `create_form`

```json
{
  "name": "Solicitud de Crédito - Main",
  "description": "Formulario principal para gestión de solicitudes de crédito",
  "type": 2,
  "objecttypecode": "axx_creditsolicitud",
  "formxml": "<form>...</form>",
  "isdefault": true
}
```

### Paso 4 — Agregar a solución y publicar

```json
// add_to_solution
{
  "ComponentId": "<formid retornado>",
  "ComponentType": 24,
  "SolutionUniqueName": "AxxonClienteXCreditOnboarding",
  "AddRequiredComponents": false
}

// publish_form
{
  "ParameterXml": "<importexportxml><entities><entity>axx_creditsolicitud</entity></entities></importexportxml>"
}
```

---

## Modificación de formulario existente (`patch_form`)

Para modificar un form existente:

1. Obtener el form con `get_form` para obtener el `formxml` actual.
2. Parsear el XML, aplicar los cambios (agregar tab/sección/campo, cambiar visibilidad, etc.).
3. Hacer `PATCH` con el `formxml` actualizado.

### Operaciones frecuentes de modificación

#### Agregar un tab nuevo

```xml
<!-- Insertar dentro de <tabs> -->
<tab name="tab_scoring" id="{NEW-TAB-GUID}" visible="true" expanded="false">
  <labels>
    <label description="Scoring Crediticio" languagecode="3082"/>
  </labels>
  <columns>
    <column width="100%">
      <sections>
        <section name="sec_dictamen" id="{NEW-SEC-GUID}" visible="true" showlabel="true" columns="1">
          <labels>
            <label description="Dictamen" languagecode="3082"/>
          </labels>
          <rows>
            <row>
              <cell id="{NEW-CELL-GUID}" showlabel="true" rowspan="1" colspan="1">
                <labels><label description="Resultado" languagecode="3082"/></labels>
                <control id="axx_resultadoscoring" classid="{3EF39988-22BB-4F0B-BBBE-64B5A3748AEE}" datafieldname="axx_resultadoscoring" disabled="false"/>
              </cell>
            </row>
          </rows>
        </section>
      </sections>
    </column>
  </columns>
</tab>
```

#### Ocultar campo condicionalmente

Los campos no tienen visibilidad condicional nativa en el FormXml — eso se maneja vía
**Business Rules** o **JavaScript Web Resources**. Cuando el usuario pide "ocultar el campo X
si el campo Y tiene el valor Z", derivá a `business-rule-engine`.

Sin embargo, podés ocultar un campo **incondicionalmente** seteando el tab o sección a
`visible="false"`, o usando el atributo `visible` en el `<cell>`.

#### Configurar una subgrid

```xml
<cell id="{SUBGRID-CELL-GUID}" showlabel="false" rowspan="10" colspan="1">
  <control id="subgrid_garantias" classid="{E7A81278-8635-4d9e-8D4D-59480B391C5B}"
           disabled="false">
    <parameters>
      <TargetEntityType>axx_garantia</TargetEntityType>
      <RelationshipName>axx_creditsolicitud_axx_garantia</RelationshipName>
      <EnableViewPicker>false</EnableViewPicker>
      <ViewId>{VIEW-GUID}</ViewId>
      <ViewIds>{VIEW-GUID}</ViewIds>
      <RecordsPerPage>10</RecordsPerPage>
    </parameters>
  </control>
</cell>
```

---

## Buenas prácticas de diseño de formularios

| Práctica                                           | Razón                                                   |
|----------------------------------------------------|---------------------------------------------------------|
| No más de 5 tabs en un Main form                   | UX — el usuario no scrollea más allá del tercer tab     |
| Secciones con máximo 10 campos                     | Legibilidad                                             |
| Header con 2-4 campos clave (estado, nombre, fecha)| Siempre visibles sin scrollear                          |
| Timeline en tab separado o al final del General    | No interrumpe el flujo de captura de datos              |
| Campos obligatorios en la sección superior         | El usuario los ve primero                               |
| Subgrids en tab propio cuando tienen muchas filas  | Evita que el form "pese" visualmente                    |
| `isdefault: true` solo en un form por tabla        | Más de uno como default genera comportamiento errático  |

---

## Restricciones

- **Nunca** modificar el FormXml de formularios de entidades OOB (Account, Contact, Lead, etc.)
  sin verificar que la solución contiene el form como componente no administrado.
- Si el form ya está en producción como parte de una solución administrada, crear un nuevo
  form en la solución no administrada en vez de modificar el existente.
- **Nunca** eliminar el Primary Name Control del header — rompe la navegación del registro.
- **Siempre** publicar el formulario después de cada modificación; sin `publish_form` los
  cambios no son visibles para los usuarios.
  Nunca tocar formularios de producción
