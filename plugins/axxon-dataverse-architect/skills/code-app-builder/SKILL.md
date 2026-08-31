---
name: code-app-builder
description: >
  Crea y despliega Power Apps Code Apps — aplicaciones React + Vite + TypeScript que corren
  dentro de Power Apps, conectadas al Power Platform vía connectors estándar (Dataverse,
  SharePoint, Teams, Azure DevOps, OneDrive, Excel, Office 365, y más de 1500 más), con
  autenticación Microsoft Entra. Usar cuando el usuario pida una app pro-code conectada a
  Power Platform, necesite acceso a un connector que Dataverse solo no cubre, o mencione
  "code app". Adaptado del enfoque oficial de Microsoft (repo microsoft/PowerAppsCodeApps y
  microsoft/power-platform-skills). Verificar el estado GA/Preview vigente antes de
  comprometerse con el cliente — las fuentes disponibles al momento de escribir esta skill no
  eran consistentes entre sí. No usar para páginas dentro de una app model-driven existente
  (eso es `genpage-builder`) ni para tablas/forms de Dataverse (eso es `entity-builder`/
  `form-designer`).
---

# Code App Builder — Axxon Dataverse Architect

Construís **Power Apps Code Apps** — apps pro-code (React + Vite + TypeScript) que corren
gestionadas dentro de Power Platform, con acceso al catálogo completo de connectors. Es la
vía para cuando el cliente necesita algo que Dataverse + Forms/Views estándar no puede cubrir
por sí solo (un connector específico, una lógica de UI compleja, integración con un sistema
que ya tiene connector certificado).

## Antes que nada — verificar el entorno

Corré `environment-check` primero (Node.js, PAC CLI) — no arranques el scaffolding con una
herramienta faltante.

## Antes que nada (2) — verificar el estado GA/Preview

Las fuentes que encontré al construir esta skill **no eran consistentes**: el repo oficial del
producto (`microsoft/PowerAppsCodeApps`) dice que Code Apps ya está en **General
Availability**, pero otras fuentes (incluyendo guías de la propia comunidad de Microsoft) todavía
lo describen como Preview — probablemente la feature en sí ya sea GA pero el tooling de
scaffolding/deploy vía skills siga evolucionando. **Confirmá el estado vigente antes de
comprometerte con un cliente** — no repitas ninguna de las dos afirmaciones como un hecho
sin chequear.

## Stack técnico (real, no adaptado)

- **React + Vite + TypeScript**, corriendo en el puerto 3000 (requerido por el SDK).
- **`@microsoft/power-apps` SDK** para inicialización y acceso a connectors.
- **PAC CLI** (`pac code init`, `pac code run`) + herramientas `npx power-apps` para
  scaffolding, conexión de data sources, y deploy.

## Scaffolding — regla dura, no negociable

**Siempre `npx degit` para crear un proyecto nuevo — nunca `git clone` ni archivos a mano.**
Microsoft lo marca como regla explícita en su propia guía, no una preferencia de estilo:

```bash
npx degit microsoft/PowerAppsCodeApps/templates/vite {carpeta} --force
```

## Estructura del proyecto

```
src/
├── components/     # UI reutilizable
├── services/       # Servicios de connector — generados por PAC CLI, no a mano
├── models/         # Modelos TypeScript — generados por PAC CLI, no a mano
├── hooks/          # Hooks custom para integración con Power Platform
├── utils/
├── types/
├── PowerProvider.tsx  # Inicialización de Power Platform
└── main.tsx
```

`services/` y `models/` **los genera PAC CLI** al agregar un connector o data source — no los
escribas a mano, se regeneran y perderías el trabajo.

## Flujo

1. Scaffolding: `npx degit microsoft/PowerAppsCodeApps/templates/vite`.
2. `pac code init` para inicializar metadata de Power Platform.
3. Agregar data sources/connectors: `npx power-apps add-dataverse` / `npx power-apps
   add-connector` (Teams, Excel, SharePoint, etc.) — confirmá con el usuario qué connectors
   necesita antes de agregar de más.
4. Desarrollo local: `pac code run` + `vite` en paralelo (`concurrently "vite" "pac code
   run"`).
5. Deploy: `npx power-apps push` — confirmá el environment destino antes de correrlo.

## Cómo encaja en el modelo de canales de Axxon

Igual que `genpage-builder`: canal propio, PAC CLI/npx directo contra el environment — no
pasa por el D365 Architect MCP Server de Axxon. La promoción a TEST/PROD, una vez la Code App
queda como componente de la solución, sigue el canal Pipeline de `solution-packager`.

## Restricciones

- Nunca `git clone` ni archivos a mano para el scaffolding inicial — siempre `npx degit`.
- Nunca edites `services/`/`models/` a mano — son generados, se van a sobreescribir.
- Nunca despliegues (`npx power-apps push`) sin confirmar el environment destino primero.
- Nunca afirmes el estado GA/Preview de la feature sin haberlo verificado en el momento — las
  fuentes eran contradictorias al escribir esta skill.
