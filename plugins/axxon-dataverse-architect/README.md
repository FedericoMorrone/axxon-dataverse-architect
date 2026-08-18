# Axxon Dataverse Architect — Paquete de Skills (v1.2.0-rc1)

Skills de **Claude Cowork** especializadas en la construcción de soluciones sobre
**Microsoft Dataverse y Dynamics 365 CE** para Solution Architects.

> **Cambios de arquitectura:** ver `ADR-001-execution-architecture.md` (canales de
> ejecución), `ADR-002-topology.md` (topología del MCP Server), `ADR-003-cowork-platform.md`
> (pivot de plataforma: Copilot Studio → Claude Cowork), y `ADR-004-mcp-apps-adoption.md`
> (adopción de MCP Apps para vistas previas interactivas — MVP de 3 widgets en HTML/CSS/JS
> vanilla, sin React). Este README refleja el estado post-ADR-003; ADR-004 está **Aceptado**
> (diseño y stack resueltos), implementación todavía pendiente.

> **Release Candidate 1.** Cerrado para este RC: la tabla `axx_designsession` queda retirada
> por completo (el contexto conversacional vive en Cowork Projects), y la skill
> `output-formatter` (de otro paquete) queda formalmente descartada como dependencia — ver
> ADR-003 para el detalle de ambas decisiones.

---

## Contenido del paquete

| Archivo                       | Skill                  | Función                                                         | Canal principal |
|------------------------------------|--------------------------|---------------------------------------------------------------------|--------------------|
| `ADR-001-execution-architecture.md` | — (referencia) | Decisión arquitectónica: qué canal usa cada tipo de operación, y por qué | — |
| `ADR-002-topology.md` | — (referencia) | Decisión arquitectónica: topología del MCP Server (por cliente) y de Azure DevOps | — |
| `ADR-003-cowork-platform.md` | — (referencia) | Decisión arquitectónica: pivot Copilot Studio → Claude Cowork | — |
| `07-mcp-server.md`            | — (referencia, no es skill)   | Azure Function: cliente Web API (DEV) + cliente Azure DevOps (Repos+Pipelines) | — |
| `dataverse-architect/SKILL.md`     | dataverse-architect          | Skill conductora: routing por canal, contexto de proyecto, reglas transversales | — |
| `entity-builder/SKILL.md`     | entity-builder         | Tablas, columnas, relaciones, option sets                       | DEV / Web API |
| `form-designer/SKILL.md`      | form-designer           | Formularios Main/Quick Create/Quick View/Card                   | DEV / Web API |
| `business-rule-engine/SKILL.md` | business-rule-engine   | Business Rules, validaciones, visibilidad condicional           | DEV / Web API |
| `view-designer/SKILL.md`      | view-designer            | Views (SavedQuery): públicas, Quick Find, Advanced Find, Associated, Lookup | DEV / Web API |
| `duplicate-detection/SKILL.md`| duplicate-detection      | Alternate Keys y Duplicate Detection Rules                       | DEV / Web API |
| `solution-packager/SKILL.md`  | solution-packager      | Publishers, soluciones (DEV) + promoción DEV→TEST→PROD (pipeline) | DEV / Web API + Pipeline |
| `security-architect/SKILL.md` | security-architect      | Security Roles, Business Units, Teams, Column Security           | Git / XML de solución |
| `config-generator/SKILL.md`   | config-generator        | Deployment settings, SDD, deployment checklist          | Git |
| `app-composer/SKILL.md`       | app-composer            | App module, Sitemap, Dashboards, Charts                 | DEV / Web API |
| `genpage-builder/SKILL.md`    | genpage-builder         | Generative Pages (React 17 + TS + Fluent, **Preview** de Microsoft) | PAC CLI directo |
| `flow-builder/SKILL.md`       | flow-builder            | Power Automate cloud flows — vía FlowAgent MCP (servidor de Microsoft, no el propio) | MCP externo |
| `code-app-builder/SKILL.md`   | code-app-builder        | Power Apps Code Apps (React + Vite + TS, 1500+ connectors) — **verificar GA/Preview vigente** | PAC CLI / npx directo |

Todas las skills siguen el formato real de Claude Skills: carpeta propia con `SKILL.md`,
frontmatter mínimo (`name` + `description`, sin `version:` ni `triggers:` como campos
separados). Las versiones anteriores de un solo archivo quedaron en `_deprecated/` como
referencia histórica.

---

## Arquitectura de alto nivel

