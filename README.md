# Axxon Dataverse Architect — Repositorio

Repositorio interno de Axxon Consulting para el **Axxon Dataverse Architect**: un plugin de Claude
Cowork con 9 skills para diseñar y construir soluciones sobre Microsoft Dataverse / Dynamics
365 CE, siguiendo el modelo de ejecución por canal (DEV/Web API, Git, Pipeline) documentado en
`docs/ADR-001` a `docs/ADR-003`.

**Estado actual: Release Candidate 1 (`v1.0.0-rc1`)**

**Ubicación:** Azure DevOps, organización `FeMorrone`, Project **Axxon Dataverse Architect**.
El repo se llama igual que el Project (con espacios) — Azure DevOps lo crea así por defecto
al crear el Project, y no hace falta renombrarlo para que todo funcione: las URLs solo llevan
`%20` en vez de guiones.

---

## Primera subida (una sola vez)

Desde la carpeta descomprimida del repo:

```bash
git remote add origin "https://FeMorrone@dev.azure.com/FeMorrone/Axxon%20Dataverse%20Architect/_git/Axxon%20Dataverse%20Architect"
git push -u origin main
git push origin v1.0.0-rc1
```

(Te va a pedir autenticación — usá "Generate Git Credentials" desde la pantalla de Repos en
Azure DevOps si todavía no tenés un PAT configurado.)

---

## Estructura del repo

```
axxon-dataverse-architect/
├── .claude-plugin/
│   └── marketplace.json          ← catálogo (marketplace) — un solo plugin listado
├── plugins/
│   └── axxon-dataverse-architect/     ← el plugin instalable
│       ├── .claude-plugin/
│       │   └── plugin.json       ← manifiesto del plugin
│       ├── skills/                ← las 9 skills (dataverse-architect + 8 específicas)
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
/plugin marketplace add "https://dev.azure.com/FeMorrone/Axxon%20Dataverse%20Architect/_git/Axxon%20Dataverse%20Architect"
/plugin install axxon-dataverse-architect@axxon-dataverse-architect-marketplace
```

(Requiere tener configurado el acceso git a Azure DevOps — mismas credenciales que ya usás
para los repos de clientes.)

Para actualizar a una versión nueva del plugin más adelante:

```
/plugin update axxon-dataverse-architect@axxon-dataverse-architect-marketplace
```

Instalación alternativa sin marketplace (por si alguien prefiere no agregarlo como fuente
persistente): descargar el repo y usar `claude --plugin-dir ./plugins/axxon-dataverse-architect`
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

Ver `plugins/axxon-dataverse-architect/README.md` para el detalle funcional de cada skill, el
modelo de canales de ejecución, y los prerequisitos de despliegue del MCP Server por cliente.
