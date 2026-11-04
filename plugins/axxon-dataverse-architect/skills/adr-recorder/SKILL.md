---
name: adr-recorder
description: >
  Registra Architecture Decision Records (ADRs) del proyecto del cliente en la wiki de proyecto
  de Azure DevOps, de forma estructurada: numeración correlativa, página por ADR, índice
  actualizado, ciclo de vida (Propuesto → Aceptado → Reemplazado/Deprecado) y link del ADR a la
  User Story que lo originó. Usar cuando el conductor `dataverse-architect` detecta una decisión
  de arquitectura significativa durante la construcción (elegir entre Business Rule / flow /
  plugin / Custom API, cadena Custom API → Plugin → Azure Function, integración sync vs async,
  GAP resuelto con customización, modelo de seguridad, decisión de modelo de datos, excepción a
  una regla de ADR-001), o cuando el usuario pide "registrá esta decisión", "armá un ADR",
  "documentá por qué elegimos X", "aceptá / reemplazá el ADR-00N", o "mostrame los ADRs del
  proyecto". Registra decisiones del proyecto del cliente — no los ADRs de producto de los repos
  axxon-* (esos viven en `docs/` de cada repo).
---

# ADR Recorder — Axxon Dataverse Architect

Registrás las **decisiones de arquitectura de la solución del cliente** en la **wiki de
proyecto** de su Azure DevOps, para que quede trazable *por qué* la solución está construida
como está — no solo *qué* se construyó (eso ya lo cubren el repo y el SDD).

Usás el **servidor MCP oficial de Azure DevOps** (`microsoft/azure-devops-mcp`), tools `wiki`,
`wiki_upsert_page` y `wit_work_item_link_write`. No usás `ado_client.py`.

---

## Qué amerita un ADR (y qué no)

**Sí** — decisiones con alternativas reales y consecuencias que alguien va a querer entender
dentro de 6 meses:

| Disparador | Ejemplo |
|---|---|
| Elegir mecanismo de lógica | Business Rule vs. flow vs. plugin vs. Custom API para una validación |
| Cadena de integración | Custom API → Plugin → Azure Function async, en vez de llamada HTTP dentro del plugin |
| Sync vs. async | Validación de crédito en línea vs. diferida |
| GAP resuelto con customización | Se construye custom algo que no cubre el estándar de D365 |
| Modelo de seguridad | Segregación por BU vs. teams vs. field security |
| Modelo de datos | N:N nativa vs. tabla intermedia, alternate keys, ownership User vs. Organization |
| Excepción a una regla de Axxon | Cualquier desvío de ADR-001 (canales) o de las convenciones del proyecto |

**No** — acciones de construcción sin alternativa en juego: crear una columna, agregar un campo
a un form, una vista más. Eso queda en el log de fase y en el SDD, no en un ADR.

Ante la duda, **preguntá** ("¿Querés que registre esto como ADR?") — no generes ADRs por
cada acción: un registro inflado deja de leerse.

---

## Antes de arrancar — configuración de la wiki

1. Leé la sección `## Wiki de ADRs` de `.d365-project.md` (si el proyecto se trabaja sin el
   supraagente y solo existe `.d365-session.md`, buscala ahí):

   ```markdown
   ## Wiki de ADRs
   - Wiki: {nombre o id de la wiki de proyecto, ej. ClienteX.wiki}
   - Ruta raíz: /Arquitectura/ADRs
   ```

   El Project y la organización de ADO salen de la sección `## Azure DevOps` del mismo archivo.

2. Si la sección falta o está incompleta: **pará** y pedí los datos. No asumas el nombre de la
   wiki. Podés ayudar listando las wikis existentes con `wiki` (action `list_wikis`, pasando
   `project`) y que el usuario elija.

3. Si el Project **no tiene wiki de proyecto**: el MCP no tiene una acción para crearla.
   Pedile al usuario que la cree una vez desde ADO (*Overview → Wiki → Create project wiki*)
   y seguí cuando confirme.

4. Verificá que el MCP de Azure DevOps responda (una llamada `wiki` action `get_wiki`). Si no
   responde → modo offline (ver al final).

---

## Estructura en la wiki

```
{Ruta raíz, ej. /Arquitectura/ADRs}              ← página índice
├── ADR-0001-modelo-seguridad-por-bu
├── ADR-0002-validacion-credito-plugin-async
└── ADR-0003-...
```

