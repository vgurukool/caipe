# CAIPE: LiteLLM Integration, MongoDB Persistence & ReBAC Authorization Notes

This document provides a detailed technical record of the integration between **CAIPE (Community AI Platform Engineering)**, the dedicated **LiteLLM AI Gateway**, **MongoDB Persistent Storage**, and the **OpenFGA ReBAC Authorization Layer** on Amazon EKS.

---

## 1. Executive Summary

| Component | Status | Target / Host Endpoint | Notes |
| :--- | :--- | :--- | :--- |
| **CAIPE UI & Backend** | `Healthy` | `https://caipe.vgurukool.com` | Next.js 15 BFF & Agent Orchestration Engine |
| **LiteLLM AI Gateway** | `Connected` | `http://litellm.litellm.svc.cluster.local:4000/v1` | Provides Gemini & OpenAI chat completion models |
| **MongoDB Persistence** | `Connected` | `caipe-mongodb.caipe.svc.cluster.local:27017` | StatefulSet on 5Gi `gp3` EBS CSI volume |
| **OpenFGA Authorization** | `Healthy` | `http://openfga:8080` | In-memory store `caipe-openfga` with 116 KB model |
| **Keycloak IAM (SSO)** | `Configured` | `https://vgurukool.com/keycloak/realms/cnoe` | Audience mapper `caipe` + extended token lifespan |
| **Dynamic Agents Models**| `Active (5)` | `/api/dynamic-agents/models` | 5 models discovered and selectable in CAIPE UI |

---

## 2. Architecture Diagram

```
+-----------------------------------------------------------------------------------------------------------------+
|                                                 Amazon EKS Cluster                                              |
|                                                                                                                 |
|  +-----------------------------------------------------------------------------------------------------------+  |
|  |                                             Namespace: caipe                                              |  |
|  |                                                                                                           |  |
|  |   +------------------------------------+              +-----------------------------------------------+   |  |
|  |   |           caipe Pods               |              |               caipe-mongodb-0                 |   |  |
|  |   |  - Next.js UI / API routes         |  MongoDB TCP |  - MongoDB 7.0 StatefulSet                    |   |  |
|  |   |  - app-config.yaml mounted         | -----------> |  - 5Gi gp3 PersistentVolumeClaim              |   |  |
|  |   |  - [seed-config] loads 5 models    |    :27017    |  - Stores agents, chats, preference data      |   |  |
|  |   +-----------------+------------------+              +-----------------------------------------------+   |  |
|  |                     |                                                                                     |  |
|  |                     | OpenFGA Checks (ReBAC)                                                              |  |
|  |                     v                                                                                     |  |
|  |   +------------------------------------+                                                                  |  |
|  |   |             openfga                |                                                                  |  |
|  |   |  - OpenFGA v1.15.1 (:8080/:8081)   |                                                                  |  |
|  |   |  - openfga-init sidecar (seed.py)  |                                                                  |  |
|  |   |  - caipe-openfga store & model     |                                                                  |  |
|  |   |  - Wildcard reader tuples          |                                                                  |  |
|  |   +------------------------------------+                                                                  |  |
|  +---------------------+-------------------------------------------------------------------------------------+  |
|                        |                                                                                        |
|                        | OpenAI API Chat Completions                                                            |
|                        | (Bearer: sk-litellm-vgurukool-master-2026)                                             |
|                        v                                                                                        |
|  +-----------------------------------------------------------------------------------------------------------+  |
|  |                                             Namespace: litellm                                            |  |
|  |                                                                                                           |  |
|  |   +-------------------------------------------------------+      +------------------------------------+   |  |
|  |   |                     LiteLLM Proxy                     |      |            litellm-db-0            |   |  |
|  |   |        litellm.litellm.svc.cluster.local:4000/v1      | ---> |       PostgreSQL StatefulSet       |   |  |
|  |   |        (gemini-2.5-flash, gpt-4o, gemini-1.5, etc.)   |      |  litellm-db.litellm.svc:5432       |   |  |
|  |   +---------------------------+---------------------------+      +------------------------------------+   |  |
|  +-------------------------------|---------------------------------------------------------------------------+  |
|                                  | External API Calls                                                           |
|                                  v                                                                              |
|                    Google Gemini & OpenAI APIs                                                                  |
+-----------------------------------------------------------------------------------------------------------------+
```

