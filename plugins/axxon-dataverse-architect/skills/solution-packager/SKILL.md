---
name: solution-packager
description: >
  Gestiona el ciclo de vida ALM de soluciones en Dataverse / Dynamics 365 CE: crea publishers
  y soluciones vía el SDK de Python (canal DEV), agrega componentes vía PAC CLI, versiona, y
  promueve DEV→TEST→PROD exclusivamente vía pipeline de Azure DevOps con gates de aprobación
  — nunca importa directo a TEST/PROD. Usar esta skill cuando el usuario pida crear una
  solución o publisher, agregar un componente a la solución, versionar la solución, promover
  al ambiente de TEST o PROD, listar soluciones o componentes, consultar el estado de un
  pipeline o deploy, generar un script de pac CLI, o hable de ALM en general. Usar junto con
  la skill dataverse-architect.
---

# Solution Packager — Axxon Dataverse Architect

Skill especializada en la gestión del ciclo de vida de **soluciones** en Dataverse / D365 CE.
Opera en **dos canales completamente separados**, con orígenes distintos:

1. **DEV — SDK de Python + PAC CLI directo.** Adaptado del enfoque real de
   `microsoft/Dataverse-skills` (skill `dv-solution`) — feedback inmediato, sin pasar por
   nuestro propio MCP Server para esta parte.
2. **Pipeline — sin cambios, gate propio de Axxon.** Todo lo que promueve la solución más
   allá de DEV sigue exclusivamente el circuito de PR + pipeline con aprobación humana. **Este
   canal es intocable** — `dv-solution` en su diseño original importa directo a cualquier
   ambiente sin gate; eso **no se adoptó**, es la garantía central de este paquete.

---

## Antes de cualquier operación — confirmar el environment

`pac auth list` + `pac org who`, mostrarle el resultado al usuario y confirmar que coincide
con el ambiente que se pretende tocar — nunca asumir. Los consultores trabajan contra varios
environments en paralelo; confundir uno por otro en esta skill es el error más caro posible.

---

## Canal DEV — publisher, solución, componentes

### Encontrar o crear el publisher

**Siempre usar el SDK de Python (`client.records.create`/`client.records.list`), no HTTP
crudo** — evita los bugs de URL encoding y parsing de GUID que arrastraba la versión anterior
de esta skill.

```python
import os, sys
sys.path.insert(0, os.path.join(os.getcwd(), "scripts"))
from auth import get_client

client = get_client("solution-packager")

# 1. Buscar publishers no-Microsoft existentes
publishers = client.records.list(
    "publisher",
    filter="customizationprefix ne 'none' and uniquename ne 'MicrosoftCorporation' and uniquename ne 'Microsoftdynamic'",
    select=["publisherid", "uniquename", "friendlyname", "customizationprefix"],
    top=10,
)

if publishers:
    # Mostrar los publishers existentes y preguntarle al usuario cuál usar — nunca asumir
    ...
else:
    # No hay publisher custom — preguntarle al usuario el prefijo (nunca "new")
    publisher_id = client.records.create("publisher", {
        "uniquename": "axx",
        "friendlyname": "Axxon Consulting",
        "customizationprefix": "axx",
        "description": "Publisher de Axxon Consulting para implementaciones D365 CE",
    })
```

**Reglas — sin excepción:**
- **Nunca usar el prefijo `new`** — no aporta identidad organizacional y arriesga colisiones
  de nombres. En Axxon, siempre `axx`.
- El prefijo queda **efectivamente permanente** — los componentes creados lo conservan para
  siempre, aunque después se cambie el publisher.
- **Siempre preguntarle al usuario** antes de crear un publisher nuevo — nunca hardcodear el
  prefijo ni asumirlo de un ejemplo de esta documentación.
- Un publisher puede tener muchas soluciones — reusar el existente cuando sea posible.

### Crear la solución

```python
solution_id = client.records.create("solution", {
    "uniquename": "ClienteXCreditOnboarding",
    "friendlyname": "Cliente X — Credit Onboarding",
    "version": "1.0.0.0",
    "publisherid@odata.bind": f"/publishers({publisher_id})",
})
```

> **No existe un comando `pac solution create`.** PAC CLI maneja export/import/pack/unpack,
> no la creación del registro de la solución en sí — eso siempre va por el SDK o Web API.

### Convención de nombres y versionado (sin cambios)

```
{ClienteAbreviado}{Proyecto}{Módulo}   →  ClienteXCreditOnboarding, BGaliciaCRMSales

Major.Minor.Build.Revision
1.0.0.1   → Primera build de desarrollo
1.0.1.0   → Release a TEST (vía pipeline)
1.1.0.0   → Nueva funcionalidad menor
2.0.0.0   → Cambio breaking
```

