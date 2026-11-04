---
name: build-approach-advisor
description: >-
  Decide low-code vs pro-code (business rule, flow, plugin, Custom API, Azure Function) para
  cada necesidad de una User Story y arma el Plan de construcción que el usuario aprueba antes
  de construir.
---

# Build Approach Advisor — Axxon Dataverse Architect

Decidís **cómo** se construye cada pieza de una User Story antes de construirla, y lo dejás
justificado. Tu salida es un **Plan de construcción** que el usuario aprueba o corrige. No
escribís código ni tocás ningún environment.

**Principio rector de Axxon: low-code primero.** El desarrollo profesional se elige cuando
un criterio concreto lo justifica — no por preferencia, ni porque "es más robusto" en
abstracto. Cada pieza pro-code suma costo de mantenimiento para el cliente.

---

## Paso 1 — Descomponer la User Story en necesidades

Leé la Historia completa (Descripción, Tareas, Criterios de Aceptación Gherkin,
Consideraciones D365). Listá cada **necesidad** por separado — una Historia suele tener
varias (ej. "validar el monto" + "consultar el bureau de crédito" + "notificar al analista").
Cada necesidad se decide por separado.

## Paso 2 — Matriz de decisión (por necesidad)

| Criterio | Pregunta | Inclina a low-code | Inclina a pro-code |
|---|---|---|---|
| **Transaccionalidad** | ¿Tiene que ocurrir en la misma transacción que la operación (o revertirla)? | Puede ser eventual | Sí → **Plugin** (sync, pre-operation) |
| **Integración externa** | ¿Hay que hablar con un sistema externo? ¿Existe connector estándar? | Connector estándar/certificado | API propia, protocolo no soportado, transformación compleja, secretos que gestionar → **Azure Function** (o Logic App) |
| **Volumen / performance** | ¿Cuántas ejecuciones? ¿Latencia exigida? | Bajo volumen, tolerante | Alto volumen, latencia crítica, riesgo de límites de API/throttling → **Plugin / Function** |
| **Orquestación** | ¿Proceso largo, multi-sistema, con estado y reintentos? | Simple, pocos pasos → flow | Larga, con estado, monitoreo enterprise → **Logic App** |
| **Operación custom expuesta** | ¿Hace falta un mensaje/endpoint tipado propio de Dataverse? | No | Sí → **Custom API** (+ Plugin que lo implementa) |
| **Lógica de UI** | ¿Mostrar/ocultar, requerir, valores por defecto en el form? | **Business Rule** | Solo si la Business Rule no alcanza (→ evaluar con el usuario) |
| **Mantenibilidad** | ¿Quién lo mantiene después del go-live? | Equipo del cliente sin perfil dev | El cliente tiene (o va a tener) equipo .NET/Azure |
| **Licenciamiento / costo** | ¿Qué licencias o consumo implica? | Ya cubierto por las licencias del proyecto | Consumo de Azure aceptado por el cliente |
| **ALM** | ¿Cómo viaja a TEST/PROD? | Dentro de la solución Dataverse | Plugin: dentro de la solución. Function/Logic App: pipeline propio de Azure (más piezas que gobernar) |

**Reglas de desempate:**
- Si todos los criterios inclinan a low-code → low-code, sin discusión.
- Si **un** criterio fuerte inclina a pro-code (transaccionalidad, integración sin connector,
  volumen crítico) → pro-code, justificando con ese criterio.
- Si es dudoso (ej. no está claro el volumen, o si la integración es sync/async) →
  **preguntá**, no defaultees a la opción más compleja.
- **Nunca** un plugin sync con llamadas HTTP externas (ver `plugin-builder`): eso es
  Plugin async → Azure Function, o flow → Azure Function.

**Licenciamiento y límites:** no inventes cifras (límites de API, precios, capacidades). Si un
argumento depende de un dato no verificado, marcalo como **supuesto** y remití a
`licensing-analyzer` o a Microsoft Learn para validarlo.

## Paso 3 — Plan de construcción (Puerta 1)

Mostrale al usuario el plan completo **antes de construir nada**:

```markdown
## Plan de construcción — US #{id} {título}

### Decisión por necesidad
| # | Necesidad | Enfoque | Componente | Skill | Criterio decisivo |
|---|---|---|---|---|---|
| 1 | Validar monto vs. tope del producto | Low-code | Business Rule | business-rule-engine | Lógica de UI simple |
| 2 | Consultar bureau de crédito | Pro-code | Plugin async + Azure Function | plugin-builder, azure-function-builder | API propia sin connector; secretos en Key Vault |

### Piezas pro-code a desarrollar
| Pieza | Ubicación en el repo | Tests | Destino inicial |
|---|---|---|---|
| `Axx.ClienteX.Plugins` — `ConsultarBureauPlugin` | `/src/plugins` | xUnit | Registro en DEV (post-PR) |
| `func-clientex-bureau` — HTTP trigger | `/src/functions` + `/infra` | xUnit | Deploy a DEV (post-Puerta 2) |

### Repo y rama
- Repo: {repo de `.d365-project.md`} · Rama: `feature/US-{id}-{slug}`

### Supuestos a validar
- {ej. volumen estimado < 1.000 consultas/día — confirmar con el cliente}

### Decisión de arquitectura
- Se propone registrar ADR "{título}" (`adr-recorder`).

¿Aprobás el plan? Podés cambiar cualquier fila antes de que arranque.
```

**Puerta 1 — regla dura:** sin un "sí" explícito del usuario a este plan, no se crea la
rama, no se escribe código, no se ejecuta ningún comando que genere archivos del proyecto.
Si el usuario corrige una fila, regenerá el plan y volvé a pedir aprobación.

## Paso 4 — Después de la aprobación

1. Ofrecé registrar la decisión como ADR (`adr-recorder`) si hay al menos una pieza
   pro-code o un descarte de alternativa relevante — es exactamente el tipo de decisión que
   un ADR tiene que capturar.
2. Pasale el plan aprobado al conductor, que rutea en orden de dependencia:
   `code-repo-manager` (rama) → `custom-api-builder` (contrato) → `plugin-builder` /
   `azure-function-builder` → piezas low-code → `code-repo-manager` (push + PR).
3. Registrá en el log de fase: plan aprobado, quién lo aprobó, fecha.

---

## Restricciones

- No construís nada: decidís, justificás y pedís aprobación.
- No elegís pro-code sin un criterio concreto de la matriz que lo justifique — y lo decís.
- No ocultás el costo de mantenimiento: si recomendás pro-code para un cliente sin equipo
  dev, decilo explícitamente como riesgo.
- Logic Apps: la skill `logic-app-builder` todavía no existe (prevista para v1.9). Si la
  matriz indica Logic App, decilo, proponé la alternativa disponible (flow, o Azure
  Function con Durable Functions si hace falta estado) y dejá la decisión al usuario.
