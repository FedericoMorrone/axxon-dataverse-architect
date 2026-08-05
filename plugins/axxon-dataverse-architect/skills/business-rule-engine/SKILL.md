---
name: business-rule-engine
description: >
  Crea y documenta Business Rules en Dataverse / Dynamics 365 CE vía Web API contra DEV,
  evaluando cuándo conviene Business Rule vs Power Fx vs JavaScript Web Resource. Usar esta
  skill cuando el usuario pida crear una regla de negocio o business rule, describa lógica del
  tipo "si el campo X entonces el campo Y", pida ocultar un campo cuando pase algo, hacer un
  campo obligatorio condicionalmente, validar que un monto sea mayor a un valor, bloquear un
  campo según el estado, mostrar un mensaje de error, o describa cualquier lógica condicional
  de formulario. Usar junto con la skill dataverse-architect.
---

# Business Rule Engine — Axxon Dataverse Architect

Skill especializada en la creación de **Business Rules** en Dataverse / Dynamics 365 CE.
Las Business Rules son configuración declarativa que ejecuta lógica en el cliente (formulario)
y/o en el servidor (entity scope) sin código.

---

## Cuándo usar Business Rule vs alternativas

| Lógica requerida                                          | Recomendación                  | Razón                                           |
|-----------------------------------------------------------|--------------------------------|-------------------------------------------------|
| Mostrar/ocultar campo según valor de otro campo           | Business Rule                  | OOB, sin código, se aplica en todos los forms  |
| Hacer campo obligatorio condicionalmente                  | Business Rule                  | OOB, server + client                           |
| Set value de un campo según otro                         | Business Rule                  | OOB                                             |
| Mostrar mensaje de error / warning al usuario            | Business Rule                  | OOB, `ShowError` action                         |
| Lógica con múltiples condiciones anidadas (AND/OR/NOT)   | Business Rule (hasta 3 niveles)| Si es más complejo → JavaScript                |
| Cálculos aritméticos entre campos                        | Calculated Column o Power Fx   | Business Rule no soporta aritmética             |
| Lógica cross-entity (lookup a otro registro)             | Power Automate / Plugin        | Business Rules no navegan lookups              |
| Lógica que depende del rol del usuario                   | JavaScript Web Resource        | Business Rules no tienen acceso al rol         |
| Validación en bulk / import                              | Plugin (server-side)           | Business Rules client-only no aplican en bulk  |

Cuando la lógica excede lo que puede hacer una Business Rule, esta skill genera la
especificación en JavaScript o Power Automate según corresponda.

---

## MCP Tools disponibles

| Tool name                | Método HTTP | Endpoint Dataverse                                                            |
|--------------------------|-------------|-------------------------------------------------------------------------------|
| `list_business_rules`    | GET         | `/api/data/v9.2/workflows?$filter=category eq 2 and primaryentity eq 'xxx'`  |
| `get_business_rule`      | GET         | `/api/data/v9.2/workflows(<workflowid>)`                                      |
| `create_business_rule`   | POST        | `/api/data/v9.2/workflows`                                                    |
| `update_business_rule`   | PATCH       | `/api/data/v9.2/workflows(<workflowid>)`                                      |
| `activate_rule`          | POST        | `/api/data/v9.2/SetStateRequest` (statecode: 1, statuscode: 2)                |
| `deactivate_rule`        | POST        | `/api/data/v9.2/SetStateRequest` (statecode: 0, statuscode: 1)                |
| `add_to_solution`        | POST        | `/api/data/v9.2/AddSolutionComponent` (ComponentType: 29)                     |

---

## Inputs requeridos del usuario

| Input              | Descripción                                              | Ejemplo                                          |
|--------------------|----------------------------------------------------------|--------------------------------------------------|
| `tableName`        | Logical name de la tabla donde aplica la regla           | `axx_creditsolicitud`                          |
| `ruleName`         | Nombre descriptivo de la regla                           | `Monto requerido si estado es Enviada`           |
| `scope`            | `Entity` (server+client) o `Form` (solo client)          | `Entity`                                         |
| `conditions`       | Descripción de las condiciones en lenguaje natural       | "si estado = Enviada Y monto = 0"                |
| `actions`          | Descripción de las acciones a ejecutar                   | "hacer campo monto obligatorio, mostrar error"   |

---

## Estructura interna de una Business Rule

