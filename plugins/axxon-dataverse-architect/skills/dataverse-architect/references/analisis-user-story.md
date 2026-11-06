# Análisis de alcance al recibir una User Story

Leer **antes de rutear** cuando el trabajo arranca desde una Historia de Usuario real (la que
armó `d365-requirements-analyst` en Fase 1, con Descripción, Tareas, Criterios de Aceptación
en Gherkin y Consideraciones D365 — ya sea porque el usuario la pega, o porque se leyó de
Azure DevOps). **No rutees directo a la primera skill obvia.** Primero analizá el alcance:

1. **Leé toda la Historia**, no solo el título — el patrón que se busca evitar es rutear a
   `entity-builder` porque la Descripción menciona una tabla, sin notar que las
   Consideraciones D365 ya adelantan que hace falta una integración externa.
2. **Detectá cadenas de dependencia entre skills**, no solo una skill aislada. La más común y
   la que más se pasa por alto: **Custom API → Plugin → Azure Function**. Preguntas que
   tenés que hacerte explícitamente:
   - ¿La Historia necesita exponer una operación custom (no CRUD estándar)? → `custom-api-builder`.
   - ¿Esa operación (o cualquier lógica mencionada) necesita ejecutar código server-side
     transaccional? → `plugin-builder`.
   - ¿Ese plugin necesita hablar con un sistema externo (una API de terceros, otro servicio)?
     → **no metas esa llamada HTTP dentro del plugin sync** (ver restricción explícita en
     `plugin-builder`) — necesitás `azure-function-builder` para la pieza de integración, y el
     plugin la llama de forma async.
3. **Decidí low-code vs. pro-code con `build-approach-advisor`** y mostrale al usuario el
   **Plan de construcción** completo (Puerta 1) antes de ejecutar el primer paso — cuántas
   piezas, con qué enfoque, en qué orden, y el criterio que justifica cada pieza pro-code (ej.
   "Custom API + Plugin + Azure Function, porque la validación de crédito tiene que consultar
   un sistema externo sin connector"). Nunca arranques a construir la primera pieza sin el
   plan aprobado explícitamente — el usuario tiene que poder frenarte si el alcance o el
   enfoque que inferiste está mal.
4. Si después de leer la Historia completa **no está claro** si hace falta esta cadena (ej. no
   queda claro si la integración es sync u async, o si el sistema externo ya tiene un
   connector estándar que resolvería esto con `flow-builder` en vez de código custom),
   preguntá antes de asumir — no default a la opción más compleja (Plugin + Azure Function)
   si un Business Rule o un flow alcanzan.
5. Si el plan implica una **decisión de arquitectura con alternativas reales** (criterios en
   `adr-recorder`, sección "Qué amerita un ADR"), decilo al mostrar el plan y ofrecé
   registrarla como ADR — antes de construir, así el ADR documenta la decisión y no se
   reconstruye después.

## Pedidos que mezclan varias operaciones

Si el pedido mezcla varias operaciones ("creá la tabla, el form, y bloqueá el campo monto si
el estado es Aprobada"), descomponelo y secuencialo en orden de dependencia — tabla →
columnas → vista/form → business rule → seguridad — mostrando el plan antes de ejecutar el
primer paso.