```
Arquitecto D365 (usuario)
        │ lenguaje natural
        ▼
┌───────────────────────────────────────────────────────────────────────┐
│  Claude Cowork — skill `dataverse-architect` + skills específicas          │
│  Clasifica cada operación por canal antes de actuar (ver ADR-001)     │
│  ┌────────────┐ ┌─────────────┐ ┌──────────────────┐                  │
│  │entity-     │ │form-        │ │business-rule-    │  canal: DEV/API  │
│  │builder     │ │designer     │ │engine             │                  │
│  └────────────┘ └─────────────┘ └──────────────────┘                  │
│  ┌────────────┐ ┌─────────────┐                                       │
│  │view-       │ │duplicate-   │                        canal: DEV/API │
│  │designer    │ │detection    │                                       │
│  └────────────┘ └─────────────┘                                       │
│  ┌────────────┐ ┌─────────────┐                                       │
│  │security-   │ │config-      │                          canal: Git   │
│  │architect   │ │generator    │                                       │
│  └────────────┘ └─────────────┘                                       │
│  ┌──────────────────────────────┐                                     │
│  │solution-packager               │        canal: DEV/API + Pipeline  │
│  └──────────────────────────────┘                                     │
└───────────────────────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼────────────────────────┐
        ▼                                          ▼
┌──────────────────────┐              ┌──────────────────────────────┐
│  MCP Server (Azure Fn) │              │  MCP Server (Azure Fn)        │
│  cliente Web API — DEV │              │  cliente Azure DevOps          │
└──────────────────────┘              │  (Git Repos + Pipelines API)   │
        │                              └──────────────────────────────┘
        ▼                                          │
┌──────────────────────┐                          ▼
│  Dataverse DEV         │              ┌──────────────────────────────┐
│  Metadata API·Solution │              │  Azure DevOps                  │
│  API                    │              │  Repo (XML de solución) +      │
└──────────────────────┘              │  Pipeline (pac pack/check/import)│
                                        └──────────────────────────────┘
                                                    │
                                    ┌───────────────┴───────────────┐
                                    ▼                                 ▼
                          Dataverse TEST (gate)              Dataverse PROD (gate manual,
                                                              nunca disparado por el agente)
```

---

## Prerequisitos para el despliegue

> **Topología (ver ADR-002):** cada cliente obtiene su propia instancia del MCP Server,
> desplegada desde una plantilla única que Axxon mantiene — nunca un runtime compartido entre
> clientes. Los ítems siguientes se repiten **por cliente**, no se resuelven una sola vez.

### Azure (por cliente)
- [ ] Azure Subscription — del cliente (default) o de Axxon (excepción documentada en ADR-002)
- [ ] Azure Function App dedicada a este cliente (Node.js 20, v4 runtime)
- [ ] Azure Key Vault dedicado a este cliente, con sus propios secretos
- [ ] App Registration en el Azure AD del cliente, con permisos Dataverse **acotados a DEV**
- [ ] Application Insights habilitado en la Function App, con `correlationId` end-to-end
- [ ] Azure Blob Storage — ya no requerido (ver `07-mcp-server.md`, eliminado en v2.0.0)

### Dataverse / Power Platform (por cliente)
- [ ] Environment DEV disponible
- [ ] Service Principal registrado como Application User con rol System Customizer **en DEV
      únicamente** — nunca en TEST/PROD
- [ ] Publisher `axx` creado en el environment DEV

### Azure DevOps (por cliente, dentro de la organización de Axxon — ver ADR-002)
- [ ] Project dedicado para este cliente (no se reutiliza el Project de otro cliente)
- [ ] Repo de Git para la solución de este cliente (source of truth de lo commiteado por
      `security-architect`/`config-generator`/`solution-packager`)
- [ ] Pipeline `{Proyecto}-CD` configurado con Power Platform Build Tools
- [ ] Environments de Azure DevOps (`TEST`, `PROD`) con gates de aprobación — PROD con
      aprobación manual obligatoria, sin excepción
- [ ] Service Connection / PAT del Project de este cliente, sin visibilidad desde otros
      Projects, con permisos de Code (Read & Write) y Build (Read & Execute)
- [ ] Excepción de organización de ADO separada evaluada y descartada (o aprobada) al iniciar
      el proyecto — ver criterio en ADR-002

### Claude Cowork (por cliente/proyecto — ver ADR-003)
- [ ] Cowork Project creado para este cliente (aislamiento de contexto entre clientes)
- [ ] Conector MCP agregado (Customize → Connectors) apuntando a la instancia del MCP Server
      de este cliente — ver `07-mcp-server.md`, sección "Registro del MCP Server como Conector
      en Cowork"