En Dataverse, las Business Rules se almacenan como **Workflows** (`category: 2`) con un
campo `xaml` que contiene la lógica en formato XAML/Activities. Sin embargo, la API acepta
una representación JSON simplificada vía el campo `clientdata`.

### Schema del `clientdata` (JSON)

```json
{
  "Version": "1.0",
  "Scope": "Entity",
  "Conditions": {
    "LogicalOperator": "And",
    "Conditions": [
      {
        "FieldName": "axx_estadosolicitud",
        "Operator": "Equal",
        "ValueType": "OptionSetValue",
        "Value": "100000001"
      }
    ]
  },
  "TrueActions": [
    {
      "Type": "SetRequired",
      "FieldName": "axx_montosolicitado",
      "Value": "required"
    },
    {
      "Type": "ShowError",
      "ErrorMessage": "El monto solicitado es obligatorio cuando la solicitud está en estado Enviada."
    }
  ],
  "FalseActions": [
    {
      "Type": "SetRequired",
      "FieldName": "axx_montosolicitado",
      "Value": "none"
    }
  ]
}
```

---

## Operadores de condición disponibles

| Operador               | `Operator` value          | Tipos de campo compatibles            |
|------------------------|---------------------------|---------------------------------------|
| Igual a                | `Equal`                   | Todos                                 |
| No es igual a          | `NotEqual`                | Todos                                 |
| Contiene datos         | `ContainsData`            | Todos (verifica not null)             |
| No contiene datos      | `DoesNotContainData`      | Todos (verifica null)                 |
| Mayor que              | `GreaterThan`             | Número, Dinero, Fecha                 |
| Menor que              | `LessThan`                | Número, Dinero, Fecha                 |
| Mayor o igual          | `GreaterThanOrEqual`      | Número, Dinero, Fecha                 |
| Menor o igual          | `LessThanOrEqual`         | Número, Dinero, Fecha                 |
| Comienza con           | `BeginsWith`              | Texto                                 |
| Termina con            | `EndsWith`                | Texto                                 |
| Contiene               | `Contains`                | Texto                                 |

---

## Tipos de acciones disponibles

| Acción                     | `Type` value       | Descripción                                                        |
|----------------------------|--------------------|--------------------------------------------------------------------|
| Hacer campo obligatorio     | `SetRequired`      | `"Value": "required"` o `"none"`                                   |
| Hacer campo recomendado     | `SetRequired`      | `"Value": "recommended"`                                           |
| Mostrar / ocultar campo     | `SetVisibility`    | `"Value": "show"` o `"hide"`                                       |
| Bloquear / desbloquear      | `SetDisabled`      | `"Value": "lock"` o `"unlock"`                                     |
| Asignar valor               | `SetFieldValue`    | `"Value"` puede ser literal, otro campo (`FieldName`), o formula   |
| Mostrar mensaje de error    | `ShowError`        | `"ErrorMessage"`: texto del mensaje                                |
| Mostrar notificación        | `ShowNotification` | `"NotificationMessage"` + `"NotificationType"`: info/warning/error |
| Asignar valor de default    | `SetDefaultValue`  | Solo se ejecuta si el campo está vacío                             |

---

## Ejemplos de reglas frecuentes en FSI / Banca

### Ejemplo 1 — Campo obligatorio condicional

**Requerimiento:** "El campo CUIT es obligatorio si el tipo de persona es Jurídica."

```json
{
  "Version": "1.0",
  "Scope": "Entity",
  "Conditions": {
    "LogicalOperator": "And",
    "Conditions": [
      {
        "FieldName": "axx_tipopersona",
        "Operator": "Equal",
        "ValueType": "OptionSetValue",
        "Value": "100000001"
      }
    ]
  },
  "TrueActions": [
    { "Type": "SetRequired", "FieldName": "axx_cuit", "Value": "required" },
    { "Type": "SetVisibility", "FieldName": "axx_cuit", "Value": "show" }
  ],
  "FalseActions": [
    { "Type": "SetRequired", "FieldName": "axx_cuit", "Value": "none" }
  ]
}
```

### Ejemplo 2 — Bloqueo por estado

**Requerimiento:** "Bloquear todos los campos de monto cuando el estado es Aprobada o Rechazada."

