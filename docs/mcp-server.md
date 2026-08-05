---
name: mcp-server
version: 2.1.0
description: >
  Especificación de implementación del MCP Server que conecta el agente Claude Cowork (vía Conector MCP) con
  Dataverse (acotado a DEV) y con Azure DevOps (Repos + Pipelines) para todo lo que promueve
  más allá de DEV. Incluye autenticación OAuth 2.0 con Azure AD, estructura de tools, manejo
  de errores con retry/backoff, y guía de despliegue. Ver ADR-001 para el razonamiento
  arquitectónico detrás de esta división.

tipo: infrastructure
audiencia: desarrollador que implementa el backend del agente
---

# MCP Server — Axxon Dataverse Architect

El MCP Server es el **backend** del agente: una **Azure Function** (Node.js 20 LTS) que expone
las tools del agente como endpoints HTTP. A partir de v2.0.0 tiene **dos responsabilidades
separadas**, no una:

1. **Dataverse Web API client — acotado a DEV.** Todo lo que necesita feedback conversacional
   inmediato (tablas, columnas, forms, business rules) sigue yendo directo contra el
   environment DEV de la sesión.
2. **Azure DevOps client — Repos + Pipelines.** Todo lo que es seguridad, environment
   variables, y cualquier promoción más allá de DEV, se resuelve escribiendo archivos a un
   branch y/o disparando un pipeline run. **El MCP Server nunca ejecuta `pac` por línea de
   comandos** — eso corre en el agente de build del pipeline, que sí es una máquina persistente
   con PAC CLI instalado. Una Azure Function serverless no es el lugar para shell out a un CLI.

---

## Arquitectura

```
Claude Cowork (skill `dataverse-architect` + skills específicas)
        │
        │ HTTP POST /api/tools/{toolName}
        │ Authorization: Bearer {token del Conector MCP de Cowork}
        ▼
Azure Function (D365 Architect MCP Server)
        │
        ├─── canal="dev-webapi" ──────────────────────────────┐
        │    1. Validar token inbound (firma + audience)      │
        │    2. Obtener token Dataverse (Client Credentials)   │
        │    3. Ejecutar request a Dataverse Web API (DEV)     │
        │    4. Transformar respuesta                           │
        │                                                       ▼
        │                                          Dataverse DEV Web API
        │
        └─── canal="git" | canal="pipeline" ──────────────────┐
             1. Validar token inbound (firma + audience)       │
             2. Obtener token Azure DevOps (Client Credentials)│
             3. Git: commit archivo(s) a branch, o             │
                Pipeline: trigger run / consultar estado       │
             4. Transformar respuesta                          │
                                                                 ▼
                                                    Azure DevOps (Repos + Pipelines)
```

---

## Stack tecnológico

| Componente             | Tecnología               | Justificación                                   |
|-------------------------|----------------------------|---------------------------------------------------|
| Runtime                | Azure Function v4 (Node.js 20) | Serverless, cold start < 2s, integración nativa con Azure AD |
| Autenticación inbound  | Azure AD (JWT validation, firma + audience) | El token emitido por el flujo OAuth del Conector MCP de Cowork se valida contra el App Registration del MCP — **con verificación de firma vía JWKS**, no solo decodificación |
| Autenticación outbound Dataverse | MSAL.js (Client Credentials) | Service Principal con permisos **acotados a DEV** |
| Autenticación outbound Azure DevOps | MSAL.js (Client Credentials) o PAT en Key Vault | Service Principal/PAT con permisos de Contribute en Repos + Queue Build en Pipelines |
| SDK Dataverse          | fetch nativo contra Web API v9.2 | Sin cambios respecto a v1.0.0 |
| SDK Azure DevOps       | Azure DevOps REST API (Git + Build) vía fetch nativo | Mismo patrón que `ado_client.py` (skill `ado-connector` de Axxon) — reutilizar tipos de excepción y backoff |
| Manejo de errores      | Structured error responses + retry con backoff exponencial | Alineado con el estándar ya adoptado en las skills `ado-*` de Axxon |
| Monitoreo              | Application Insights, con `correlationId` end-to-end | Une: turno de Cowork ↔ log de Function ↔ `.d365-session.md` del Project ↔ PR/pipeline de ADO |
| Secretos               | Azure Key Vault (reference en App Settings) | Nunca hardcodear en código |

