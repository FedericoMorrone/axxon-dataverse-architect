# ADR-003 — Pivot de plataforma: Copilot Studio → Claude Cowork

**Estado:** Aceptado
**Fecha:** 2026-08-05
**Contexto del proyecto:** D365 Architect Agent — Axxon Consulting
**Depende de / modifica:** ADR-001 (modelo de ejecución), ADR-002 (topología)

---

## Contexto

Las v1.0.0–v2.2.0 del agente asumían Copilot Studio como plataforma de ejecución: "skills"
como Topics con NLU propio, un Custom Connector registrado en AI Capabilities para hablar con
el MCP Server, y Azure AD como único mecanismo de auth entre el agente y el backend.

El agente se va a ejecutar en **Claude Cowork** en su lugar. Cowork no es un chatbot con
Topics — es la misma arquitectura agéntica de Claude Code (bash, archivos, sub-agentes,
Skills reales) empaquetada en una interfaz de escritorio/web/mobile. Esto no es un simple
find-and-replace de "Copilot Studio" por "Cowork": cambia quién orquesta, cómo se cargan las
"skills", cómo se autentica contra el backend, y — potencialmente — si algunas piezas del
backend siguen siendo necesarias.

---

## Decisión 1 — Formato de las skills: Copilot-Studio-style → Claude Skill real

El formato que usamos hasta v2.2.0 (`name`/`version`/`description`/`triggers:` como lista)
era una convención propia inspirada en Copilot Studio, no el formato real de Claude Skills.
El formato oficial (`anthropics/skills`, verificado contra la documentación vigente) es:

```
skill-name/                    ← kebab-case, igual al nombre de carpeta
├── SKILL.md                   ← obligatorio
│   ├── YAML frontmatter       ← solo `name` y `description` son obligatorios
│   └── Instrucciones Markdown ← lo que Claude lee cuando la skill se activa
├── scripts/                   ← opcional — código ejecutable (Python/Bash)
├── references/                ← opcional — docs que se cargan solo si hacen falta
└── assets/                    ← opcional — plantillas, templates de salida
```

Diferencias clave con lo que veníamos escribiendo:

- **No existe un campo `triggers:` separado.** Las frases disparadoras van *dentro* de
  `description` — es el único texto que Claude ve antes de decidir si carga la skill completa,
  así que tiene que ser explícito y, según la guía oficial de autoría, "un poco insistente"
  (mencionar literalmente los términos que un usuario tipearía, no solo la categoría general).
- **No existe un campo `version:` de primera clase** en el estándar portable — si se quiere
  trackear versión, va dentro de `metadata:` (opcional) o se documenta aparte (como venimos
  haciendo con los ADRs). Mantenemos el versionado en el README del paquete, no en cada
  frontmatter, para no depender de un campo no estándar.
- **Límite de 1024 caracteres en `description`**, sin tags XML.
- Las tablas de MCP Tools, ejemplos de payload, restricciones — todo el contenido técnico que
  ya construimos — se mantiene igual. Lo que cambia es el envoltorio (frontmatter) y dónde vive
  cada skill (carpeta propia en vez de un único archivo `0N-nombre.md`).

## Decisión 2 — Orquestación: Topic raíz de Copilot Studio → skill "conductora" + Claude nativo

Copilot Studio necesitaba un Topic/agente raíz explícito (`00-orchestrator`) porque el NLU y el
routing entre Topics son mecanismos propios de esa plataforma. Cowork no tiene ese concepto:
**Claude mismo decide, por descripción, qué skill(s) cargar** para una tarea — igual que hace
ahora mismo con las skills que ya tenés instaladas (`d365-requirements-analyst`,
`ado-daily-prep`, etc.).

Pero perder el Topic raíz no significa perder las reglas *transversales* que vivían ahí: el
routing por canal (ADR-001), el patrón confirm-before-execute, el manejo de errores común, las
restricciones de PROD. Esas reglas no pertenecen a ninguna skill específica — si las
repetimos en cada una, se desincronizan con el tiempo.

