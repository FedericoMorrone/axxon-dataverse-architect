# D365 Architect Agent — Repositorio

Repositorio interno de Axxon Consulting para el **D365 Architect Agent**: un plugin de Claude
Cowork con 9 skills para diseñar y construir soluciones sobre Microsoft Dataverse / Dynamics
365 CE, siguiendo el modelo de ejecución por canal (DEV/Web API, Git, Pipeline) documentado en
`docs/ADR-001` a `docs/ADR-003`.

**Estado actual: Release Candidate 1 (`v1.0.0-rc1`)**

---

## Estructura del repo

```
d365-architect-agent/
├── .claude-plugin/
│   └── marketplace.json          ← catálogo (marketplace) — un solo plugin listado
├── plugins/
│   └── d365-architect-agent/     ← el plugin instalable
│       ├── .claude-plugin/
│       │   └── plugin.json       ← manifiesto del plugin
│       ├── skills/                ← las 9 skills (d365-architect + 8 específicas)
│       ├── README.md              ← documentación funcional del agente
│       └── CHANGELOG.md
└── docs/                          ← decisiones de arquitectura (no son skills)
    ├── ADR-001-execution-architecture.md
    ├── ADR-002-topology.md
    ├── ADR-003-cowork-platform.md
    └── mcp-server.md              ← spec del backend (Azure Function MCP Server)
```

---

## Instalación para el equipo (Claude Cowork)

Cada consultor, al empezar un proyecto que use el agente, corre esto una vez:

```
/plugin marketplace add https://dev.azure.com/axxon/{Project}/_git/d365-architect-agent
/plugin install d365-architect-agent@axxon-d365-architect-agent
```

(Requiere tener configurado el acceso git a Azure DevOps — mismas credenciales que ya usás
para los repos de clientes.)

Para actualizar a una versión nueva del plugin más adelante:

```
/plugin update d365-architect-agent@axxon-d365-architect-agent
```

Instalación alternativa sin marketplace (por si alguien prefiere no agregarlo como fuente
persistente): descargar el repo y usar `claude --plugin-dir ./plugins/d365-architect-agent`
para esa sesión únicamente.

---

## Por qué Azure DevOps y no GitHub

Se evaluaron ambas opciones. Azure DevOps se eligió porque:
- El equipo ya tiene identidad y accesos ahí — cero fricción, cero cuenta nueva.
- Consistente con el resto de la infraestructura de Axxon (ver ADR-002).
- La instalación manual de un plugin (`/plugin marketplace add <url>`) funciona igual con
  cualquier host git, incluido Azure DevOps — no es una funcionalidad exclusiva de GitHub.

La única razón real para preferir GitHub sería habilitar el auto-provisioning silencioso de
un admin de Cowork Team/Enterprise para todo el equipo sin instalación manual — eso hoy
requiere que la fuente externa sea un repo público en GitHub/GitLab. Si en el futuro se
necesita ese modelo, se reevalúa entonces; no es el caso de uso actual.

---

## Para desarrollo del plugin en sí

Ver `plugins/d365-architect-agent/README.md` para el detalle funcional de cada skill, el
modelo de canales de ejecución, y los prerequisitos de despliegue del MCP Server por cliente.