---

## Estructura del proyecto

```
axxon-dataverse-architect-mcp/
├── src/
│   ├── functions/
│   │   ├── toolDispatcher.ts        # Entry point — recibe todos los tool calls, asigna correlationId
│   │   ├── tools/
│   │   │   ├── entityBuilder.ts     # canal=dev-webapi — create_table, add_column, etc.
│   │   │   ├── formDesigner.ts      # canal=dev-webapi — create_form, patch_form, etc.
│   │   │   ├── businessRules.ts     # canal=dev-webapi — create_business_rule, etc.
│   │   │   ├── securityArchitect.ts # canal=git — commit de roles/BU/teams a XML de solución
│   │   │   ├── configGenerator.ts   # canal=git — pac solution create-settings (delegado al pipeline), commit de settings JSON
│   │   │   └── solutionPackager.ts  # canal=dev-webapi (creación) + canal=pipeline (promoción)
│   ├── services/
│   │   ├── dataverseClient.ts       # Cliente HTTP para Dataverse Web API — SOLO DEV
│   │   ├── azureDevOpsClient.ts     # Cliente HTTP para Git API + Pipelines API — NUEVO en v2.0.0
│   │   └── authService.ts           # MSAL Client Credentials + validación de firma JWT inbound
│   ├── models/
│   │   ├── toolRequest.ts
│   │   └── toolResponse.ts
│   └── utils/
│       ├── xmlBuilder.ts            # Constructor de FormXml y de XML de solución (roles, env vars)
│       ├── guidGenerator.ts
│       ├── retryPolicy.ts           # Backoff exponencial — mismo patrón que ado_client.py
│       └── errorHandler.ts
├── host.json
├── local.settings.json.example
├── package.json
└── tsconfig.json
```

---

## Implementación del entry point (`toolDispatcher.ts`)

```typescript
import { app, HttpRequest, HttpResponseInit, InvocationContext } from "@azure/functions";
import { randomUUID } from "crypto";
import { validateInboundToken } from "../services/authService";
import { toolRegistry } from "./tools";

app.http("toolDispatcher", {
  methods: ["POST"],
  authLevel: "anonymous", // La auth la manejamos nosotros via JWT
  route: "tools/{toolName}",
  handler: async (request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
    const toolName = request.params.toolName;
    const correlationId = request.headers.get("x-correlation-id") ?? randomUUID();
    const startTime = Date.now();

    try {
      const authHeader = request.headers.get("authorization");
      if (!authHeader) return { status: 401, jsonBody: { error: "Missing authorization header", correlationId } };
      await validateInboundToken(authHeader.replace("Bearer ", ""));

      const body = await request.json() as Record<string, unknown>;

      const toolHandler = toolRegistry[toolName];
      if (!toolHandler) {
        return {
          status: 404,
          jsonBody: { error: `Tool '${toolName}' not found`, suggestion: `Available tools: ${Object.keys(toolRegistry).join(", ")}`, correlationId }
        };
      }

      const result = await toolHandler(body, { ...context, correlationId });

      // Log mínimo: nunca el body completo (puede contener CUIT, CBU, etc.)
      context.log(`[MCP] correlationId=${correlationId} tool=${toolName} duration=${Date.now() - startTime}ms status=success`);
      return { status: 200, jsonBody: { ...result, correlationId } };

    } catch (error: any) {
      context.error(`[MCP] correlationId=${correlationId} tool=${toolName} error=${error.message}`);
      return {
        status: error.statusCode ?? 500,
        jsonBody: {
          error: error.message,
          details: error.details ?? "An unexpected error occurred",
          suggestion: error.suggestion ?? "Check the MCP Server logs in Application Insights",
          correlationId
        }
      };
    }
  }
});
```