- **Nombre de página:** `ADR-{NNNN}-{slug}` — número con 4 dígitos, slug en minúsculas, solo
  letras, números y guiones (sin `#`, `?`, `/`, `\`, `:` ni espacios).
- Si la página raíz o sus padres (`/Arquitectura`) no existen, creálos primero (comprobando
  con `get_page`) con un contenido breve — no asumas que la API crea los padres sola.

### Plantilla de la página índice

```markdown
# Architecture Decision Records — {Nombre del proyecto}

Registro de decisiones de arquitectura de la solución. Cada ADR es inmutable una vez
aceptado: si la decisión cambia, se registra un ADR nuevo que reemplaza al anterior.

| # | Título | Estado | Fecha | User Story | Reemplaza / Reemplazado por |
|---|---|---|---|---|---|
| [ADR-0001](./ADR-0001-modelo-seguridad-por-bu) | Modelo de seguridad por BU | Aceptado | 2026-11-04 | #1234 | — |
```

### Plantilla de cada ADR

Mismo formato que los ADRs de Axxon, con los campos propios del proyecto:

```markdown
# ADR-{NNNN} — {Título}

| | |
|---|---|
| **Estado** | Propuesto · Aceptado · Reemplazado por ADR-{NNNN} · Deprecado |
| **Fecha** | {YYYY-MM-DD} |
| **Decisores** | {nombres / roles} |
| **Aceptado por** | {nombre} — {fecha} *(vacío mientras esté Propuesto)* |
| **Fase SbD** | Implement |
| **User Story / Épica** | #{id} {título} |
| **Componentes afectados** | {tablas, plugins, flows, roles…} |
| **Skills involucradas** | {ej. custom-api-builder, plugin-builder, azure-function-builder} |
| **Reemplaza** | ADR-{NNNN} *(si aplica)* |

## Contexto
{Problema, restricciones (licenciamiento, volumen, latencia, seguridad, FSI), y qué fuerza
la decisión. Hechos verificados separados de supuestos — marcá los supuestos.}

## Decisión
{Qué se decidió, en una o dos frases, y cómo se implementa.}

## Alternativas consideradas
| Alternativa | Por qué se descartó |
|---|---|
| {ej. flow con connector HTTP} | {ej. no garantiza transaccionalidad con el create} |

## Consecuencias
- **Positivas:** …
- **Negativas / costos:** …
- **Impacto en ALM / despliegue:** …

## Pendiente
- {lo que queda abierto, si aplica}
```

---

## Flujo — registrar un ADR nuevo

1. **Redactá el borrador** con la plantilla, a partir de la conversación y de la User Story.
   No inventes alternativas ni consecuencias que no se discutieron: si falta algo, preguntá o
   dejalo explícitamente como *"a completar"*.
2. **Mostrá el borrador en el chat y pedí confirmación** (patrón de confirmación del conductor):

   ```
   ⚠️ Estás por publicar:
   **Acción:** Crear ADR "Validación de crédito vía plugin async + Azure Function" (estado: Propuesto)
   **Wiki:** ClienteX.wiki — /Arquitectura/ADRs
   **User Story:** #1234
   ¿Confirmás?
   ```

3. **Calculá el número justo antes de escribir** (no al redactar): leé el índice con `wiki`
   action `get_page_content` y tomá el mayor `ADR-NNNN` + 1. Si el índice no existe, el
   número es `0001`.
4. **Verificá que la página no exista** con `wiki` action `get_page` (`wikiIdentifier`,
   `project`, `path`). ⚠️ `wiki_upsert_page` **sobrescribe en silencio** si la página ya
   existe — si `get_page` la encuentra, otro consultor tomó ese número en paralelo: volvé al
   paso 3.
5. **Creá la página** con `wiki_upsert_page` (`wikiIdentifier`, `project`, `path`, `content`).
   Pasá **siempre** `project` explícito — si se omite, la URL se arma con el project vacío.
6. **Actualizá el índice**: releé su contenido, agregá la fila, y escribilo con
   `wiki_upsert_page`. Si la escritura falla por conflicto de versión (otro escribió el índice
   entre tu lectura y tu escritura), releé y reintentá — nunca pises el índice con una versión
   vieja.
7. **Enlazá el ADR a la User Story** con `wit_work_item_link_write` action `add_artifact_link`,
   `linkType: "Wiki"`, `workItemId`, `wikiId`, `pagePath` (y `pageId` si lo obtuviste de
   `get_page`). Si el link falla, no es bloqueante: agregá un comentario en la User Story con
   `wit_work_item_comment_write` con el link a la página y reportalo.
8. **Registrá la acción** en el log de la fase (`.d365-fases/fase-2-implement.md`, o el
   historial de `.d365-session.md`):

   | Fecha | Subagente | Skill | Canal | Acción | Resultado |
   |---|---|---|---|---|---|
   | 2026-11-04 | dataverse-builder | adr-recorder | ADO Wiki (MCP) | ADR-0002 creado (Propuesto) | {link a la página} |

9. Devolvé al usuario el link a la página y al índice.

---

## Flujo — ciclo de vida

| Transición | Quién la dispara | Qué hacés |
|---|---|---|
| Propuesto → **Aceptado** | **Solo** el usuario, explícitamente en el chat ("aceptá el ADR-0002") | Actualizás `Estado` y `Aceptado por` (nombre + fecha que te indique el usuario) en la página y en el índice |
| Aceptado → **Reemplazado** | Se registra un ADR nuevo que lo reemplaza | Creás el nuevo (flujo completo, con `Reemplaza: ADR-000N`), y en el viejo cambiás **solo** `Estado` a `Reemplazado por ADR-000M` — el resto del contenido no se toca |
| Cualquiera → **Deprecado** | El usuario, explícitamente | Cambiás solo el `Estado` y agregás una línea en `Pendiente` con el motivo |

**Reglas duras del ciclo de vida:**

- **Nunca** marcás un ADR como Aceptado por iniciativa propia, ni porque "se construyó así".
  La wiki de proyecto no tiene un gate de aprobación (a diferencia de un PR) — la aceptación
  explícita del usuario en el chat *es* el gate. Si el usuario no dice quién acepta, preguntá.
- **Nunca** editás Contexto / Decisión / Alternativas / Consecuencias de un ADR **Aceptado**:
  si la decisión cambió, es un ADR nuevo. Un ADR **Propuesto** sí se puede corregir.
- **Nunca** borrás un ADR ni una fila del índice.
- Para editar una página existente, leé primero su contenido actual y modificá sobre eso —
  nunca regeneres la página desde cero.

---

## Consultar ADRs

"¿Qué ADRs hay?" / "¿Por qué usamos X?" → leé el índice (`get_page_content`) y, si hace falta,
la página puntual. Citá el número y el link. Antes de proponer una decisión nueva, chequeá el
índice: si ya existe un ADR aceptado sobre el mismo tema, mencionalo — la propuesta nueva
probablemente es un reemplazo, no un ADR independiente.

---

## Modo offline (MCP de Azure DevOps no disponible)

Si el MCP no responde, no frenes el trabajo de construcción:

1. Guardá el borrador en `.d365-adr/pendiente-{slug}.md` dentro del Cowork Project, **sin
   número** (el número se asigna al publicar, para no chocar con otros consultores).
2. Avisale al usuario que quedó pendiente de publicar y por qué.
3. En la próxima sesión con el MCP disponible, al primer uso de esta skill: listá
   `.d365-adr/pendiente-*.md`, ofrecé publicarlos (flujo completo, con confirmación), y
   borrá cada archivo local solo después de verificar que su página existe en la wiki.

---

## Manejo de errores

| Error | Causa probable | Qué hacer |
|---|---|---|
| `wiki` no encuentra la wiki | Nombre/id incorrecto en `.d365-project.md`, o no hay wiki de proyecto | `list_wikis` y pedirle al usuario que confirme / la cree |
| `get_page` encuentra la página del número calculado | Otro consultor creó un ADR en paralelo | Recalcular el número (paso 3) |
| Conflicto al escribir el índice | El índice cambió entre lectura y escritura | Releer y reintentar, sin pisar |
| Error de `project` vacío en la URL | Se omitió `project` en `wiki_upsert_page` | Pasarlo siempre explícito |
| `add_artifact_link` falla | Formato del link de wiki no aceptado, o permisos | Comentario en la User Story con el link y reportarlo |
| MCP sin responder | Servidor no conectado en la sesión | Modo offline |

---

## Restricciones

- Registrás decisiones del **proyecto del cliente** en **su** Azure DevOps — nunca en el ADO
  de Axxon ni en los repos de producto `axxon-*`.
- **Nunca** incluís en un ADR credenciales, secrets, connection strings, tokens, datos
  productivos ni datos personales — solo nombres de componentes, decisiones y razones. Si el
  contexto los trae, abstraelos ("el endpoint del bureau de crédito", no la URL con key).
- **No inventás** comportamientos de producto de Microsoft, límites ni cifras para justificar
  una decisión: si el argumento depende de un dato no verificado, marcalo como supuesto e
  indicá dónde verificarlo.
- El ADR documenta la decisión — **no** la ejecuta. La construcción sigue por las skills
  correspondientes y sus canales (ADR-001).
- Es contenido que lee el cliente: recordale al usuario, la primera vez en la conversación,
  que conviene revisión humana antes de marcarlo Aceptado.
