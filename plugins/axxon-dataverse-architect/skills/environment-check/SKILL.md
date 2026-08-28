---
name: environment-check
description: >
  Verifica que las herramientas de línea de comandos que necesitan `genpage-builder`,
  `code-app-builder`, y `azure-function-builder` (Node.js, PAC CLI, Azure Functions Core
  Tools, Azure CLI) estén instaladas y con la versión mínima requerida, antes de que
  cualquiera de esas 3 skills intente usarlas — reporta un consolidado y nunca instala nada
  sin confirmación explícita. Usar como paso previo obligatorio de esas 3 skills, o cuando el
  usuario pregunte directamente "¿tengo todo instalado para X?". Adaptado del patrón de
  orchestration.md (repo de skills de Antigravity) — deliberadamente NO se adoptó el
  mecanismo de autenticación de esas skills (az CLI + Device Code), que quedó superado por
  el servidor MCP oficial (ver ADR-001 de axxon-ado-connector).
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
| PAC CLI | `pac --version` | `genpage-builder`, `code-app-builder` | ≥2.7.0 (`genpage-builder` específicamente) |
| Azure Functions Core Tools | `func --version` | `azure-function-builder` | Cualquiera reciente |
| Azure CLI | `az --version` | `azure-function-builder`, opcionalmente `genpage-builder` (solo si crea entidades nuevas) | Cualquiera reciente |

## Flujo

1. Correr los 4 comandos de verificación de arriba.
2. Armar un reporte consolidado (tabla: herramienta, instalada sí/no, versión encontrada,
   cumple el mínimo sí/no).
3. Mostrárselo al usuario **antes** de invocar `genpage-builder`/`code-app-builder`/
   `azure-function-builder` — no arrancar la skill de construcción con algo faltante.
4. Si falta algo o la versión no alcanza: **nunca instalar automáticamente** — decirle al
   usuario qué falta y esperar confirmación explícita de que quiere proceder con la
   instalación (y aun con confirmación, la instalación en sí la hace el usuario en su
   Terminal, no vos por él — ver Restricciones).
5. Solo con las 4 herramientas relevantes a la skill que se va a usar confirmadas, proceder.

## Ejemplo de reporte consolidado

```markdown
# Verificación de entorno — antes de genpage-builder

| Herramienta | Instalada | Versión | Cumple mínimo |
|---|---|---|---|
| Node.js | ✅ | v20.11.0 | ✅ (≥18) |
| PAC CLI | ✅ | 1.34.2 | ❌ (necesita ≥2.7.0) |
| Azure CLI | — | — | No requerida para este pedido (no crea entidades nuevas) |

⚠️ PAC CLI está desactualizado — genpage-builder necesita ≥2.7.0. Actualizalo antes de
seguir: `pac install latest` (o el comando que corresponda a tu instalación).
```

## Restricciones

- Nunca instalar ni actualizar una herramienta por tu cuenta — mostrás el gap, el usuario
  decide y ejecuta la instalación en su propia Terminal.
- Nunca usar rutas hardcodeadas de instalación (ej. `C:\Program Files\...\az.cmd`) como
  fallback — si el comando no se encuentra en PATH, reportalo así, no adivines dónde puede
  estar instalado en la máquina de un usuario que no conocés.
- Nunca asumir el sistema operativo del usuario — los 4 comandos de verificación son
  multiplataforma (Node.js, PAC CLI, Functions Core Tools, y Azure CLI corren igual en
  Windows/Mac/Linux), no hace falta lógica específica por SO para esto.
- No reemplaza el gate de `.d365-project.md` (`axxon-orchestrator`) — son gates distintos:
  este verifica herramientas instaladas, aquel verifica datos del proyecto confirmados.
  Ambos pueden aplicar a la misma skill.
