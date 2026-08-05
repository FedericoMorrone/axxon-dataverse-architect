---
name: duplicate-detection
description: >
  Configura los dos mecanismos de Dataverse / Dynamics 365 CE para evitar registros
  duplicados: Alternate Keys (constraint duro de unicidad a nivel de índice) y Duplicate
  Detection Rules (advertencia difusa al guardar), vía Web API contra DEV. Especialmente
  relevante en banca/FSI para evitar clientes o solicitudes duplicadas. Usar esta skill cuando
  el usuario pida evitar duplicados, crear una clave alternativa o alternate key, una regla de
  detección de duplicados, no permitir clientes repetidos, validar que no exista ya un CUIT u
  otro identificador, detectar registros duplicados, definir unicidad de un campo, un
  constraint único, correr un job de detección de duplicados, o buscar duplicados existentes.
  Usar junto con la skill dataverse-architect.
---

# Duplicate Detection — Axxon Dataverse Architect

Skill especializada en evitar y detectar registros duplicados en Dataverse / D365 CE. Cubre
los **dos mecanismos** disponibles, que resuelven problemas distintos y no son intercambiables:

| Mecanismo | Qué hace | Cuándo actúa | Se puede evitar |
|---|---|---|---|
| **Alternate Key** | Constraint duro a nivel de índice — el registro **no se puede guardar** si viola la unicidad | Síncrono, en cada create/update, incluida integración vía API | No — es un rechazo duro |
| **Duplicate Detection Rule** | Advertencia difusa (fuzzy match) — el usuario ve un diálogo con los posibles duplicados y decide si continuar | Síncrono en el formulario (si está habilitado), o asíncrono vía Job para escaneo retroactivo | Sí — el usuario puede confirmar igual |

**Criterio de uso:** si el campo tiene una identidad inequívoca (CUIT, número de expediente,
external ID de un sistema origen) → **Alternate Key**. Si la coincidencia es aproximada
(nombre + apellido + fecha de nacimiento, razón social similar) → **Duplicate Detection Rule**.
En banca, frecuentemente se necesitan ambos sobre la misma tabla.

---

## MCP Tools disponibles

### Alternate Keys (Metadata API)

| Tool name                  | Método HTTP | Endpoint Dataverse                                                        |
|-------------------------------|-------------|---------------------------------------------------------------------------|
| `list_alternate_keys`      | GET         | `/api/data/v9.2/EntityDefinitions(LogicalName='xxx')/Keys`                |
| `create_alternate_key`     | POST        | `/api/data/v9.2/EntityDefinitions(LogicalName='xxx')/Keys`                |
| `get_key_index_status`     | GET         | `/api/data/v9.2/EntityKeyDefinitions(<keyid>)?$select=EntityKeyIndexStatus` |

### Duplicate Detection Rules

| Tool name                        | Método HTTP | Endpoint Dataverse                                                    |
|--------------------------------------|-------------|---------------------------------------------------------------------------|
| `list_duplicate_rules`           | GET         | `/api/data/v9.2/duplicaterules?$filter=baseentityname eq 'xxx'`        |
| `create_duplicate_rule`          | POST        | `/api/data/v9.2/duplicaterules`                                          |
| `add_duplicate_rule_condition`   | POST        | `/api/data/v9.2/duplicaterulesconditions`                                |
| `publish_duplicate_rule`         | POST        | `/api/data/v9.2/PublishDuplicateRule`                                    |
| `unpublish_duplicate_rule`       | POST        | `/api/data/v9.2/UnpublishDuplicateRule`                                  |
| `run_duplicate_detection_job`    | POST        | `/api/data/v9.2/BulkDetectDuplicates`                                    |
| `get_duplicate_job_status`       | GET         | `/api/data/v9.2/asyncoperations(<asyncoperationid>)?$select=statuscode,message` |
| `get_duplicate_job_results`      | GET         | `/api/data/v9.2/duplicaterecords?$filter=duplicatedetectionjobid eq '<id>'` |

---

## 1. Alternate Keys

### Cuándo usar