**Decisión:** se mantiene una skill "conductora" — `d365-architect/SKILL.md` — con una
descripción diseñada para activarse en *cualquier* pedido relacionado con Dataverse/D365 CE,
que documenta las reglas transversales y remite a la skill específica correspondiente. No es
un Topic que "delega" en el sentido de Copilot Studio — es una skill más que Claude carga
junto con las demás cuando corresponde, y que actúa como fuente de verdad de las reglas
compartidas. Reemplaza a `00-orchestrator.md`.

## Decisión 3 — Consumo del MCP Server: Custom Connector → Conector MCP nativo de Cowork

Esto es lo que **no** cambia de fondo: ADR-002 (una instancia de MCP Server por cliente,
alojada preferentemente en la suscripción del cliente) sigue siendo la decisión correcta —
de hecho queda **más justificada** todavía. Cowork puede correr en sesiones locales, en la
notebook de cada consultor de Axxon; si el acceso a Dataverse de cada banco dependiera de
secretos guardados en la notebook de cada consultor, un solo dispositivo comprometido
expondría el Service Principal de un cliente bancario. Enrutar toda escritura a Dataverse (y
toda operación de Git/Pipeline) a través del MCP Server remoto, con sus propios secretos en
Key Vault, es hoy más importante que antes, no menos.

Lo que cambia es **cómo Cowork llama al MCP Server**:

| | Antes (Copilot Studio) | Ahora (Cowork) |
|---|---|---|
| Mecanismo de conexión | Custom Connector (Swagger 2.0) registrado en Settings → AI Capabilities | Conector MCP remoto (Streamable HTTP + OAuth) agregado en Customize → Connectors |
| Autenticación inbound | Token emitido por Copilot Studio, validado por audience/appid | Token emitido por el flujo OAuth del conector de Cowork, validado igual (firma + audience) contra el mismo App Registration |
| Descubrimiento de tools | Registro manual de la spec Swagger | Descubrimiento nativo del protocolo MCP (`tools/list`) — **ya no hace falta mantener la spec OpenAPI/Swagger 2.0** |
| Spec a mantener | `06-config-generator.md` generaba YAML Swagger 2.0 | El propio MCP Server expone su tool registry vía el protocolo MCP estándar — se elimina esa sección de `config-generator` |

El código del MCP Server (`dataverseClient.ts`, `azureDevOpsClient.ts`, `authService.ts`,
`retryPolicy.ts`) definido en `07-mcp-server.md` **se mantiene sin cambios de fondo** — sigue
siendo la Azure Function que expone tools vía MCP. Cambia el consumidor (Cowork en vez de
Copilot Studio) y desaparece la necesidad de la capa Swagger/Custom Connector.

## Decisión 4 — Estado de sesión: tabla Dataverse → Cowork Projects

`axx_designsession` existía para resolver un problema específico de Copilot Studio: varias
conversaciones concurrentes sin aislamiento nativo entre proyectos de distintos clientes.
**Cowork resuelve esto de forma nativa con Projects** (espacios de trabajo persistentes, con
sus propios archivos, memoria e instrucciones). La forma natural de mapear esto:

- **Un Cowork Project por cliente/proyecto** (ej: "Cliente X — Credit Onboarding"), igual que
  ya definimos un Azure DevOps Project por cliente en ADR-002 — mismo criterio de aislamiento,
  ahora también a nivel de la herramienta que usa el consultor.
- El contexto conversacional (environment DEV, solución, publisher, branch activo) vive como
  un archivo `.d365-session.md` dentro de ese Project. Se resuelve con las herramientas de
  archivo nativas de Cowork, sin tool MCP dedicado.

**Actualización para RC1:** `axx_designsession` se **retira por completo** — no se conserva ni
siquiera como log de auditoría opcional. Cowork Projects (archivos + memoria nativa) es la
única fuente de contexto conversacional. Si en el futuro un proyecto puntual necesita un log
de auditoría regulatorio fuera de Cowork, se evalúa como requerimiento específico de ese
proyecto, no como parte del paquete base. Consecuencia directa: el MCP Server ya no necesita
un `sessionService.ts` — se retira de `07-mcp-server.md` (ver Consecuencias).

