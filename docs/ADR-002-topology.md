# ADR-002 — Topología del MCP Server y de Azure DevOps

**Estado:** Aceptado
**Fecha:** 2026-08-05
**Contexto del proyecto:** Axxon Dataverse Architect — Axxon Consulting
**Depende de:** ADR-001 (modelo de ejecución)

> **Nota de actualización (ver ADR-003):** esta decisión de topología (una instancia de MCP
> Server por cliente) sigue vigente sin cambios. Lo que cambió es el mecanismo por el cual el
> agente invoca esa instancia — de Custom Connector de Copilot Studio a Conector MCP nativo de
> Cowork. Las referencias a "Copilot Studio" en el cuerpo de este documento reflejan el
> contexto en el que se tomó la decisión originalmente; el razonamiento de fondo (aislamiento
> por cliente) no cambia y, de hecho, se refuerza bajo Cowork — ver ADR-003, Decisión 3.

---

## Contexto

ADR-001 definió *qué* canal usa cada operación (DEV/Web API, Git, Pipeline) pero dejó abierto
*dónde* corre la infraestructura que ejecuta esos canales — un único MCP Server compartido
entre todos los clientes de Axxon, o una instancia por cliente. Sin resolver esto, no se puede
completar el `07-mcp-server.md` con una guía de despliegue real, ni presupuestar el trabajo de
ALM con clientes bancarios.

Restricción de partida que ya condiciona la respuesta: el agente de Copilot Studio de cada
proyecto **ya vive dentro del tenant Azure AD del cliente** — no hay alternativa, porque
necesita un Application User dado de alta en el Dataverse de ese cliente específico (ver
`00-orchestrator.md`, "Información requerida al inicio de sesión"). El Custom Connector que
conecta Copilot Studio con el MCP Server también se registra en ese tenant. Esto ya empuja
fuertemente hacia una topología por cliente, no hacia un servicio compartido.

## Decisión

### 1. MCP Server: **una instancia por cliente**, desplegada por IaC desde una plantilla única

- Axxon mantiene **un solo repositorio fuente** (código del MCP Server + Bicep/Terraform de
  infraestructura) — no se reimplementa por cliente.
- Cada cliente obtiene su **propia Azure Function App**, su propio Key Vault, su propio
  Service Principal con permisos acotados a su(s) environment(s) DEV — desplegados desde ese
  mismo template vía pipeline, parametrizado por cliente (nombre, tenant ID, environment URL).
- **Ubicación de hosting, por defecto: la suscripción Azure del cliente**, no la de Axxon.
  Justificación: en banca, el cliente típicamente exige que cualquier componente con acceso a
  su Dataverse (aunque sea acotado a DEV) resida bajo su propio control de suscripción,
  auditoría y políticas de red — no bajo un proveedor externo compartido con otros bancos.
  - **Excepción documentada:** si el cliente no tiene capacidad/interés en operar
    infraestructura propia (típico en proyectos más chicos, no bancarios), se aloja en la
    suscripción de Axxon, pero **siempre como Function App dedicada a ese cliente** — nunca
    compartiendo runtime con otro cliente.
- **Se descarta explícitamente** un MCP Server multi-tenant compartido que sirva a varios
  bancos desde un único runtime. Razones:
  1. Un solo compromiso de seguridad afecta a más de un cliente FSI simultáneamente —
     blast radius inaceptable para el perfil de riesgo de estos proyectos.
  2. La revisión de seguridad de un banco típicamente **no aprueba** que su Service Principal
     y sus tokens convivan en la misma Application Insights / Key Vault que los de otro banco,
     sin importar cuán bien particionados estén lógicamente.
  3. El ahorro operativo de un runtime compartido no compensa el costo de una eventual
     negociación de seguridad fallida con un cliente bancario — el checklist de deployment
     (`06-config-generator.md`) ya asume, por diseño, que cada ambiente es independiente.

### 2. Azure DevOps: **una organización de Axxon, un Project por cliente**

- No se crea una organización de Azure DevOps separada por cliente por defecto — Axxon opera
  una organización propia con **un Project dedicado por cliente/proyecto** (mismo patrón que ya
  usás hoy para `ado-daily-prep`/`ado-us-planner`).
- Cada Project tiene su propio Repo, su propio Pipeline, y su propio **Service Connection**
  hacia los environments Dataverse de ese cliente — nunca una Service Connection compartida
  entre proyectos de distintos clientes.
- Los secretos (Client Secret del Service Principal DEV, credencial de Azure DevOps del MCP
  Server) viven en **Variable Groups o Key Vault por Project**, con acceso restringido al
  equipo asignado a ese cliente — no a nivel de organización completa.
- **Excepción documentada:** si un contrato con un cliente bancario exige explícitamente una
  organización de Azure DevOps separada (aislamiento contractual, no solo técnico), se crea
  una organización dedicada para ese cliente. Es la excepción, no el default — se evalúa
  caso por caso al firmar el proyecto, no se decide en este documento de forma anticipada.

### 3. Consecuencia sobre el repositorio de la solución (fuente de verdad de ADR-001)

El repo de "solución como código" que aparece en ADR-001 (branch/PR con el XML de seguridad,
env variables, etc.) es el **repo del Project de ese cliente específico** — no un repo
monolítico de Axxon con carpetas por cliente. Esto refuerza el aislamiento: nadie con acceso
al Project del Cliente Y puede ver el histórico de commits del Cliente X.

---

## Consecuencias

- El trabajo de "implementar el MCP Server" deja de ser un proyecto único — es una **plantilla
  reutilizable** (código + IaC) que se instancia por cliente. El esfuerzo real de Axxon está en
  mantener la plantilla, no en reimplementar por proyecto.
- El onboarding de un cliente nuevo incluye, como paso estándar, la decisión explícita de
  "¿quién hostea la infraestructura del agente?" — se agrega como pregunta obligatoria en la
  fase de discovery/Phase 0 de cualquier proyecto que use este agente.
- El deployment checklist de `06-config-generator.md` y los prerequisitos del `README.md` se
  actualizan para reflejar que cada cliente tiene su propio Project de Azure DevOps y su propia
  instancia de MCP Server — no una infraestructura compartida a completar una sola vez.
- El costo operativo de Axxon escala linealmente con la cantidad de clientes activos (N Function
  Apps, N Key Vaults) — se acepta conscientemente a cambio del aislamiento; se mitiga
  manteniendo el template lo más delgado y automatizado posible (ver `07-mcp-server.md`,
  sección de despliegue).

## Pendiente para las próximas iteraciones

- Definir el pipeline "meta" que Axxon usa internamente para actualizar la plantilla del MCP
  Server en todas las instancias de clientes activos cuando se libera una nueva versión (parche
  de seguridad, por ejemplo) — hoy no está diseñado, y con N instancias por cliente se vuelve
  necesario más pronto que tarde.
- Definir el criterio exacto (tamaño de cliente, sector, cláusula contractual) que dispara la
  excepción de organización de Azure DevOps separada — hoy queda como juicio caso a caso.