- Campos que representan un identificador único de negocio: CUIT, CBU, número de expediente,
  ID del core bancario (ej: ID en Bantotal), external ID de un sistema origen en integraciones.
- Cuando la integridad debe garantizarse **incluso en cargas por API/integración**, no solo
  desde el formulario — un Alternate Key rechaza el insert sin importar el canal de entrada.
- Como target de `upsert` en integraciones (permite hacer upsert por el alternate key en vez
  de por GUID, muy usado en Power Automate / integraciones del Motor IA Documental).

### Crear un Alternate Key (`create_alternate_key`)

```json
{
  "SchemaName": "axx_CUITKey",
  "DisplayName": "Clave CUIT",
  "KeyAttributes": ["axx_cuit"],
  "EntityKeyIndexStatus": "Pending"
}
```

Para claves compuestas (ej: número de solicitud + sucursal, cuando el número por sí solo no
es único entre sucursales):

```json
{
  "SchemaName": "axx_NumeroSucursalKey",
  "DisplayName": "Clave Número + Sucursal",
  "KeyAttributes": ["axx_numerosolicitud", "axx_sucursal"]
}
```

### Restricciones técnicas de Alternate Keys

| Restricción | Detalle |
|---|---|
| Máximo de atributos por clave | 5 |
| Máximo de claves por tabla | 5 (activas) |
| Tipos de atributo soportados | Text, Whole Number, Decimal, DateTime, Picklist, Lookup — **no** soporta Memo, Money con precisión variable, ni Multi-select Choice |
| Indexado | Asíncrono — el status pasa por `Pending` → `InProgress` → `Active`. **No usar la clave para upsert hasta que el status sea `Active`** |
| Valores nulos | Un valor null en el/los atributo(s) de la clave no viola la unicidad (comportamiento estándar SQL) — si el campo puede quedar vacío, la clave no protege esos registros |

### Verificar el estado del índice antes de usar la clave

```
GET /api/data/v9.2/EntityKeyDefinitions(<keyid>)?$select=EntityKeyIndexStatus
```

> **Importante:** en tablas con volumen alto de datos ya cargados (ej: migración de una base
> de clientes existente), la creación del índice puede tardar minutos u horas. La skill
> **siempre** advierte esto y ofrece hacer polling del estado en vez de asumir que quedó listo.

---

## 2. Duplicate Detection Rules

### Estructura de una regla

Una regla compara una **entidad base** (la que se está creando/editando) contra una
**entidad de coincidencia** (puede ser la misma tabla u otra) usando una o más condiciones.

```json
{
  "name": "DR - Posible cliente duplicado por CUIT y Razón Social",
  "description": "Detecta contactos/cuentas con CUIT similar o razón social muy parecida",
  "baseentityname": "axx_creditsolicitud",
  "matchingentityname": "axx_creditsolicitud",
  "excludeinactiverecords": true
}
```

### Condiciones de coincidencia (`add_duplicate_rule_condition`)

| Operador               | `operatorcode` | Uso típico                                                    |
|---------------------------|------------------|--------------------------------------------------------------------|
| Exact Match             | `0`              | CUIT, CBU, DNI — campos donde solo una coincidencia exacta importa |
| Same First Characters   | `1`              | Razón social — detecta "Constructora del Plata SA" vs "Constructora del Plata SRL" |
| Same Last Characters    | `2`              | Sufijos/códigos compartidos                                        |
| Exact Match (case insensitive por defecto en texto) | `0` | Nombres de personas                                    |

```json
{
  "duplicateruleid@odata.bind": "/duplicaterules(<ruleid>)",
  "baseattributename": "axx_cuit",
  "matchingattributename": "axx_cuit",
  "operatorcode": 0
}
```