> **`correlationId`**: se genera una vez por tool call, se loguea en Application Insights, se
> devuelve a Claude, y la skill `dataverse-architect` lo anota en `.d365-session.md` del Cowork
> Project activo. Cuando el canal es `git`/`pipeline`, el mismo `correlationId` se agrega como
> tag al commit o al pipeline run, cerrando la trazabilidad completa: chat → log de Function →
> PR/pipeline.

---

## Cliente Dataverse (`dataverseClient.ts`) — sin cambios de fondo, acotado a DEV

```typescript
import { getDataverseToken } from "./authService";
import { withRetry } from "../utils/retryPolicy";

const API_VERSION = "v9.2";

export class DataverseClient {
  private environmentUrl: string;

  constructor(environmentUrl: string) {
    this.environmentUrl = environmentUrl.replace(/\/$/, "");
    // Guard real (no depende de ninguna tabla de sesión, retirada en RC1 — ver ADR-003):
    // el Service Principal de este MCP Server (por instancia de cliente, ver ADR-002) tiene
    // permisos de System Customizer ÚNICAMENTE en el environment DEV de ese cliente. Si esta
    // URL apuntara a TEST/PROD, Dataverse responde 403 porque el Service Principal no tiene
    // Application User ahí — el guard vive en los permisos, no en un lookup de contexto.
  }

  private async getHeaders(): Promise<HeadersInit> {
    const token = await getDataverseToken(this.environmentUrl);
    return {
      "Authorization": `Bearer ${token}`,
      "Content-Type": "application/json",
      "OData-MaxVersion": "4.0",
      "OData-Version": "4.0",
      "Accept": "application/json",
      "Prefer": "odata.include-annotations=*"
    };
  }

  async get<T>(path: string): Promise<T> {
    return withRetry(async () => {
      const url = `${this.environmentUrl}/api/data/${API_VERSION}/${path}`;
      const response = await fetch(url, { headers: await this.getHeaders() });
      if (!response.ok) await this.handleError(response, "GET", path);
      return response.json();
    });
  }

  async post<T>(path: string, body: unknown): Promise<T> {
    return withRetry(async () => {
      const url = `${this.environmentUrl}/api/data/${API_VERSION}/${path}`;
      const response = await fetch(url, {
        method: "POST",
        headers: await this.getHeaders(),
        body: JSON.stringify(body)
      });
      if (!response.ok) await this.handleError(response, "POST", path);

      const entityId = response.headers.get("OData-EntityId");
      if (response.status === 204 && entityId) {
        const guid = entityId.match(/\(([^)]+)\)/)?.[1];
        return { id: guid } as T;
      }
      return response.status === 204 ? {} as T : response.json();
    });
  }

  private async handleError(response: Response, method: string, path: string): Promise<never> {
    const body = await response.json().catch(() => ({}));
    const dvError = body?.error?.message ?? "Unknown Dataverse error";
    const dvCode = body?.error?.code ?? "unknown";

    const knownErrors: Record<string, string> = {
      "0x80040220": "Ya existe un componente con ese nombre — verificar con get_metadata antes de crear.",
      "0x8004F036": "Dependencia faltante — otro componente requiere este antes de importar.",
      "0x80040217": "No se puede eliminar este componente porque hay datos asociados.",
    };

    throw {
      message: knownErrors[dvCode] ?? `Dataverse error (${dvCode}): ${dvError}`,
      statusCode: response.status,
      details: `${method} ${path} → HTTP ${response.status}`,
      retryable: response.status === 429 || response.status >= 500,
      suggestion: dvCode === "0x80040220"
        ? "Verificá si el componente ya existe con get_table o get_column antes de crearlo."
        : "Revisá los logs de Application Insights para el trace completo."
    };
  }
}
```

