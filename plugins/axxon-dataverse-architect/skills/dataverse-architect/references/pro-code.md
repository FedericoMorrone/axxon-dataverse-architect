# Desarrollo profesional (pro-code) — las tres puertas

Leer cuando el plan incluye alguna pieza pro-code (Plugin, Custom API con implementación,
Azure Function). Cada pieza pasa por tres validaciones humanas, sin excepción y aunque el
usuario tenga apuro:

| Puerta | Cuándo | Quién valida | Qué se muestra |
|---|---|---|---|
| **1 — Plan** | Antes de crear la rama o escribir una línea | El usuario, en el chat | Plan de construcción de `build-approach-advisor`: piezas, enfoque, criterio, repo/rama, destino |
| **2 — Resultado** | Antes de `git push` y de registrar/desplegar algo en DEV | El usuario, en el chat | Archivos, build y tests (verde/rojo), registration manifest, qué se va a desplegar |
| **3 — PR** | Antes de mergear | Una persona del equipo, en Azure DevOps | El PR enlazado a la User Story — el agente nunca aprueba ni mergea |

## Plan ya aprobado en ADO (trabajo entre roles)

Si el pedido llega desde `ado-dev-queue` (un dev tomó una Task `[PC]`), la Puerta 1 ya está
cumplida en Azure DevOps: la US tiene el tag `plan-aprobado` y los comentarios
`[axx:plan-construccion v1]` / `[axx:aprobacion-procode v1]` (contrato ADO,
`axxon-dataverse-agent/docs/ado-contract.md`). En ese caso **no** vuelvas a correr
`build-approach-advisor` ni a pedir la aprobación del plan completo: mostrale al dev solo
**su pieza** según el `[axx:contrato-tecnico v1]` de la Task y esperá su OK antes de crear la
rama. Si el contrato técnico falta o contradice el plan, frená y pedí que se corrija en ADO —
no lo resuelvas por tu cuenta. Al terminar, el resultado se registra en la Task
(`[axx:resultado v1]`, vía `ado-dev-queue`).

## Orden de ruteo después de la Puerta 1

`code-repo-manager` (rama) → `custom-api-builder` (contrato, si aplica) → `plugin-builder` /
`azure-function-builder` → piezas low-code → `code-repo-manager` (Puerta 2, push, PR).
TEST/PROD siguen exclusivamente por pipeline con gate (ADR-001, sin cambios).