Para campos numéricos/monetarios se puede definir una tolerancia porcentual en vez de
coincidencia exacta (ej: "monto solicitado dentro del 5% de otra solicitud del mismo cliente
en los últimos 30 días" — un patrón útil para detectar múltiples solicitudes fraccionadas).

### Publicar la regla (obligatorio para que tenga efecto)

Las reglas se crean en estado **Draft** (`statecode: 0`). Igual que las Business Rules,
**no tienen efecto hasta publicarse**:

```json
// POST /api/data/v9.2/PublishDuplicateRule
{ "DuplicateRuleId": "<ruleid>" }
```

### Habilitar la detección en el formulario

La detección en tiempo real al guardar requiere además que **Duplicate Detection Settings**
esté habilitado a nivel de organización (`Settings → Data Management → Duplicate Detection
Settings`) con al menos "Duplicate Detection during record creation and update" activo. Si el
usuario reporta que las reglas no disparan, este es el primer punto a verificar — no es un
problema de la regla en sí.

### Job de detección retroactiva (`run_duplicate_detection_job`)

Para escanear datos **ya existentes** (ej: después de una migración, o al activar una regla
nueva sobre una tabla con historial):

```json
{
  "JobName": "Detección retroactiva - Solicitudes de Crédito Q3 2026",
  "Query": "<fetch><entity name='axx_creditsolicitud'/></fetch>",
  "DuplicateRuleId": "<ruleid>",
  "RecurrencePattern": null,
  "SendEmailNotification": false
}
```

Es una operación **asíncrona** — la skill dispara el job y hace polling de
`get_duplicate_job_status`, y cuando termina, resume los resultados vía
`get_duplicate_job_results` en una `ac-table-card` con los pares de registros detectados como
posibles duplicados, sin resolverlos automáticamente (la resolución/merge de duplicados existentes
es una decisión de negocio que requiere revisión humana, no se automatiza desde esta skill).

---

## Combinando ambos mecanismos — patrón recomendado para FSI

Para una tabla como `axx_creditsolicitud` en un proceso de onboarding:

1. **Alternate Key** sobre `axx_numerosolicitud` (o el ID del sistema origen si la solicitud
   se crea vía integración) — garantiza que la misma solicitud no se cree dos veces por un
   reintento de integración fallido.
2. **Duplicate Detection Rule** sobre `Contact`/`Account` sobre CUIT (Exact Match) y sobre
   razón social (Same First Characters) — detecta que el *mismo cliente* está iniciando una
   *nueva* solicitud, sin bloquear el flujo (puede ser legítimo: un cliente con más de una
   solicitud activa), simplemente advirtiendo.

Esta combinación es la que se documenta por defecto cuando el usuario pide "evitar
duplicados" sin especificar cuál de los dos escenarios tiene en mente — la skill pregunta cuál
aplica si no es evidente por el contexto.

---

## Advertencia — agregar estos componentes a la solución

A diferencia de tablas, forms o vistas, **no incluir un `ComponentType` hardcodeado** para
Alternate Keys ni Duplicate Detection Rules al armar el `add_component` en `solution-packager`.
Se encontró un caso real reportado donde `ComponentType=44` (usado comúnmente en ejemplos
online para `DuplicateRule`) fue rechazado por Dataverse. Antes de la implementación,
verificar el código correcto contra `$metadata` del environment target — no asumir un valor
de una fuente no oficial. Ver advertencia equivalente en `04-solution-packager.md`.

---

## Restricciones

- **Nunca** crear un Alternate Key sobre un campo que ya tiene valores duplicados en datos
  existentes — la creación del índice falla. Ejecutar primero una consulta de verificación
  (`get_view`/FetchXML con `<attribute>` agrupado, vía `view-designer`) antes de proponer la
  clave.
- **Nunca** asumir que una Duplicate Detection Rule bloquea el guardado — es una advertencia,
  no un constraint. Si el requerimiento de negocio es "no debe poder guardarse", usar
  Alternate Key o Business Rule con `ShowError`, no esta skill.
- Los Jobs de detección retroactiva sobre tablas grandes pueden tardar — siempre informar al
  usuario que es asíncrono antes de disparar el job, nunca prometer un resultado inmediato.
- Máximo 5 Alternate Keys activas por tabla — si se necesitan más combinaciones de unicidad,
  evaluar si corresponde modelo de datos distinto (tabla de identificadores externos aparte).