---

## Cliente Azure DevOps (`azureDevOpsClient.ts`) — NUEVO en v2.0.0

Reutiliza el mismo patrón de excepciones tipadas, retry y paginación que `ado_client.py`
(skill `ado-connector`), portado a TypeScript para que el MCP Server no reimplemente la lógica
desde cero.

```typescript
import { getAzureDevOpsToken } from "./authService";
import { withRetry } from "../utils/retryPolicy";

const API_VERSION = "7.1";

export class AzureDevOpsClient {
  constructor(private organization: string, private project: string) {}

  private async getHeaders(): Promise<HeadersInit> {
    const token = await getAzureDevOpsToken();
    return { "Authorization": `Bearer ${token}`, "Content-Type": "application/json" };
  }

  /** Commitea uno o más archivos a un branch. Crea el branch si no existe (a partir de main). */
  async commitFiles(repoId: string, branch: string, files: Array<{ path: string; content: string }>, comment: string): Promise<{ commitId: string; prUrl?: string }> {
    return withRetry(async () => {
      // 1. Resolver el objectId del último commit del branch (o de main, si el branch es nuevo)
      // 2. POST a /_apis/git/repositories/{repoId}/pushes con los changes (add/edit)
      // 3. Devolver el commitId — la creación de PR es un paso separado y explícito, nunca automático
      // ... implementación completa en el repo del proyecto
      throw new Error("Implementar contra Azure DevOps Git REST API");
    });
  }

  /** Dispara un run de pipeline. No aprueba gates — eso lo hace un humano en Azure DevOps. */
  async triggerPipeline(pipelineId: number, branch: string, parameters?: Record<string, string>): Promise<{ runId: number; url: string }> {
    return withRetry(async () => {
      // POST a /_apis/pipelines/{pipelineId}/runs
      throw new Error("Implementar contra Azure DevOps Pipelines REST API");
    });
  }

  /** Solo lectura — usado para reportar estado al usuario sin disparar nada. */
  async getPipelineRunStatus(pipelineId: number, runId: number): Promise<{ state: string; result?: string; url: string }> {
    return withRetry(async () => {
      throw new Error("Implementar contra Azure DevOps Pipelines REST API");
    });
  }
}
```

> **Nota de implementación:** `commitFiles` nunca abre un Pull Request automáticamente — deja
> el commit en el branch y devuelve el link. La apertura del PR es una acción de canal
> "Explicit permission required" (ver reglas de seguridad del agente): el orchestrator debe
> pedir confirmación antes de que cualquier skill abra un PR, aunque commitear al branch de
> trabajo no la requiera.

---

## Servicio de autenticación (`authService.ts`) — corrige gap de seguridad de v1.0.0

> **Hallazgo de la revisión de arquitectura:** la v1.0.0 decodificaba el JWT inbound y
> comparaba `aud`/`appid`, pero **nunca verificaba la firma**. Cualquiera que supiera el
> `App Registration ID` podía construir un token falso y sería aceptado. Esto queda corregido
> en v2.0.0 con verificación de firma vía JWKS — es un requisito no negociable antes de
> desplegar contra cualquier environment real, y más aún en un cliente bancario.

