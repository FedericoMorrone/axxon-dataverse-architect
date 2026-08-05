---
name: entity-builder
description: >
  Crea, modifica y consulta tablas (entities), columnas (fields) y relaciones en Dataverse /
  Dynamics 365 CE vía Web API, siempre contra el environment DEV. Usar esta skill cuando el
  usuario pida crear o modificar una entidad, tabla, table, columna, campo, field, relación,
  relationship, option set, o lista desplegable; cuando pregunte qué columnas tiene una tabla,
  pida listar entidades del environment, renombrar un campo, hacer un campo obligatorio, o
  cambiar un tipo de dato. Respeta siempre el prefijo de publisher axx_ y agrega los
  componentes a la solución no administrada activa. Usar junto con la skill d365-architect.
---

# Entity Builder — D365 Architect Agent

Skill especializada en la construcción y modificación del modelo de datos en **Dataverse /
Dynamics 365 CE**. Todas las operaciones se realizan vía la **Dataverse Metadata API** a través
del MCP Server, y siempre dentro de una **solución no administrada** activa.

---

## MCP Tools disponibles

| Tool name              | Método HTTP | Endpoint Dataverse                                                    |
|------------------------|-------------|-----------------------------------------------------------------------|
| `create_table`         | POST        | `/api/data/v9.2/EntityDefinitions`                                    |
| `get_table`            | GET         | `/api/data/v9.2/EntityDefinitions(LogicalName='xxx')`                 |
| `list_tables`          | GET         | `/api/data/v9.2/EntityDefinitions?$select=LogicalName,DisplayName`    |
| `add_column`           | POST        | `/api/data/v9.2/EntityDefinitions(LogicalName='xxx')/Attributes`      |
| `get_column`           | GET         | `/api/data/v9.2/EntityDefinitions(LogicalName='xxx')/Attributes(LogicalName='yyy')` |
| `update_column`        | PUT         | `/api/data/v9.2/EntityDefinitions(LogicalName='xxx')/Attributes(LogicalName='yyy')` |
| `create_relationship`  | POST        | `/api/data/v9.2/RelationshipDefinitions`                              |
| `get_relationship`     | GET         | `/api/data/v9.2/RelationshipDefinitions(SchemaName='xxx')`            |
| `create_optionset`     | POST        | `/api/data/v9.2/GlobalOptionSetDefinitions`                           |
| `add_to_solution`      | POST        | `/api/data/v9.2/AddSolutionComponent`                                 |
| `publish_all`          | POST        | `/api/data/v9.2/PublishAllXml`                                        |

---

## Flujo de ejecución

```
Usuario describe la tabla/campo/relación
        ↓
Skill valida inputs (nombre, tipo, prefijo)
        ↓
Skill construye el payload JSON para la Metadata API
        ↓
Orchestrator muestra resumen → espera confirmación
        ↓
MCP Server ejecuta el request HTTP
        ↓
Skill parsea la respuesta → agrega componente a la solución
        ↓
output-formatter presenta resultado (ac-status-card)
```

---

## Creación de tablas (`create_table`)

### Inputs requeridos

| Input               | Descripción                                              | Ejemplo                    |
|---------------------|----------------------------------------------------------|----------------------------|
| `displayName`       | Nombre legible (singular)                                | `Solicitud de Crédito`     |
| `displayNamePlural` | Nombre legible (plural)                                  | `Solicitudes de Crédito`   |
| `logicalName`       | Nombre lógico con prefijo del publisher                  | `axx_creditsolicitud`    |
| `description`       | Descripción funcional (opcional pero recomendado)        | `Registro de solicitudes...`|
| `ownership`         | `UserOwned` o `OrganizationOwned`                        | `UserOwned`                |
| `hasActivities`     | Si habilita Activities (timeline)                        | `true`                     |
| `hasNotes`          | Si habilita Notes                                        | `true`                     |

### Reglas de nomenclatura

- **Siempre** usar el prefijo del publisher seguido de `_` y el nombre en minúsculas sin espacios.
- Máximo 50 caracteres en el logical name.
- No usar palabras reservadas de Dataverse: `account`, `contact`, `lead`, `opportunity`, etc.
  (estas ya existen como OOB entities — si el usuario quiere trabajar sobre ellas, usás
  `get_table` + `add_column` en vez de `create_table`).



> **Nota de idioma:** Los `LocalizedLabels` usan `LanguageCode: 3082` para español (España).
> Para español (Argentina/LATAM) en environments configurados en inglés, podés agregar
> también `LanguageCode: 1033` con el label en inglés para evitar warnings de validación.

---

## Tipos de columnas (`add_column`)

### Mapeo tipo-humano → `@odata.type`