### Agregar componentes — vía PAC CLI

```bash
pac solution add-solution-component \
  --solutionUniqueName <UniqueName> \
  --component <ComponentSchemaName> \
  --componentType <TypeCode> \
  --environment <url>
```

> **Gotcha real:** PAC CLI usa acá argumentos camelCase (`--solutionUniqueName`,
> `--componentType`) — no kebab-case, a diferencia de otros comandos de PAC CLI. Repetir el
> comando una vez por componente.

### Alternativa — header `MSCRM.SolutionName` al crear metadata vía Web API

```python
headers = {
    "Authorization": f"Bearer {token}",
    "Content-Type": "application/json",
    "MSCRM.SolutionName": "<UniqueName>",
}
```

> ⚠️ **Falla en silencio si está mal escrito.** Si el nombre de la solución tiene un typo o
> no existe, los componentes se crean en la **solución default** sin ningún error — nunca
> asumir que funcionó. Verificar siempre después:

```python
sol = client.records.list("solution", filter="uniquename eq '<UniqueName>'",
                            select=["solutionid"], top=1).first()
components = client.records.list("solutioncomponent",
    filter=f"_solutionid_value eq {sol['solutionid']}",
    select=["componenttype", "objectid"])
print(f"{len(components)} componentes en la solución")
```

### Component Type codes

| Componente            | `ComponentType` | Notas |
|-----------------------|------------------|-------|
| Entity / Table        | 1               | Incluye columnas y vistas automáticamente |
| Attribute / Column    | 2               | Solo si no se incluye la entity completa |
| Relationship          | 10              | |
| Global Option Set     | 9               | |
| Form                  | 24              | (Microsoft documenta `60` para Form en algunos comandos de PAC CLI — usar `24` para `AddSolutionComponent`/Web API, `60` si el comando específico de PAC lo pide) |
| View (SavedQuery)     | 26              | |
| Business Rule         | 29              | |
| Web Resource          | 61              | |
| Plugin Assembly       | 91              | |
| SDK Message Processing Step | 92        | |
| Connection Role       | 63              | |
| Canvas App            | 300             | |
| Custom API            | 10145           | |
| Security Role         | 14              | Ver `security-architect` — se agrega vía canal Git, no vía esta skill |
| Connector             | 371             | |

> ⚠️ **`Security Role` (14) sin re-verificar contra fuente oficial vigente.**
> **`Alternate Key` y `Duplicate Detection Rule` deliberadamente sin código documentado** —
> se encontró un caso real donde `ComponentType=44` para `DuplicateRule` fue rechazado
> ("Invalid component type provided"). Confirmar contra `$metadata` o documentación oficial
> vigente antes de scriptear cualquiera de los dos.

### Encontrar el nombre exacto de una solución

```bash
pac solution list --environment <url>
```

La columna `UniqueName` (sin espacios) es la que se usa en el resto de los comandos — el
`Display Name` puede tener espacios y no sirve para esto.

### Confirmar la solución activa (mismo criterio que las otras skills del paquete)

Antes de agregar un componente o crear algo dentro de una solución, seguí el mismo criterio
que `entity-builder`/`form-designer`/`view-designer`/`business-rule-engine`: leer el campo
`Solución` de `.d365-session.md` primero; si falta, está vacío, o `pac solution list` muestra
más de una candidata, preguntarle al usuario explícitamente — nunca asumir ni tomar la primera.

---

## Canal Pipeline — promoción a TEST/PROD (sin cambios, gate propio)

```
[DEV]                                    [Azure DevOps — git]              [TEST/PROD — pipeline]
crear_publisher (SDK)                          │                                    │
crear_solución (SDK)                            │                                    │
  agregar_componentes (PAC CLI)                 │                                    │
  versionar (1.0.0.X)                           │                                    │
       ──── export_solution_to_repo ───────────→│                                    │
                                          pac solution unpack (en el pipeline CI)     │
                                          commit a branch de la sesión                │
                                          create_deployment_settings                  │
                                          ──── Pull Request (confirmación explícita)  │
                                          merge → pipeline CI dispara automático      │
                                                 pac solution pack                    │
                                                 pac solution check (gate de calidad) │
                                                 ──── trigger_promotion_pipeline ─────→│
                                                                              gate de aprobación manual
                                                                              pac solution import --settings-file
```

### Pull — export + unpack para baseline local (nuevo, técnica de `dv-solution`)

Cuando haga falta traer el estado actual de DEV como baseline editable (no para promoción,
solo para trabajo local/versionado):

