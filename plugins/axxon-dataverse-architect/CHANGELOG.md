# Changelog — Axxon Dataverse Architect

## v1.0.0-rc1 (2026-08-05)

Primer release candidate del paquete migrado a Claude Cowork. Consolida el trabajo de
revisión arquitectónica documentado en `docs/ADR-001` a `docs/ADR-003`.

### Cerrado en este RC
- Migración completa de las 9 skills al formato real de Claude Skills (carpeta + `SKILL.md`,
  frontmatter `name`+`description`).
- Pivot de plataforma: Copilot Studio → Claude Cowork (ver ADR-003).
- Modelo de ejecución por canal: DEV/Web API, Git, Pipeline (ver ADR-001).
- Topología de MCP Server: una instancia por cliente (ver ADR-002).
- Prefijo de publisher unificado a `axx_` en todo el paquete.
- Skills nuevas: `view-designer`, `duplicate-detection`.
- Retirada la tabla `axx_designsession` — el contexto conversacional vive en Cowork Projects
  (`.d365-session.md`).
- Descartada la dependencia de la skill `output-formatter` (Copilot Studio / Adaptive Cards).

### Conocido / pendiente (no bloqueante para RC1)
- Verificar contra `$metadata` de un environment real los códigos de `ComponentType` para
  `Security Role`, `Alternate Key` y `Duplicate Detection Rule` (ver `docs/ADR-002`, nota de
  `04-solution-packager`/`solution-packager`).
- Diseñar el pipeline "meta" que propague actualizaciones de la plantilla del MCP Server a
  todas las instancias de clientes activos (ver `docs/ADR-002`).
- Confirmar restricciones de red de Cowork para sesiones en la nube contra el MCP Server (ver
  `docs/ADR-003`).

---

## Historial previo (pre-repo)

Registrado acá por trazabilidad; no corresponde a commits reales de este repositorio.

- **v2.2.0** — Topología del MCP Server (ADR-002): una instancia por cliente, Azure DevOps
  Project dedicado por cliente.
- **v2.1.0** — Correcciones de una validación end-to-end (canal de Environment Variables,
  confirm-before-execute para skills nuevas, advertencia sobre ComponentType sin verificar).
- **v2.0.0** — Pivot de ejecución (ADR-001): de Web API contra cualquier environment a modelo
  híbrido DEV/Git/Pipeline. Prefijo unificado a `axx_`. Skills nuevas: `view-designer`,
  `duplicate-detection`.
- **v1.0.0** — Paquete original sobre Copilot Studio + Azure Function MCP Server con Web API
  directa contra todos los environments, incluido PROD.