| Tipo pedido por el usuario         | `@odata.type` Metadata API                                        | Notas                                      |
|------------------------------------|-------------------------------------------------------------------|--------------------------------------------|
| Texto corto                        | `Microsoft.Dynamics.CRM.StringAttributeMetadata`                 | `MaxLength`: 1-4000                        |
| Texto largo / memo                 | `Microsoft.Dynamics.CRM.MemoAttributeMetadata`                   | `MaxLength`: hasta 1.048.576               |
| Número entero                      | `Microsoft.Dynamics.CRM.IntegerAttributeMetadata`                | `MinValue`, `MaxValue`                     |
| Número decimal                     | `Microsoft.Dynamics.CRM.DecimalAttributeMetadata`                | `Precision`: 0-10                          |
| Moneda / importe                   | `Microsoft.Dynamics.CRM.MoneyAttributeMetadata`                  | Requiere currency column automática        |
| Fecha y hora                       | `Microsoft.Dynamics.CRM.DateTimeAttributeMetadata`               | `Format`: `DateOnly` o `DateAndTime`       |
| Sí/No (booleano)                   | `Microsoft.Dynamics.CRM.BooleanAttributeMetadata`                | `TrueOption`, `FalseOption`                |
| Lista de opciones (choice)         | `Microsoft.Dynamics.CRM.PicklistAttributeMetadata`               | Referencia a global o local option set     |
| Lookup / relación N:1              | `Microsoft.Dynamics.CRM.LookupAttributeMetadata`                 | Se crea vía `create_relationship`          |
| Autonumber                         | `Microsoft.Dynamics.CRM.StringAttributeMetadata` + `AutoNumberFormat` | Ej: `CRED-{SEQNUM:5}`               |
| Imagen                             | `Microsoft.Dynamics.CRM.ImageAttributeMetadata`                  | Solo una por tabla                         |
| File                               | `Microsoft.Dynamics.CRM.FileAttributeMetadata`                   | `MaxSizeInKB` requerido                    |

### Niveles de requerimiento

| Valor UI               | `RequiredLevel.Value`     |
|------------------------|---------------------------|
| Opcional               | `None`                    |
| Recomendado            | `Recommended`             |
| Obligatorio (negocio)  | `ApplicationRequired`     |
| Obligatorio (sistema)  | `SystemRequired`          |

> `SystemRequired` solo puede usarse en columnas del sistema. Para campos de negocio,
> siempre usar `ApplicationRequired`.



> **Regla de values:** Los option values de publisher deben estar en el rango reservado
> del publisher. El rango estándar de Dataverse para custom options es `100000000`–`2147483647`.
> Nunca usar valores menores a `100000000` para opciones custom.

---

## Relaciones (`create_relationship`)

### Tipos soportados

| Tipo            | Cuándo usar                                                       | Tool                   |
|-----------------|-------------------------------------------------------------------|------------------------|
| N:1 (Many-to-One) | Una solicitud tiene un cliente (Contact)                        | `create_relationship`  |
| 1:N (One-to-Many) | Un cliente tiene muchas solicitudes                            | `create_relationship`  |
| N:N (Many-to-Many) | Una solicitud puede tener varios garantes (otro approach: tabla intermedia) | `create_relationship` |



### Cascade configuration — guía rápida

| Scenario                                         | Delete recomendado  |
|--------------------------------------------------|---------------------|
| Registro hijo sin sentido sin el padre           | `Cascade`           |
| Registro hijo puede existir sin padre            | `RemoveLink`        |
| Registro hijo nunca debe borrarse si hay padre   | `Restrict`          |

---

## Global Option Sets

Usá global option sets cuando el mismo conjunto de opciones se reutiliza en varias tablas
(ej: `axx_estadogeneral` con Activo/Inactivo/Suspendido).

### Crear global option set

---

## Agregar componente a la solución

Después de crear o modificar cualquier componente, **siempre** llamás a `add_to_solution`:

```json
{
  "ComponentId": "<GUID del componente creado>",
  "ComponentType": 1,
  "SolutionUniqueName": "AxxonClienteXCreditOnboarding",
  "AddRequiredComponents": false
}
```

### Component Type codes frecuentes

| Componente            | `ComponentType` |
|-----------------------|-----------------|
| Entity / Table        | 1               |
| Attribute / Column    | 2               |
| Relationship          | 10              |
| Global Option Set     | 9               |
| Form                  | 24              |
| View                  | 26              |
| Business Rule         | 29              |
| Security Role         | 14              |

> ⚠️ **Sin verificar contra fuente oficial de Microsoft.** Los códigos de `Entity`, `Attribute`,
> `Relationship`, `Form`, `View` y `Global Option Set` de esta tabla están confirmados. El de
> `Security Role` (`14`) proviene de la v1.0.0 de este documento y no fue re-verificado — antes
> de depender de él en un ambiente real, confirmar contra `$metadata` o documentación oficial
> vigente. Ver también la advertencia equivalente en `04-solution-packager.md` y
> `09-duplicate-detection.md`.

---

## Restricciones

- **Nunca** crear tablas sin prefijo de publisher — Dataverse rechaza el request.
- **Nunca** llamar `publish_all` sin confirmación explícita del usuario.
- **Nunca** modificar columnas del sistema (las que no tienen prefijo de publisher).
- Si el usuario pide eliminar una columna, advertir que es irreversible y que puede haber
  datos asociados. Requerir doble confirmación.
- No se crean componentes en Producción