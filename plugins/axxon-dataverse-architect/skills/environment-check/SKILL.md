---
name: environment-check
description: >-
  Verifica herramientas locales (Node.js, PAC CLI, .NET SDK, Git, Azure CLI, Functions Core
  Tools) antes de las skills de construcción. Solo reporta, nunca instala.
---

# Environment Check — Axxon Dataverse Architect

Verificás que las herramientas de CLI que 3 de nuestras skills necesitan estén instaladas
**antes** de que esas skills arranquen a trabajar y fallen a mitad de camino por falta de una
herramienta. No instalás nada — reportás y esperás confirmación.

---

## De dónde sale este diseño

Se adaptó el patrón de `orchestration.md` (un repo de skills de **Antigravity**, otro
framework de agentes, no Claude Code) — específicamente la idea de **verificar el entorno con
un reporte consolidado y un gate de autorización explícito antes de cualquier workflow**.

**Deliberadamente no se adoptó** el mecanismo de autenticación de las otras 2 skills de ese
mismo repo (`devops-login.md`, Device Code flow + `az` CLI para Azure DevOps) — ese enfoque
ya quedó superado en nuestro ecosistema por el servidor MCP oficial de Azure DevOps (ver
ADR-001 de `axxon-ado-connector`), que no necesita nada de la complejidad que esas skills
manejan (recarga de PATH, rutas de fallback hardcodeadas, troubleshooting de TF400813).

## Herramientas que verifica

| Herramienta | Comando de verificación | La necesita | Versión mínima |
|---|---|---|---|
| Node.js | `node --version` | `genpage-builder`, `code-app-builder` | LTS (≥18) |
| PAC CLI | `pac` sin argumentos (la versión sale en la 2.ª línea: `Version: 1.x.y+…`) | `genpage-builder`, `code-app-builder`, `plugin-builder`, `solution-packager`, `dataverse-connect` | Generative Pages: **la última** (Microsoft pide "Latest"); plug-in packages: ≥1.17. La numeración es **1.x** |
| Azure Functions Core Tools | `func --version` | `azure-function-builder` | Cualquiera reciente |
| Azure CLI | `az --version` | `azure-function-builder`, opcionalmente `genpage-builder` (solo si crea entidades nuevas) | Cualquiera reciente |

> PAC CLI no documenta `pac --version`: se usa `pac` sin argumentos. Desde la 1.32, `pac org`
> pasó a `pac env` (`org` sigue funcionando). Actualizar depende de cómo se instaló: .NET Tool
> → `dotnet tool update --global Microsoft.PowerApps.CLI.Tool`; MSI de Windows →
> `pac install latest`; extensión de VS Code → se actualiza con la extensión. Fuente:
> Microsoft Learn, *Install and configure Microsoft Power Platform CLI*.
| .NET SDK | `dotnet --version` | `plugin-builder`, `azure-function-builder` (build y tests) | El que soporte el target del proyecto (plugins: .NET Framework `net462` vía SDK-style; Functions: la versión de .NET del proyecto) |
| Git | `git --version` | `code-repo-manager` | Cualquiera reciente, con credenciales configuradas para el ADO del proyecto (no se verifican ni se piden: si un `git fetch` falla por auth, se reporta) |

## Dónde se verifica (host)

Los comandos corren donde corre el agente, que **no siempre es la máquina del usuario**: en
una sesión en la nube, un contenedor o una máquina remota, verificarías el host equivocado.
Antes de reportar:

1. Identificá el host: sistema operativo y nombre (`hostname`; en Windows también `ver`, en
   Linux/macOS `uname -a`).
2. Si hay señales de que no es el equipo del usuario — sesión en la nube o remota, contenedor,
   un SO distinto del que el usuario dijo usar, rutas tipo `/home/user` o `/workspace` cuando
   el usuario trabaja en Windows —, **decilo en el reporte**: "Esto verifica el entorno donde
   corro yo, no tu máquina". Y si las skills de construcción van a correr en tu máquina (no
   acá), pedile al usuario que corra él los comandos y te pase la salida.
3. El reporte siempre incluye una línea `Host verificado: {SO} — {hostname}`.

## Flujo

1. Correr los comandos de verificación de las herramientas que necesita la skill que se va a
   usar (no todas siempre).
2. Armar un reporte consolidado (tabla: herramienta, instalada sí/no, versión encontrada,
   cumple el mínimo sí/no), con la línea de host.
3. Mostrárselo al usuario **antes** de invocar `genpage-builder`/`code-app-builder`/
   `azure-function-builder` — no arrancar la skill de construcción con algo faltante.
4. Si falta algo o la versión no alcanza: **nunca instalar automáticamente** — decirle al
   usuario qué falta y esperar confirmación explícita de que quiere proceder con la
   instalación (y aun con confirmación, la instalación en sí la hace el usuario en su
   Terminal, no vos por él — ver Restricciones).
5. Solo con las herramientas relevantes a la skill que se va a usar confirmadas, proceder.

## Ejemplo de reporte consolidado

```markdown
# Verificación de entorno — antes de genpage-builder

Host verificado: Windows 11 — LAPTOP-CONSULTOR (máquina del usuario)

| Herramienta | Instalada | Versión | Cumple mínimo |
|---|---|---|---|
| Node.js | ✅ | v20.11.0 | ✅ (≥18) |
| PAC CLI | ✅ | 1.34.2 | ⚠️ Generative Pages pide la última versión — confirmá que no haya una más nueva |
| Azure CLI | — | — | No requerida para este pedido (no crea entidades nuevas) |

⚠️ Generative Pages requiere PAC CLI en su última versión. Actualizalo antes de seguir con
el comando que corresponda a tu instalación (.NET Tool: `dotnet tool update --global
Microsoft.PowerApps.CLI.Tool`; MSI: `pac install latest`).
```

## Restricciones

- Nunca instalar ni actualizar una herramienta por tu cuenta — mostrás el gap, el usuario
  decide y ejecuta la instalación en su propia Terminal.
- Nunca usar rutas hardcodeadas de instalación (ej. `C:\Program Files\...\az.cmd`) como
  fallback — si el comando no se encuentra en PATH, reportalo así, no adivines dónde puede
  estar instalado en la máquina de un usuario que no conocés.
- Nunca asumir el sistema operativo del usuario — los comandos de verificación son
  multiplataforma, no hace falta lógica específica por SO para esto. (Ojo: compilar plugins
  `net462` fuera de Windows requiere los reference assemblies de .NET Framework — si el
  build falla por eso, reportalo, no lo "arregles" cambiando el target.)
- No reemplaza el gate de `.d365-project.md` (`axxon-orchestrator`) — son gates distintos:
  este verifica herramientas instaladas, aquel verifica datos del proyecto confirmados.
  Ambos pueden aplicar a la misma skill.