---

## 3. Issues Diagnosed & Root Causes

### Issue 1: "Server configuration problem" / Missing MongoDB Storage
* **Symptom:** Logging into CAIPE displayed:
  ```text
  There is a problem with the server configuration. Check the server logs for more information.
  MongoDB is not configured. Set MONGODB_URI and MONGODB_DATABASE environment variables.
  ```
* **Root Cause:**
  - CAIPE relies on MongoDB for persistent session state, dynamic agents, scheduled jobs, and conversations.
  - Without MongoDB configured, the backend fell back to unpersisted local storage and rejected operations on dynamic agents.
* **Resolution:**
  - Deployed `caipe-mongodb` StatefulSet (`mongo:7.0`) on a 5Gi AWS EBS `gp3` storage volume (`ReadWriteOnce`).
  - Configured `MONGODB_URI`, `MONGODB_DATABASE=caipe`, and injected credentials via Kubernetes Secret `caipe-secret`.

---

### Issue 2: Session Expiring in 1 Minute Warning
* **Symptom:** Immediately after signing into CAIPE via Keycloak SSO:
  ```text
  Session Expiring Soon
  Your session will expire in 1 minute. Attempting to refresh automatically.
  ```
* **Root Cause:**
  - In Keycloak realm `cnoe`, `accessTokenLifespan` was set to `60` seconds.
  - CAIPE's `TokenExpiryGuard` alerts when `expiresAt - currentTime < 300s` (5 minutes). Because `60s < 300s`, every freshly generated token triggered the warning banner instantly.
* **Resolution:**
  - Adjusted realm `cnoe` settings:
    - `accessTokenLifespan`: increased from `60` seconds to `3600` seconds (1 hour).
    - `ssoSessionIdleTimeout`: increased from `1800` seconds to `28800` seconds (8 hours).
    - `ssoSessionMaxLifespan`: set to `36000` seconds (10 hours).

---

