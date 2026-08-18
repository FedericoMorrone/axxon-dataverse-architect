---
name: genpage-builder
description: >
  Crea, actualiza, y despliega Generative Pages (genux) — páginas React 17 + TypeScript +
  Fluent UI V9 embebidas nativamente (sin iframe) dentro de una app model-driven de Dynamics
  365 / Power Platform, vía PAC CLI. Usar cuando el usuario pida una página custom dentro de
  la app model-driven, un dashboard/grid con código a medida, una página con lógica que
  Business Rules/Views no puedan cubrir, o mencione "generative page", "genpage", o "genux".
  Adaptado del enfoque oficial de Microsoft (repo microsoft/power-platform-skills, plugin
  model-apps). **Feature en Preview de Microsoft** — confirmar con el cliente antes de
  usarla en un engagement real, no asumir que es GA. No usar para tablas/forms/vistas
  estándar (eso es `entity-builder`/`form-designer`/`view-designer`) ni para el módulo de App
  y su Sitemap (eso es `app-composer`).
---

# Generative Page Builder — Axxon Dataverse Architect

Generás páginas React con código real (no low-code) que corren **nativamente dentro del
shell de la app model-driven** — sin iframe, con acceso directo al contexto de Dataverse. Es
la vía moderna para cubrir necesidades de UI que Forms/Views estándar no resuelven bien
(dashboards con lógica custom, grids con interacciones complejas, vistas cruzando varias
tablas con una experiencia a medida).

## Antes que nada — es Preview de Microsoft

Esta capacidad está en **Preview** (no GA) según la documentación oficial de Microsoft
(learn.microsoft.com/power-apps/maker/model-driven-apps/generative-pages). Antes de usarla en
un proyecto real:

1. Confirmá con el usuario que el cliente acepta usar una feature en preview — no lo asumas.
2. Documentalo explícitamente en el FDD del GAP correspondiente (Fase 1) si no está ya.
3. Preferí Web Resources/Custom Pages (la vía madura y soportada) si el cliente tiene
   restricciones de compliance sobre usar features en preview.

## Stack técnico (real, no adaptado)

- **React 17 + TypeScript**, un archivo `.tsx` por página con `export default GeneratedComponent`.
- **Fluent UI V9** (`@fluentui/react-components`) — únicos íconos permitidos son los de
  `@fluentui/react-icons` verificados; no importar de otras librerías de íconos.
- **DataAPI tipada** — operaciones CRUD contra tablas de Dataverse vía `props.dataApi`, nunca
  fetch directo a la Web API desde la página.
- **PAC CLI** para todo: generación de schema y deployment. **Prerequisitos**: Node.js LTS,
  PAC CLI ≥ 2.7.0, Azure CLI (`az login`, mismo tenant que `pac auth list`) — este último
  **solo** si la página necesita crear entidades nuevas; si trabaja sobre tablas ya
  existentes, no hace falta `az`.

## Flujo (adaptado del oficial: Clarify → Generate → Deploy → Verify)

1. **Prerequisitos** — verificá versión de Node.js y PAC CLI (`pac --version` ≥ 2.7.0).
2. **Autenticación** — confirmá `pac auth list` apunta al environment correcto de
   `.d365-project.md`; si hace falta crear entidades, confirmá también `az login`.
3. **Clarificar requerimiento** — tipo de página (grid, dashboard, formulario custom), fuente
   de datos (tabla existente vs. mock), features específicas. No asumas — preguntá si no está
   claro en la Historia de Usuario que originó el pedido.
4. **Crear entidades** (opcional) — solo si las tablas que necesita la página no existen
   todavía. Si hace falta, coordiná con `entity-builder` en vez de crear tablas por tu cuenta
   con lógica distinta.
5. **Generar schema**: `pac model genpage generate-types` sobre las entidades de Dataverse
   involucradas.
6. **Generar código**: el componente `.tsx` completo. Para pedidos de varias páginas, generá
   cada una en paralelo, no secuencialmente.
7. **Deploy**: `pac model genpage upload` (o `push`, según versión de PAC CLI) hacia la app
   model-driven de destino — confirmá con el usuario cuál app antes de subir.
8. **Verificar** — confirmá que la página aparece en el sitemap de la app y abre
   correctamente. Playwright es opcional para verificación interactiva, no obligatorio.

## Gotcha técnico real — caché de módulo entre navegaciones

La plataforma genpage **re-evalúa el script del módulo en cada navegación**, incluso al
volver a una página ya visitada — las variables a nivel de módulo (`let _cache = null`) se
resetean, causando refetch y spinner de carga incluso en visitas de vuelta.

**Fix**: inicializar las variables de caché desde `window`, y escribir de vuelta a `window`
después de cada fetch — `window` persiste durante toda la sesión del browser,
independientemente de la re-evaluación del módulo:

```typescript
// Convención: window.__pp<NombreEntidad>Cache, para evitar colisiones con otros scripts
let _cache = (window as any).__ppContactCache ?? null;

async function fetchData() {
  const records = await props.dataApi.retrieveMultiple(...);
  _cache = records;
  (window as any).__ppContactCache = records; // escribir de vuelta
  return records;
}
```

También: usar **un solo objeto de estado batcheado** (`{ records, loading, error }`) en vez de
varios `setState` sueltos dentro de una función async — en React 17, cada `setState` por
separado dispara un render distinto, cada uno mostrando un estado intermedio visible al usuario.

## Cómo encaja en el modelo de canales de Axxon

No es ni DEV/Web API puro ni Pipeline — es su propio canal (PAC CLI directo al environment,
igual que la parte de Pipeline de `solution-packager` ya usa `pac` CLI). Una vez desplegada,
la página queda como componente de la solución — su promoción a TEST/PROD sigue el mismo
canal Pipeline que cualquier otro componente, vía `solution-packager`. No la despliegues vos
mismo a TEST/PROD con `pac model genpage upload` apuntando directo a esos environments.

## Restricciones

- Nunca asumas que la feature es GA — es Preview, confirmalo con el usuario en cada proyecto.
- Nunca fetch directo a la Web API desde el componente — siempre `props.dataApi`.
- Nunca imports de íconos fuera de `@fluentui/react-icons` verificados.
- Nunca despliegues directo a TEST/PROD — la promoción va por `solution-packager`, canal Pipeline.
- Si la página necesita crear entidades y no hay `az login` configurado, decilo explícitamente
  en vez de intentar workarounds — sin `az`, `genpage-builder` solo funciona sobre tablas
  existentes o datos mock.
