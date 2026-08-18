# ADR-004 — Adopción de MCP Apps para vistas previas interactivas

**Estado:** Aceptado (MVP acotado, stack resuelto)
**Fecha:** 2026-08-18 (actualizado el mismo día — stack de widgets resuelto)
**Contexto previo:** ADR-001 (canales de ejecución), ADR-002 (topología del MCP Server)

---

## Contexto

Al comparar nuestro modelo de ejecución (MCP Server propio + CLI/npx directo para
`genpage-builder`/`code-app-builder`) contra el repo `microsoft/power-platform-skills`,
encontramos el plugin `mcp-apps` — **no es una alternativa de ejecución**, es el protocolo
`@modelcontextprotocol/ext-apps` (spec SEP-1865): permite que un tool result de MCP incluya
`_meta.ui.resourceUri` apuntando a un recurso HTML que el host renderiza como **widget
interactivo** en vez de solo texto/JSON plano.

Hoy, cuando `form-designer`/`view-designer`/`app-composer` terminan una operación, el
consultor tiene que abrir el browser manualmente para verificar cómo quedó el form/vista/
sitemap — un paso de verificación 100% manual en cada iteración.

## Decisión

Adoptar MCP Apps **progresivamente**, empezando por un MVP de **3 widgets** (no las 13 skills
de una — se prioriza por dónde el texto plano es menos representativo del resultado real):

| Prioridad | Skill | Widget | Por qué primero |
|---|---|---|---|
| 1 | `form-designer` | Preview del layout del form (tabs, secciones, campos) | JSON/XML de un form es lo menos legible de leer "a ojo" de las 13 skills |
| 2 | `view-designer` | Preview de la grilla (columnas, orden, filtros aplicados) | Mismo problema — FetchXML/LayoutXML no se "ve" sin renderizar |
| 3 | `app-composer` | Árbol de navegación del Sitemap (Area > Group > SubArea) | Estructura jerárquica, ideal para un árbol visual en vez de XML |

**Fuera del MVP, evaluar después**: `business-rule-engine` (la lógica condición→acción se
presta a un flowchart, pero es la 4ta prioridad, no la 1ra) y `entity-builder`/
`security-architect` (más estructurales, el texto ya es razonablemente legible ahí).

## Diseño técnico

1. **El Azure Function (MCP Server) sirve los recursos de widget** — nuevos endpoints
   estáticos (`ui://widget/form-preview.html`, etc.), siguiendo el patrón de
   `registerWidget()` documentado en la spec — HTML+CSS+JS autocontenido por widget, con
   `resourceDomains` para CSP explícito (mismo criterio de seguridad que ya aplicamos en
   otros lados: nada de dominios sin allowlistar).
2. **Cada tool relevante agrega `_meta.ui.resourceUri`** en su response, apuntando al widget
   correspondiente, sin cambiar el payload de datos que ya devuelve — el widget consume esos
   mismos datos (FormXml, FetchXml/LayoutXml, sitemapxml), no pide nada nuevo al servidor.
3. **Fallback obligatorio**: si el host de Cowork no soporta MCP Apps (spec nueva, no todos
   los hosts la implementan todavía), el tool tiene que seguir devolviendo el texto/JSON
   plano de siempre — MCP Apps es una mejora progresiva, nunca un reemplazo que rompa el
   comportamiento actual si el host no lo soporta.

## Consecuencias

- **Inversión de ingeniería real** — no es gratis: nuevos endpoints en el Azure Function,
  desarrollo de 3 widgets, testing de que el fallback funciona cuando MCP Apps no está
  disponible. No es un cambio de una tarde.
- **No reemplaza la verificación humana** — sigue siendo una vista previa aproximada, no un
  reemplazo del gate de aprobación humana antes de Pipeline (ADR-001 no cambia).
- Abre la puerta a expandir a más skills después de validar el MVP con las 3 primeras.

## Decisión de stack — resuelta: HTML/CSS/JS vanilla, sin framework

Se descartó React a favor de HTML/CSS/JS plano, autocontenido por widget, sin build pipeline.
Razones:

1. **El Azure Function (MCP Server) no tiene pipeline de frontend hoy** — ni bundler, ni
   transpilación. Sumar React significa sumar y mantener esa cadena completa solo para 3
   widgets — costo de infraestructura desproporcionado al MVP.
2. **El contenido es estructural, no interactivo-complejo**: un layout de form, una grilla, y
   un árbol de sitemap son vistas que se resuelven bien con HTML/CSS/SVG directo — no
   necesitan el modelo de componentes de React para esto.
3. **La consistencia con `genpage-builder`/`code-app-builder` no aplica acá** — esas skills
   usan React porque generan **páginas persistentes de la app real**, un artefacto
   completamente distinto a una vista previa efímera dentro de un tool result de chat. Mismo
   ecosistema, casos de uso distintos — no hace falta que compartan stack.

**Si el alcance crece significativamente más allá del MVP** (muchos widgets, interactividad
real más allá de una preview de solo lectura), revisar esta decisión — no es una regla para
siempre, es la correcta para el alcance actual de 3 widgets.

**Nota de estilo**: estos widgets son herramienta interna de verificación para el consultor,
no un entregable de cara al cliente — no necesitan la paleta de marca Axxon 2025 que sí usan
`kickoff-brief`/`fdd-builder`. Priorizar legibilidad funcional sobre estética de marca.

## Pendiente de implementación (no bloqueante, orden sugerido)

1. Prototipo del widget de `form-designer` (HTML/CSS/JS vanilla) — el de mayor valor esperado.
2. Validar fallback real: probar el mismo tool call desde un host que no soporte MCP Apps.
3. `view-designer` y `app-composer` una vez validado el patrón con el primero.
