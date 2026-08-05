---
name: security-architect
description: >
  Diseña el modelo de seguridad en Dataverse / Dynamics 365 CE — Security Roles, Business
  Units, Teams, Column/Field Security, Hierarchy Security — generando XML de solución y
  commiteando a git en vez de ejecutar en vivo, porque la seguridad es sensible y se beneficia
  del PR review. Usar esta skill cuando el usuario pida crear un rol de seguridad o security
  role, configurar permisos, asignar permisos sobre una entidad, crear una business unit o
  team, configurar field level security o column security, diseñar el modelo de seguridad,
  preguntar quién puede ver qué, configurar sharing, hierarchy security, armar la matriz de
  seguridad, o preguntar qué permisos tiene un rol. Usar junto con la skill d365-architect.
---

# Security Architect — D365 Architect Agent

Skill especializada en el diseño e implementación del **modelo de seguridad** en Dataverse /
Dynamics 365 CE. La seguridad en D365 es multicapa — esta skill cubre las cuatro capas
principales: Environment, Business Unit, Team, y Record/Field level.

**Cambio de canal en v2.0.0:** a diferencia de `entity-builder`/`form-designer`, esta skill
**no** ejecuta cambios en vivo contra DEV. Genera el XML de solución correspondiente a cada
componente de seguridad y lo commitea directo a un branch de Azure DevOps Repos. La razón:
los roles de seguridad normalmente se definen una vez con una matriz clara y cambian poco —
el review por PR importa más acá que la iteración conversacional rápida.

---

## Las 4 capas de seguridad en Dataverse

```
┌─────────────────────────────────────────────────────────────────┐
│  Environment  (quién puede entrar al environment)               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Security Role  (qué puede hacer — permisos por entidad)  │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Business Unit / Team  (scope de los datos visibles) │  │  │
│  │  │  ┌───────────────────────────────────────────────┐  │  │  │
│  │  │  │  Column Security Profile  (campos sensibles)  │  │  │  │
│  │  │  └───────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## MCP Tools disponibles

### Canal lectura — Web API (para consultar el estado actual, siempre permitido)

| Tool name                   | Método HTTP | Endpoint Dataverse                                                        |
|--------------------------------|-------------|---------------------------------------------------------------------------|
| `list_roles`                | GET         | `/api/data/v9.2/roles?$select=name,roleid,businessunitid`                 |
| `get_role`                  | GET         | `/api/data/v9.2/roles(<roleid>)?$expand=roleprivileges`                   |
| `list_business_units`       | GET         | `/api/data/v9.2/businessunits?$select=name,parentbusinessunitid`          |
| `list_teams`                | GET         | `/api/data/v9.2/teams?$select=name,teamtype,businessunitid`               |
| `list_column_security`      | GET         | `/api/data/v9.2/fieldsecurityprofiles`                                    |
| `get_entity_privileges`     | GET         | `/api/data/v9.2/EntityDefinitions(LogicalName='xxx')/Privileges` — para cachear GUIDs de privilegios por entidad en `.d365-session.md` del Cowork Project |

### Canal escritura — Git (genera XML, commitea, nunca ejecuta directo)

| Tool name                       | Acción                                                                 |
|-------------------------------------|------------------------------------------------------------------------------|
| `generate_role_xml`             | Construye el XML de Security Role (`<Role>` con `<RolePrivileges>`) según la matriz provista |
| `generate_businessunit_xml`     | Construye el XML de Business Unit para la solución                     |
| `generate_team_xml`             | Construye el XML de Team (Owner/Access/AAD)                            |
| `generate_fieldsecurity_xml`    | Construye el XML de Field Security Profile + Field Permissions          |
| `commit_security_artifacts`     | Commitea todos los XML generados en la sesión al branch activo          |

> Ya no existen `create_role`, `add_privilege_to_role`, `create_business_unit`, `create_team`,
> `create_column_security`, `set_column_permission`, `enable_field_security` como tools que
> ejecutan directo — quedan reemplazados por la generación de XML + commit.

---

## Security Roles

### Estructura de privilegios

Cada Security Role define permisos por entidad con dos dimensiones:
- **Tipo de privilegio:** Create, Read, Write, Delete, Append, AppendTo, Assign, Share
- **Profundidad (depth):** None, User, Business Unit, Parent:Child Business Units, Organization

| Depth value | Constante              | Significado                                          |
|---------------|---------------------------|------------------------------------------------------------|
| `0`         | `None`                 | Sin acceso                                           |
| `1`         | `Basic` (User)         | Solo registros propios                               |
| `2`         | `Local` (BU)           | Registros de su BU                                   |
| `3`         | `Deep` (Parent:Child)  | Registros de su BU y BUs hijas                       |
| `4`         | `Global` (Organization)| Todos los registros                                  |

### Obtener privilege GUIDs (canal lectura, se cachea en sesión)

Los privilege IDs son GUIDs fijos por entidad. Para no tener que resolverlos en cada
interacción, `get_entity_privileges` los consulta una vez por entidad y el resultado se
persiste en `.d365-session.md` del Cowork Project para el resto de la sesión:

```
GET /api/data/v9.2/EntityDefinitions(LogicalName='axx_creditsolicitud')/Privileges
```

### Generar el XML de un Security Role (`generate_role_xml`)

En vez de POST directo, la skill genera el fragmento XML que va dentro de
`Other/Roles/<rolename>.xml` en la estructura unpacked de la solución:

```xml
<Role roleid="{ROLE-GUID}" name="Axxon - Analista de Crédito">
  <businessunitid>{ROOT-BU-GUID}</businessunitid>
  <RolePrivileges>
    <RolePrivilege privilegeid="{PRVCREATE-GUID}" privilegedepthmask="1" />
    <RolePrivilege privilegeid="{PRVREAD-GUID}" privilegedepthmask="1" />
    <RolePrivilege privilegeid="{PRVWRITE-GUID}" privilegedepthmask="1" />
    <RolePrivilege privilegeid="{PRVDELETE-GUID}" privilegedepthmask="0" />
  </RolePrivileges>