## Decisión 5 — Confirmaciones: texto "¿Confirmás? (sí/no)" → modelo de permisos de Cowork

Cowork ya tiene su propio mecanismo de aprobación de acciones (paso a paso vs. modo Auto,
permisos por conector, confirmación antes de acciones irreversibles). El patrón
confirm-before-execute que documentamos en `d365-architect/SKILL.md` (heredado de
`00-orchestrator.md`) se mantiene como **instrucción explícita dentro de la skill** — le dice a
Claude qué acciones tratar como sensibles incluso si Cowork en modo Auto no las marcaría por sí
solo (ej: disparar un pipeline hacia PROD). No reemplaza el mecanismo nativo de Cowork; lo
complementa, siendo más estricto que el default en las acciones que este dominio considera
críticas.

## Decisión 6 — Tentación evitada: correr `pac` CLI directo desde el sandbox de Cowork

Cowork sí puede ejecutar bash/Python en su sandbox (sesión local o en la nube), lo que técnicamente
habilita instalar y correr `pac` CLI ahí mismo — algo que **no** era posible con la Azure
Function de Copilot Studio (serverless, sin lugar para un CLI persistente). Aun así, **no**
se adopta ese atajo para operaciones contra environments reales: el modelo de gates de ADR-001
(pack → check → import solo dentro del pipeline de Azure DevOps, con aprobación humana antes de
TEST/PROD) se mantiene intacto. Si Cowork corriera `pac solution import` directo desde la
notebook de un consultor, se pierde el PR review y el gate de aprobación que es la razón de
ser de ADR-001.

**Excepción permitida:** correr `pac solution check` localmente como chequeo rápido antes de
commitear, si el consultor tiene PAC CLI instalado — es un adelanto de lo que igual va a
correr en el pipeline, no un reemplazo del gate oficial.

---

## Consecuencias

- Se retira la sección de generación de spec Swagger 2.0 / Custom Connector de
  `06-config-generator.md` — ya no aplica.
- `00-orchestrator.md` se retira y se reemplaza por `d365-architect/SKILL.md`, con el mismo
  contenido de reglas transversales adaptado al formato real de Claude Skills.
- Cada skill 01–09 se reorganiza en su propia carpeta (`entity-builder/SKILL.md`,
  `form-designer/SKILL.md`, etc.), con frontmatter mínimo (`name` + `description`) y el
  contenido técnico existente preservado como cuerpo Markdown.
- `07-mcp-server.md` **no es una skill** — sigue siendo documentación de infraestructura para
  quien implemente el backend, no algo que Cowork carga en una conversación. Se mantiene como
  archivo de referencia junto a los ADRs, fuera de la carpeta de skills.
- Se recomienda un Cowork Project por cliente como unidad de aislamiento — a definir junto con
  el equipo si esto se documenta como policy o queda a criterio del consultor asignado.
- ADR-001 y ADR-002 **no se revisan** — sus decisiones de canal y topología siguen vigentes;
  solo cambia el mecanismo de invocación (Decisión 3) y el mecanismo de contexto (Decisión 4).

## Pendiente para las próximas iteraciones

> **Cerrado para RC1:** la migración de skills a formato de carpeta (hecha) y la decisión
> sobre `axx_designsession` (retirada por completo, ver Decisión 4 actualizada) ya no son
> pendientes — quedan resueltas en este release candidate.

- Confirmar en la documentación oficial de Cowork vigente al momento de implementar si hay
  restricciones adicionales de red para sesiones en la nube que afecten la conexión al MCP
  Server (las sesiones en la nube tienen egress restringido salvo allowlist del admin).
- La dependencia de la skill `output-formatter` (fuera de este paquete) queda formalmente
  descartada para este agente — `d365-architect` no la referencia ni la requiere en ningún
  punto (ver su sección "Output").