```typescript
import { ConfidentialClientApplication } from "@azure/msal-node";
import jwt from "jsonwebtoken";
import jwksClient from "jwks-rsa";

const tokenCache: Map<string, { token: string; expiresAt: number }> = new Map();

export async function getDataverseToken(environmentUrl: string): Promise<string> {
  return getCachedToken(environmentUrl, [`${environmentUrl}/.default`]);
}

export async function getAzureDevOpsToken(): Promise<string> {
  return getCachedToken("azure-devops", ["499b84ac-1321-427f-aa17-267ca6975798/.default"]);
}

async function getCachedToken(cacheKey: string, scopes: string[]): Promise<string> {
  const cached = tokenCache.get(cacheKey);
  if (cached && cached.expiresAt > Date.now() + 300_000) return cached.token;

  const cca = new ConfidentialClientApplication({
    auth: {
      clientId: process.env.AZURE_AD_CLIENT_ID!,
      clientSecret: process.env.AZURE_AD_CLIENT_SECRET!,
      authority: `https://login.microsoftonline.com/${process.env.AZURE_AD_TENANT_ID}`
    }
  });

  const result = await cca.acquireTokenByClientCredential({ scopes });
  if (!result?.accessToken) throw new Error(`Failed to acquire token for ${cacheKey}`);

  tokenCache.set(cacheKey, {
    token: result.accessToken,
    expiresAt: result.expiresOn?.getTime() ?? Date.now() + 3600_000
  });
  return result.accessToken;
}

const jwks = jwksClient({
  jwksUri: `https://login.microsoftonline.com/${process.env.AZURE_AD_TENANT_ID}/discovery/v2.0/keys`
});

function getSigningKey(kid: string): Promise<string> {
  return new Promise((resolve, reject) => {
    jwks.getSigningKey(kid, (err, key) => {
      if (err || !key) return reject(err ?? new Error("Signing key not found"));
      resolve(key.getPublicKey());
    });
  });
}

export async function validateInboundToken(token: string): Promise<void> {
  const decoded = jwt.decode(token, { complete: true });
  if (!decoded || typeof decoded === "string" || !decoded.header.kid) {
    throw { message: "Invalid token format", statusCode: 401 };
  }

  const signingKey = await getSigningKey(decoded.header.kid);
  const expectedAud = process.env.MCP_APP_REGISTRATION_ID;

  try {
    jwt.verify(token, signingKey, {
      audience: expectedAud,
      issuer: `https://login.microsoftonline.com/${process.env.AZURE_AD_TENANT_ID}/v2.0`,
      algorithms: ["RS256"]
    });
  } catch (err) {
    throw {
      message: "Invalid token signature or claims",
      statusCode: 401,
      suggestion: "Verificar que el Conector MCP agregado en Cowork (Customize → Connectors) usa el App Registration correcto del MCP Server"
    };
  }
}
```

---

## Política de retry (`retryPolicy.ts`) — alineada con el estándar `ado_client.py`

```typescript
export async function withRetry<T>(fn: () => Promise<T>, maxAttempts = 3): Promise<T> {
  let lastError: any;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error: any) {
      lastError = error;
      if (!error.retryable || attempt === maxAttempts) throw error;
      const backoffMs = Math.min(1000 * 2 ** (attempt - 1), 8000);
      await new Promise(resolve => setTimeout(resolve, backoffMs));
    }
  }
  throw lastError;
}
```

Solo se reintentan errores marcados `retryable: true` (HTTP 429 y 5xx) — nunca errores de
negocio como `DuplicateDetected` o validación fallida.

---

## Variables de entorno requeridas (App Settings de la Azure Function)

| Variable                     | Descripción                                              | Fuente          |
|--------------------------------|--------------------------------------------------------------|-------------------|
| `AZURE_AD_TENANT_ID`         | Tenant ID del Azure AD del cliente                       | Key Vault ref   |
| `AZURE_AD_CLIENT_ID`         | Client ID del Service Principal del MCP Server (DEV)     | Key Vault ref   |
| `AZURE_AD_CLIENT_SECRET`     | Secret del Service Principal (DEV)                        | Key Vault ref   |
| `MCP_APP_REGISTRATION_ID`    | App ID del App Registration expuesto al Conector MCP de Cowork | Key Vault ref   |
| `AZURE_DEVOPS_ORG`           | Organización de Azure DevOps                              | App Setting     |
| `AZURE_DEVOPS_PROJECT`       | Proyecto de Azure DevOps                                   | App Setting     |
| `AZURE_DEVOPS_PAT_OR_SP`     | Credencial para Git + Pipelines API                        | Key Vault ref   |
| `APPLICATIONINSIGHTS_CONNECTION_STRING` | Connection string de App Insights             | Key Vault ref   |
| `DEFAULT_LANGUAGE_CODE`      | Código de idioma default para labels (3082 = es-ES)      | `3082`          |

**Eliminado respecto a v1.0.0**: `BLOB_STORAGE_CONNECTION` — los `.zip` de exportación ahora
se generan y almacenan como artifact del pipeline de Azure DevOps, no en Blob Storage propio.

---

## Configuración del App Registration en Azure AD

Sin cambios respecto a v1.0.0, con un agregado: el Service Principal del MCP Server necesita
**dos identidades de permisos separadas**, no una:

```
App Registration "MCP Server — Axxon Dataverse Architect"
  Tipo: Single-tenant (del cliente)
  Redirect URIs: ninguna (daemon flow — Client Credentials)

  Permisos Dataverse:
    - Aplicado SOLO como Application User en el environment DEV
    - Rol: "System Customizer" — nunca System Administrator, nunca en TEST/PROD

  Permisos Azure DevOps:
    - Service Connection o PAT con scope: Code (Read & Write), Build (Read & Execute)
    - Sin permisos de administración del proyecto — solo lo necesario para commitear y
      disparar runs