- [ ] Skill `dataverse-architect` y las skills específicas (`entity-builder`, `form-designer`, etc.)
      instaladas/habilitadas en el workspace de Cowork
- [ ] `.d365-session.md` inicial creado en el Project (o se genera automáticamente en el
      primer pedido, según `dataverse-architect/SKILL.md`)

---

## Convenciones del proyecto

### Nomenclatura

```
Publisher prefix:     axx
Table prefix:         axx_
Schema name:          axx_PascalCase
Logical name:         axx_lowercase
Solution unique name: {ClienteAbreviado}{Proyecto}{Módulo}
Environment Variable: axx_PascalCase
Security Role:        Axxon - {Nombre del Rol}
Business Unit:        {División / Área}
Team:                 {Área} - {Función} - {Ubicación}
```

### Versionado de soluciones

```
Major.Minor.Build.Revision
1.0.0.X  → Sprint de desarrollo en DEV
1.0.X.0  → Release a TEST (vía pipeline, con gate)
1.X.0.0  → Release menor a PROD (vía pipeline, con gate manual obligatorio)
X.0.0.0  → Release mayor (nueva arquitectura)
```

### Idioma en la configuración

- Labels de usuarios: **español** (LanguageCode: 3082)
- Nombres lógicos, schema names, logical names: **inglés** (sin espacios ni tildes)
- Documentación técnica: **español preservando términos técnicos en inglés**

---

## Flujo de trabajo típico con el agente (v3.0.0, Cowork)

```
1. Iniciar sesión → skill `dataverse-architect` pide (o lee de `.d365-session.md`) environment DEV,
   solución, publisher, repo+branch de ADO
2. Describir la entidad en lenguaje natural
   → entity-builder crea tabla + columnas + relaciones EN DEV (feedback inmediato)
3. Describir el formulario
   → form-designer crea el FormXml y publica EN DEV
4. Describir las reglas de negocio
   → business-rule-engine crea y activa las Business Rules EN DEV
5. Configurar el modelo de seguridad
   → security-architect genera XML + commitea a un branch (no toca DEV en vivo)
6. Pedirle al agente que prepare la promoción
   → solution-packager exporta DEV → repo, config-generator genera los deployment settings
   → `dataverse-architect` pide confirmación explícita → se abre el PR
7. Revisión humana del PR → merge
   → solution-packager dispara el pipeline (pac pack → check → import) hacia TEST
8. Tras validar en TEST, promoción a PROD
   → requiere gate de aprobación manual en Azure DevOps — el agente nunca lo dispara solo
9. Pedirle documentación
   → config-generator genera SDD + deployment checklist, con links a PRs y pipeline runs
```

---

## Compatibilidad

| Componente              | Versión mínima          |
|------------------------------|------------------------------|
| Dataverse Web API       | v9.2                    |
| Dynamics 365 CE         | Wave 1 2024 (9.2.24.x)  |
| Claude Cowork           | Última versión estable de Claude Desktop/web/mobile |
| Power Platform          | 2.0.x                    |
| Power Platform CLI      | Última estable (`pac install latest`) |
| Power Platform Build Tools (Azure DevOps) | Última estable |
| Azure Functions         | v4 (Node.js 20 LTS)     |
| MCP Protocol            | Última versión estable (Streamable HTTP + OAuth) |

---

## Mantenimiento

- Rotar el Client Secret del Service Principal (Dataverse DEV) y la credencial de Azure DevOps
  cada 12 meses — **por cliente**, ya que cada instancia tiene sus propios secretos.
- Actualizar la versión de la Azure Function runtime cuando Node.js 20 llegue a EOL (abril 2026)
  — pendiente de definir el mecanismo de propagación a todas las instancias activas (ver
  ADR-002, ítem abierto).
- Revisar la compatibilidad del MCP Server (protocolo, versión de tools expuestas) con cada
  actualización relevante de Cowork o del SDK de MCP.
- Actualizar los `LanguageCode` labels si el proyecto requiere soporte multi-idioma.
- Revisar periódicamente que ningún Service Principal usado por el agente tenga permisos
  fuera de DEV — es la invariante de seguridad central de esta arquitectura.
- Al iniciar un cliente nuevo: resolver explícitamente en discovery/Phase 0 quién hostea la
  instancia (cliente o Axxon) y si corresponde una organización de Azure DevOps separada — ver
  ADR-002.

---

*Generado por Axxon Dataverse Architect — Axxon Consulting*
*Versión del paquete: 1.0.0-rc1 — Agosto 2026*