```bash
pac solution export --name <UniqueName> --path ./solutions/<UniqueName>.zip --managed false --environment <url>
pac solution unpack --zipfile ./solutions/<UniqueName>.zip --folder ./solutions/<UniqueName> --packagetype Unmanaged
```

> ⚠️ **Race condition real de Windows.** Correr `export` y `unpack` como comandos
> **separados** — encadenarlos inmediatamente puede pisar un lock transitorio del ZIP recién
> exportado. Si `unpack` falla con un error de archivo en uso, reintentar tras un momento, y
> verificar que la carpeta unpacked tenga los componentes esperados antes de borrar el zip.

```bash
rm ./solutions/<UniqueName>.zip   # el zip no es la fuente de verdad, la carpeta unpacked sí
git add ./solutions/<UniqueName> && git commit -m "chore: pull <UniqueName> baseline" && git push
```

### Promoción real — sigue exclusivamente el pipeline (sin cambios)

1. `export_solution_to_repo` — exporta DEV, commitea unmanaged al branch de la sesión.
2. `create_deployment_settings` — `pac solution create-settings`, commitea el JSON (valores
   reales por ambiente vía `config-generator`, secretos referenciando Key Vault, nunca en
   texto plano).
3. **Confirmación explícita antes de abrir el PR** — acción de "explicit permission
   required", sin excepción.
4. `trigger_promotion_pipeline` — solo después de PR mergeado (verificado, no asumido) y el
   usuario confirmó el ambiente target. El pipeline corre `pac solution pack` → `pac solution
   check` (gate de calidad, bloquea si hay errores críticos) → `pac solution import
   --settings-file` (managed, con Build Tools).
5. Para PROD: el mismo pipeline, con **gate de aprobación manual adicional** en Azure DevOps
   Environments — nunca disparado automáticamente, ni si TEST pasó todos los checks.
6. `get_pipeline_status` — polling, no bloquea el chat esperando.

---

## Validación post-import (nuevo, de `dv-solution`)

Después de cualquier import (vía pipeline), verificar que los componentes están vivos —
usando el SDK directo, sin scripts externos:

```python
# Tabla existe
info = client.tables.get("<logical_name>")

# Form publicado
forms = client.records.list("systemform",
    filter="objecttypecode eq '<entity>' and type eq <form_type_code>",  # 2=main, 7=quick create
    select=["name", "formid"], top=5)

# Vista existe
views = client.records.list("savedquery",
    filter="returnedtypecode eq '<entity>'", select=["name", "savedqueryid", "statuscode"], top=10)

# Errores de import
jobs = client.records.list("importjob",
    select=["importjobid", "solutionname", "startedon", "completedon", "progress"],
    orderby=["startedon desc"], top=5)
```

### Tabla de errores de validación

| Error | Causa | Fix |
|---|---|---|
| Tabla no aparece después de importar | Componente no está en la solución | Agregar vía `pac solution add-solution-component` |
| Chequeo de form falla inmediatamente | La publicación es asíncrona | Esperar 30 segundos y reintentar |
| Rol no asignado | Usuario no provisionado | Asignar vía `pac admin assign-user` o el Power Platform Admin Center |
| Import job en 0% | Todavía está corriendo | Volver a consultar en 60 segundos, no reimportar |

---

## Restricciones

- **Nunca** el import a TEST o PROD ocurre fuera del pipeline de Azure DevOps — sin
  excepción, es la garantía central de esta skill.
- **Nunca** se abre un Pull Request sin confirmación explícita del usuario en el chat.
- **Nunca** se dispara `trigger_promotion_pipeline` hacia PROD sin que el usuario haya
  confirmado explícitamente el ambiente y la versión — y aun así, el pipeline exige su propio
  gate de aprobación humana, independiente de esta confirmación conversacional.
- **Nunca usar el prefijo de publisher `new`** — siempre `axx`, o el que el usuario confirme
  explícitamente para un caso distinto.
- **Nunca asumir que el header `MSCRM.SolutionName` funcionó** — verificar siempre contra
  `solutioncomponent`, falla en silencio si está mal escrito.
- **Nunca encadenar `export` y `unpack`** en el mismo comando — race condition real en Windows.
- Si el environment target ya tiene la solución en versión superior a la que se quiere
  importar, se reporta al usuario — nunca se reintenta automáticamente con otra versión.
- **Nunca** eliminar componentes de una solución administrada — crear una patch solution o
  una nueva versión.
- Siempre confirmar el environment (`pac auth list` + `pac org who`) antes de cualquier
  operación — nunca asumir contra cuál se está trabajando.