```

---

## Registro del MCP Server como Conector en Cowork

A diferencia de Copilot Studio (registro manual de una spec Swagger en AI Capabilities), el
protocolo MCP se auto-describe: no hace falta mantener ninguna spec OpenAPI aparte (ver
ADR-003, Decisión 3 — se elimina esa sección de `06-config-generator.md`).

1. En Claude Desktop → Customize → Connectors → agregar conector.
2. Tipo: conector remoto (Streamable HTTP + OAuth).
3. URL: `https://func-axx-{cliente}-mcp.azurewebsites.net/api` (instancia de ese cliente,
   ver ADR-002).
4. Autenticación: OAuth 2.0 contra el App Registration configurado en Azure AD.
5. Cowork descubre las tools expuestas automáticamente vía el handshake del protocolo MCP
   (`tools/list`) — no requiere registro manual de cada tool.

---

## Despliegue — plantilla por cliente (ver ADR-002)

El código de este MCP Server es **una plantilla única** que Axxon mantiene en un solo repo.
Cada cliente obtiene su propia instancia desplegada desde esa plantilla — nunca un runtime
compartido entre clientes (ver ADR-002 para el razonamiento completo).

### Parámetros que varían por instancia

| Parámetro | Ejemplo | Fuente |
|---|---|---|
| Nombre de la Function App | `func-axx-{cliente}-mcp` | Convención de nombres del proyecto |
| Suscripción de hosting | Del cliente (default) o de Axxon (excepción documentada) | Definido en discovery/Phase 0 |
| Tenant ID Azure AD | Tenant del cliente | Provisto por el cliente |
| Environment DEV target | URL del Dataverse DEV del cliente | Provisto por el cliente |
| Azure DevOps Project | Project dedicado del cliente en la organización de Axxon | Creado al iniciar el proyecto |

### Despliegue vía Azure CLI (instancia de un cliente)