```json
{
  "Version": "1.0",
  "Scope": "Entity",
  "Conditions": {
    "LogicalOperator": "Or",
    "Conditions": [
      { "FieldName": "axx_estadosolicitud", "Operator": "Equal", "ValueType": "OptionSetValue", "Value": "100000002" },
      { "FieldName": "axx_estadosolicitud", "Operator": "Equal", "ValueType": "OptionSetValue", "Value": "100000003" }
    ]
  },
  "TrueActions": [
    { "Type": "SetDisabled", "FieldName": "axx_montosolicitado", "Value": "lock" },
    { "Type": "SetDisabled", "FieldName": "axx_plazo", "Value": "lock" },
    { "Type": "SetDisabled", "FieldName": "axx_tasainteres", "Value": "lock" }
  ],
  "FalseActions": [
    { "Type": "SetDisabled", "FieldName": "axx_montosolicitado", "Value": "unlock" },
    { "Type": "SetDisabled", "FieldName": "axx_plazo", "Value": "unlock" },
    { "Type": "SetDisabled", "FieldName": "axx_tasainteres", "Value": "unlock" }
  ]
}
```

### Ejemplo 3 — Visibilidad de sección según segmento

**Requerimiento:** "Mostrar la sección de Garantías solo si el monto es mayor a $1.000.000."

> Business Rules no pueden mostrar/ocultar secciones directamente — solo campos individuales.
> Para secciones, esta skill genera un **JavaScript Web Resource** equivalente.

```javascript
// JavaScript alternativo — Web Resource: axx_CreditSolicitud_FormLogic.js
function onMontoChanged(executionContext) {
  var formContext = executionContext.getFormContext();
  var monto = formContext.getAttribute("axx_montosolicitado").getValue();
  var seccion = formContext.ui.tabs.get("tab_general").sections.get("sec_garantias");
  if (monto !== null && monto > 1000000) {
    seccion.setVisible(true);
  } else {
    seccion.setVisible(false);
  }
}
```

Instrucciones de registro del Web Resource en el form:

```json
{
  "eventName": "onchange",
  "attributeName": "axx_montosolicitado",
  "functionName": "onMontoChanged",
  "webResourceName": "axx_CreditSolicitud_FormLogic",
  "passExecutionContext": true
}
```

---

## Payload completo para `create_business_rule`

```json
{
  "name": "BR - Monto requerido si estado Enviada",
  "description": "Hace obligatorio el campo Monto Solicitado cuando el estado de la solicitud es Enviada",
  "category": 2,
  "primaryentity": "axx_creditsolicitud",
  "scope": 1,
  "statecode": 0,
  "statuscode": 1,
  "clientdata": "{... JSON de la regla escapado como string ...}"
}
```

> **`scope`:** `1` = Entity (server + client). `2` = Form específico (solo client).
> Usar Entity scope siempre que sea posible para que la regla aplique también en bulk/API.

---

## Activación de la regla

Las reglas se crean en estado **Draft** (`statecode: 0`). Para activarlas:

```json
{
  "EntityMoniker": { "@odata.type": "Microsoft.Dynamics.CRM.workflow", "workflowid": "<GUID>" },
  "State": { "Value": 1 },
  "Status": { "Value": 2 }
}
```

**Siempre activar la regla después de crearla** — una regla en Draft no tiene efecto.

---

## Documentación de reglas generada

Después de crear cada Business Rule, la skill genera una tabla de documentación para incluir
en el Solution Design Document:

| # | Nombre de la Regla                        | Tabla                    | Scope  | Condición                         | Acción (True)                          | Acción (False)               |
|---|-------------------------------------------|--------------------------|--------|-----------------------------------|----------------------------------------|------------------------------|
| 1 | BR - Monto requerido si estado Enviada    | axx_creditsolicitud    | Entity | Estado = Enviada                  | Monto: Obligatorio + ShowError         | Monto: Opcional              |
| 2 | BR - Bloqueo campos por estado            | axx_creditsolicitud    | Entity | Estado = Aprobada OR Rechazada    | Bloquear: Monto, Plazo, Tasa           | Desbloquear: Monto, Plazo, Tasa |

---

## Restricciones

- **Máximo 10 condiciones** por Business Rule — si se necesitan más, dividir en múltiples reglas
  o usar JavaScript.
- **No usar Business Rules para lógica de precios o cálculos** — usar Calculated Columns,
  Rollup Columns, o Power Automate.
- **Business Rules no aplican en importaciones de datos** a menos que sean Entity scope —
  aclarárselo siempre al usuario.
- **No crear Business Rules en formularios administrados** sin antes verificar que la capa
  de customización está en la solución correcta.