</Role>
```

El GUID de cada privilegio se resuelve vía `get_entity_privileges` (canal lectura) antes de
generar el XML — nunca se inventan ni se hardcodean valores placeholder en el commit final.

---

## Plantillas de roles frecuentes en FSI / Banca

### Rol: Analista de Crédito

| Entidad                    | Create   | Read     | Write    | Delete   | Append   | AppendTo |
|--------------------------------|------------|------------|------------|------------|------------|------------|
| axx_creditsolicitud      | Basic    | Basic    | Basic    | None     | Basic    | Basic    |
| axx_garantia             | Basic    | Basic    | Basic    | None     | Basic    | Basic    |
| Contact                    | None     | Local    | None     | None     | None     | Basic    |
| Account                    | None     | Local    | None     | None     | None     | Basic    |
| axx_dictamen             | None     | Basic    | None     | None     | None     | Basic    |
| Annotation (notas)         | Basic    | Basic    | Basic    | Basic    | Basic    | Basic    |

### Rol: Supervisor de Crédito

| Entidad                    | Create   | Read     | Write    | Delete   | Assign   |
|--------------------------------|------------|------------|------------|------------|------------|
| axx_creditsolicitud      | Local    | Local    | Local    | None     | Local    |
| axx_garantia             | Local    | Local    | Local    | None     | None     |
| Contact                    | Local    | Local    | Local    | None     | None     |
| axx_dictamen             | Local    | Local    | Local    | None     | None     |

### Rol: Gerente de Crédito

| Entidad                    | Create   | Read     | Write    | Delete   | Assign   |
|--------------------------------|------------|------------|------------|------------|------------|
| axx_creditsolicitud      | Deep     | Deep     | Deep     | Local    | Deep     |
| axx_garantia             | Deep     | Deep     | Deep     | Local    | Deep     |
| Contact                    | Deep     | Global   | Local    | None     | Local    |
| axx_dictamen             | Deep     | Deep     | Deep     | None     | None     |

---

## Business Units

### Cuándo usar Business Units

- Cuando la organización tiene divisiones con datos completamente separados (ej: sucursales,
  regiones, unidades de negocio).
- Cuando usuarios de una BU **no deben ver** datos de otra BU.
- Para implementar hierarchy security (manager ve datos del equipo).

### Estructura de BU típica para banca

```
Root BU (Cliente X)
  ├── BU: Retail Banking
  │     ├── BU: Sucursal Buenos Aires
  │     └── BU: Sucursal Córdoba
  ├── BU: Corporate Banking
  └── BU: Riesgo y Cumplimiento
```

### Generar XML de Business Unit (`generate_businessunit_xml`)

```xml
<BusinessUnit businessunitid="{BU-GUID}">
  <name>Retail Banking</name>
  <parentbusinessunitid>{ROOT-BU-GUID}</parentbusinessunitid>
  <description>Unidad de negocio de banca minorista</description>
</BusinessUnit>
```

---

## Teams

### Tipos de Teams

| `teamtype` | Tipo         | Cuándo usar                                                    |
|--------------|----------------|----------------------------------------------------------------------|
| `0`        | Owner Team   | El team puede ser owner de registros                           |
| `1`        | Access Team  | Acceso compartido a registros específicos (no owner)           |
| `2`        | AAD Security Group | Sincronizado desde Azure Active Directory              |
| `3`        | AAD Office Group   | Sincronizado desde Microsoft 365 Groups                  |

### Generar XML de Owner Team (`generate_team_xml`)

```xml
<Team teamid="{TEAM-GUID}">
  <name>Equipo Analistas de Crédito - Buenos Aires</name>
  <description>Team para analistas de crédito de la sucursal Buenos Aires</description>
  <teamtype>0</teamtype>
  <businessunitid>{BU-GUID}</businessunitid>
  <RoleAssociations>
    <RoleAssociation roleid="{ROLEID}" />
  </RoleAssociations>