```bash
# Ejecutado dentro de la suscripción target (del cliente o de Axxon, según ADR-002)
az group create --name rg-axx-{cliente}-agent --location eastus2
az storage account create --name saaxx{cliente}agent --resource-group rg-axx-{cliente}-agent --sku Standard_LRS
az functionapp create \
  --name func-axx-{cliente}-mcp \
  --resource-group rg-axx-{cliente}-agent \
  --storage-account saaxx{cliente}agent \
  --consumption-plan-location eastus2 \
  --runtime node \
  --runtime-version 20 \
  --functions-version 4

az functionapp identity assign --name func-axx-{cliente}-mcp --resource-group rg-axx-{cliente}-agent
az keyvault set-policy --name kv-axx-{cliente} \
  --object-id <managed-identity-object-id> \
  --secret-permissions get list

az functionapp config appsettings set \
  --name func-axx-{cliente}-mcp \
  --resource-group rg-axx-{cliente}-agent \
  --settings \
    "AZURE_AD_TENANT_ID=@Microsoft.KeyVault(SecretUri=https://kv-axx-{cliente}.vault.azure.net/secrets/tenant-id/)" \
    "AZURE_AD_CLIENT_ID=@Microsoft.KeyVault(SecretUri=https://kv-axx-{cliente}.vault.azure.net/secrets/client-id/)" \
    "AZURE_AD_CLIENT_SECRET=@Microsoft.KeyVault(SecretUri=https://kv-axx-{cliente}.vault.azure.net/secrets/client-secret/)" \
    "AZURE_DEVOPS_ORG=axxon" \
    "AZURE_DEVOPS_PROJECT={cliente}"

cd axxon-dataverse-architect-mcp
npm run build
func azure functionapp publish func-axx-{cliente}-mcp
```

### Bicep parametrizado (recomendado para instancias repetibles)

El repo de la plantilla incluye `infra/main.bicep` con los parámetros de la tabla anterior
expuestos como `param`. Instanciar un cliente nuevo es correr el pipeline "provisioning" con
esos valores — no copiar y editar el código a mano.

### Actualización de instancias existentes (pendiente de diseño — ver ADR-002)

Cuando se libera una nueva versión de la plantilla (ej: un parche de seguridad), hoy no existe
un mecanismo automatizado para propagarla a todas las instancias de clientes activos. Es el
ítem abierto más urgente del backlog de este agente — con más de 2-3 clientes activos, aplicar
parches a mano deja de ser sostenible.

---

| Límite                        | Valor                        | Mitigación                                    |
|---------------------------------|---------------------------------|---------------------------------------------------|
| Timeout Azure Function (Consumption) | 10 min (configurar 9min) | Operaciones largas (pipeline runs) se resuelven por polling asíncrono, no dentro del mismo request |
| Dataverse Web API rate limit  | 6.000 req/5min por usuario   | El Service Principal DEV tiene su propio bucket; `withRetry` maneja 429 |
| Azure DevOps REST API rate limit | Ver límites por organización | `withRetry` con backoff exponencial, igual que Dataverse |
| Token del Conector MCP de Cowork | 60 min de vida (según config. OAuth) | Se renueva automáticamente según el flujo OAuth del conector   |
| Cold start de Azure Function  | 1-3 segundos                 | Usar Premium plan o Always-On para el agente en producción |

---

## Restricciones de seguridad del MCP Server

- **Nunca** loguear el cuerpo completo de los requests en Application Insights — puede
  contener datos sensibles (CUIT, CBU, etc.). Loguear solo: `correlationId`, tool name,
  duration, status.
- **Nunca** retornar stack traces en las respuestas HTTP — usar mensajes de error amigables.
- **Siempre** validar la firma del token inbound (no solo `aud`/`appid`) antes de ejecutar
  cualquier operación.
- El Service Principal de Dataverse **nunca** tiene permisos fuera de DEV. La promoción a
  TEST/PROD es responsabilidad exclusiva del Service Connection del pipeline de Azure DevOps,
  gobernado por sus propios gates de aprobación — el MCP Server no lo dispara sin ese circuito.
- El MCP Server **nunca** ejecuta binarios ni CLIs (`pac`, `git`, etc.) dentro de la Azure
  Function — toda esa ejecución vive en el agente de build del pipeline.
- Rotar el Client Secret / PAT antes de su vencimiento y actualizar la referencia en Key Vault.