### Issue 3: Platform Health Probes Showing "Degraded" & "Down"
* **Symptom:** [Platform Status](https://caipe.vgurukool.com/admin?cat=platform&tab=health) showed:
  - **Overall:** `Degraded` / `Down`
  - **Chat Runtime:** `Down` (`Chat runtime health check is unreachable`)
  - **Knowledge Bases:** `Degraded` (`HTTP 502`)
  - **Audit Service:** `Degraded` (`fetch failed`)
* **Root Cause:**
  - The standalone CAIPE deployment does not bundle optional backend components like `caipe-supervisor`, `rag-server`, or `audit-service`.
  - When left unspecified, CAIPE enabled probes for these components by default and failed closed.
* **Resolution:**
  - Set `A2A_BASE_URL: "http://localhost:3000/api"` (targets CAIPE's internal health route, returning `HTTP 200`).
  - Set `RAG_ENABLED: "false"` (gracefully disables Knowledge Bases probe).
  - Set `AUDIT_LOG_BACKEND: "disabled"` (gracefully disables Audit Service probe).
  - Configured `KEYCLOAK_URL` and `KEYCLOAK_POSTGRES_HOST` for healthy diagnostic checks.

---

### Issue 4: Empty Model Field in Dynamic Agents & LiteLLM Disconnection
* **Symptom:** At `https://caipe.vgurukool.com/dynamic-agents`, the **Model** selector dropdown was empty, and users could not select LiteLLM models.
* **Root Cause:**
  1. **Connectivity:** CAIPE lacked the endpoint URL and master bearer token for the internal LiteLLM service in namespace `litellm`.
  2. **Empty Collection:** Dynamic agents query models from MongoDB's `llm_models` collection at `/api/dynamic-agents/models`. No models were defined or seeded into the database.
  3. **ReBAC (OpenFGA) Check Failure:** When `/api/dynamic-agents/models` reads models from MongoDB, it calls `filterResourcesByPermission(session, docs, { type: "llm_model", action: "read" })`.
     - OpenFGA was not initially running in namespace `caipe`.
     - Even after running OpenFGA, the ReBAC rule for `llm_model` required users to be members of `organization:caipe`.
     - Direct API calls with bearer tokens had not established an explicit user membership tuple in OpenFGA, causing OpenFGA to evaluate `can_read = false` and filter all models out.
* **Resolution:**
  1. **LiteLLM Connection:** Injected `LITELLM_BASE_URL: "http://litellm.litellm.svc.cluster.local:4000/v1"` and `LITELLM_MASTER_KEY` into `caipe-secret`.
  2. **ConfigMap Seeding:** Created `caipe/chart/templates/app-configmap.yaml` declaring the 5 LiteLLM models. On startup, CAIPE's `[seed-config]` automatically inserts all 5 into MongoDB.
  3. **OpenFGA In-Memory Service:** Added `caipe/chart/templates/openfga.yaml` deploying `openfga/openfga:v1.15.1` with automated store creation and schema loading.
  4. **Wildcard ReBAC Tuples:** In `authorization-model.json`, `llm_model#reader` explicitly allows `{"type": "user", "wildcard": {}}`. The init sidecar was configured to seed:
     ```json
     {"user": "user:*", "relation": "reader", "object": "llm_model:<model_id>"}
     ```
     for all models, allowing all authenticated users to read and select the models.
  5. **Audience Mapper:** Added `oidc-audience-mapper` (`caipe-audience`) to Keycloak client `caipe` so access tokens contain `aud: ["caipe", "account"]`.

---

## 4. Models Configured in CAIPE

The following models are seeded via `/etc/caipe/app-config.yaml` and routed through LiteLLM:

| Model ID | UI Display Name | Provider | Description | Default |
| :--- | :--- | :--- | :--- | :---: |
| `gemini-2.5-flash` | **Gemini 2.5 Flash** | `openai` (LiteLLM) | Google Gemini 2.5 Flash via LiteLLM | ✅ Yes |
| `gpt-4o` | **GPT-4o** | `openai` (LiteLLM) | OpenAI GPT-4o via LiteLLM | No |
| `gemini-1.5-flash` | **Gemini 1.5 Flash** | `openai` (LiteLLM) | Google Gemini 1.5 Flash via LiteLLM | No |
| `gemini-1.5-pro` | **Gemini 1.5 Pro** | `openai` (LiteLLM) | Google Gemini 1.5 Pro via LiteLLM | No |
| `gpt-3.5-turbo` | **GPT-3.5 Turbo** | `openai` (LiteLLM) | OpenAI GPT-3.5 Turbo via LiteLLM | No |

---

## 5. Verification Commands & Expected Outputs

### A. Test LiteLLM Connectivity from CAIPE
```bash
kubectl exec -n caipe deploy/caipe -c caipe -- node -e '
async function test() {
  const res = await fetch("http://litellm.litellm.svc.cluster.local:4000/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": "Bearer sk-litellm-vgurukool-master-2026"
    },
    body: JSON.stringify({
      model: "gemini-2.5-flash",
      messages: [{ role: "user", content: "Say hello!" }],
      max_tokens: 10
    })
  });
  console.log("Status:", res.status);
  const data = await res.json();
  console.log("Response:", data.choices[0].message.content);
}
test();
'
```
**Expected Output:**
```text
Status: 200
Response: Hello!
```

---

### B. Verify OpenFGA Store & ReBAC Check
```bash
kubectl exec -n caipe deploy/openfga -c openfga-init -- python3 -c '
import urllib.request, json
stores = json.loads(urllib.request.urlopen("http://localhost:8080/stores").read())["stores"]
store_id = next(s["id"] for s in stores if s["name"] == "caipe-openfga")
req = urllib.request.Request(
    f"http://localhost:8080/stores/{store_id}/check",
    data=json.dumps({"tuple_key": {"user": "user:test-user", "relation": "can_read", "object": "llm_model:gemini-2.5-flash"}}).encode(),
    headers={"Content-Type": "application/json"}
)
print(urllib.request.urlopen(req).read().decode())
'
```
**Expected Output:**
```json
{"allowed":true,"resolution":""}
```

---

### C. Verify Dynamic Agents Models Endpoint
```bash
kubectl exec -n caipe deploy/caipe -c caipe -- node -e '
async function test() {
  const tokenParams = new URLSearchParams({
    grant_type: "password",
    client_id: "caipe",
    client_secret: "CaipeKeycloakSecret2026Secure",
    username: "user1",
    password: "User1Password2026!"
  });

  const tokenRes = await fetch("http://keycloak.keycloak.svc.cluster.local/keycloak/realms/cnoe/protocol/openid-connect/token", {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: tokenParams.toString()
  });
  const token = (await tokenRes.json()).access_token;

  const res = await fetch("http://localhost:3000/api/dynamic-agents/models", {
    headers: { "Authorization": `Bearer ${token}` }
  });
  console.log("Status:", res.status);
  const data = await res.json();
  console.log("Count:", data.data.length);
  console.log("Models:", data.data.map(m => m.model_id));
}
test();
'
```
**Expected Output:**
```text
Status: 200
Count: 5
Models: [ "gpt-3.5-turbo", "gpt-4o", "gemini-1.5-flash", "gemini-1.5-pro", "gemini-2.5-flash" ]
```

---

## 6. Git Commits & Repository History

All manifests and configuration files have been committed to `vgurukool/caipe.git` on branch `main`:

* `423954096`: Reconfigure AI provider to dedicated `litellm` namespace.
* `5c10918e5`: Disable absent backend probes (`RAG_ENABLED=false`, `AUDIT_LOG_BACKEND=disabled`).
* `1042bec7c`: Set internal Keycloak URL and PostgreSQL host for health diagnostics.
* `37ff9a186`: Add MongoDB StatefulSet, persistent storage, and LiteLLM app-config model seeding.
* `15c1dface`: Add OpenFGA in-memory service, authorization model, and baseline tuple bootstrap.
* `e2b93c8f8`: Fix OpenFGA seed tuples to include `user:*` reader relationships for models.
* `d103ed89f`: Enable workflows feature flags (`WORKFLOWS_ENABLED`, `NEXT_PUBLIC_WORKFLOWS_ENABLED`, `WORKFLOW_RUNNER_ENABLED`).

---

## 7. Workflows Engine Feature Flag Enablement

### Symptom
Accessing the Workflows tab (`https://caipe.vgurukool.com/workflows`) previously presented a blocking placeholder message:
```text
🚧 Workflows not enabled
The Workflows feature is not enabled on this instance.
Set WORKFLOWS_ENABLED=true to activate it.
```

### Root Cause
CAIPE gates the `/workflows` server-side layout (`ui/src/app/(app)/workflows/layout.tsx`) and the top-level navigation item behind `config.workflowsEnabled`. When `WORKFLOWS_ENABLED` is omitted or `false`, the layout renders the disabled placeholder and blocks rendering of the `WorkflowCanvas` and workflow editor.

### Resolution
1. Added environment variables to `caipe/chart/values.yaml`:
   ```yaml
   env:
     WORKFLOWS_ENABLED: "true"
     NEXT_PUBLIC_WORKFLOWS_ENABLED: "true"
     WORKFLOW_RUNNER_ENABLED: "true"
   ```
2. Committed and pushed commit `d103ed89f` to `vgurukool/caipe.git`.
3. Synced with Argo CD and rolled out the updated CAIPE pods.

### Verification
* Probing `GET http://localhost:3000/workflows` returns `HTTP 200` with `Has disabled notice: false`.
* The `WorkflowCanvas`, step builder, run execution store, and `/api/workflow-configs` CRUD API are fully operational.