</Team>
```

---

## Column Security (Field Level Security)

### Cuándo usar Column Security

- Campos con información sensible: CUIT, CBU, ingresos, score crediticio, dictamen.
- Campos que solo ciertos roles deben poder ver (ej: tasa de interés solo para gerentes).
- Campos que solo ciertos roles deben poder editar (ej: resultado de scoring solo para el
  motor IA).

### Habilitar Field Security en la columna

> **Excepción de canal:** habilitar `IsSecured: true` en una columna es un cambio de
> *metadata*, no de seguridad en sí — se resuelve vía `entity-builder` (canal DEV/Web API),
> no vía esta skill. `security-architect` empieza a partir de ahí: crea el profile y los
> permisos que gobiernan ese campo ya asegurado.

### Generar XML de Field Security Profile + permisos (`generate_fieldsecurity_xml`)

```xml
<fieldsecurityprofile fieldsecurityprofileid="{PROFILE-GUID}">
  <name>FSP - Datos Sensibles de Crédito</name>
  <description>Acceso a CUIT, CBU e ingresos declarados</description>
  <FieldPermissions>
    <FieldPermission attributelogicalname="axx_cuit" entityname="axx_creditsolicitud"
                      canread="1" cancreate="1" canupdate="1" />
  </FieldPermissions>
</fieldsecurityprofile>
```

| Valor | Significado |
|---------|---------------|
| `0`   | No permitido |
| `1`   | Permitido    |

---

## Matriz de seguridad — artefacto generado

Después de generar el XML del modelo de seguridad, la skill produce la **Matriz de
Seguridad** en Markdown para incluir en el PR (además de en el SDD vía `config-generator`):

```markdown
## Matriz de Seguridad — Cliente X Credit Onboarding

### Security Roles

| Rol                      | Scope datos  | axx_creditsolicitud (C/R/W/D) | Contact (R) | Dictamen (R/W) |
|------------------------------|----------------|------------------------------------|---------------|-------------------|
| Analista de Crédito      | Basic (Own)  | C✓ / R✓ / W✓ / D✗               | R-Local     | R✓ / W✗       |
| Supervisor de Crédito    | Local (BU)   | C✓ / R✓ / W✓ / D✗               | R-Local     | R✓ / W✓       |
| Gerente de Crédito       | Deep (BU+H)  | C✓ / R✓ / W✓ / D-Local           | R-Global    | R✓ / W✓       |
| Motor IA (Service Account)| Global      | C✓ / R✓ / W✓ / D✗               | R-Global    | R✓ / W✓       |

### Business Unit Hierarchy

Root BU → Retail Banking → Sucursal BA / Sucursal CBA
Root BU → Corporate Banking
Root BU → Riesgo y Cumplimiento
```

---

## Flujo completo (`commit_security_artifacts`)

```
Usuario describe el modelo de seguridad en lenguaje natural
        ↓
Skill genera la matriz + confirma con el usuario (siempre — es sensible)
        ↓
Skill genera los XML (roles, BU, teams, field security profiles)
        ↓
commit_security_artifacts → branch de la sesión en Azure DevOps Repos
        ↓
Orchestrator pide confirmación explícita para abrir el PR
        ↓
Review humano → merge → entra en el siguiente ciclo de promoción de solution-packager
```

---

## Restricciones

- **Nunca** crear roles con acceso Global (Organization) para usuarios de negocio — reservar
  Global para Service Accounts, integraciones y administradores de sistema.
- **Nunca** asignar el rol System Administrator a usuarios que no sean IT/equipo de
  implementación.
- **Nunca** ejecutar cambios de seguridad directo contra ningún environment — todo pasa por
  el commit + PR, sin excepción, incluso para DEV.
- Antes de generar el XML de `IsSecured: true` sobre un campo ya existente con datos,
  verificar (vía `list_column_security`) que ya existen los Field Security Profiles con
  permisos asignados — de lo contrario el campo queda invisible para todos tras el import.
- Si el environment usa Managed Environments (Power Platform Governance), verificar que los
  nuevos roles cumplen con las políticas de DLP antes de incluirlos en el PR.
- Documentar **siempre** cada cambio en el modelo de seguridad en la Matriz de Seguridad del
  PR y en el Solution Design Document del proyecto.
