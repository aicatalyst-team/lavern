# PoC Plan: Lavern

## Project Classification
- **Type:** llm-app
- **Key Technologies:** TypeScript, Node.js, Fastify, Anthropic Claude SDK, OpenAI SDK, Mistral AI, SQLite (better-sqlite3), Vite (frontend), Puppeteer, Stripe, Zod
- **ODH Relevance:** Lavern is a multi-agent LLM application that orchestrates 67 AI agent prompts through a structured debate protocol for legal document analysis. It demonstrates advanced agentic AI patterns (multi-agent coordination, verification loops, human-in-the-loop gates) that are directly relevant to the Open Data Hub ecosystem. The system supports multiple LLM providers including local inference via Ollama, making it a strong candidate for demonstrating LLM application deployment on OpenShift AI.

## PoC Objectives
What we want to prove:
1. The Lavern Fastify API server builds and starts successfully in a containerized environment on OpenShift
2. The bundled frontend dashboard (Vite build) is served correctly from the same container
3. The health check endpoint confirms the system is operational
4. Core API routes (capabilities, agents list) respond correctly, proving the agent definitions and routing are loaded
5. The SQLite database initializes properly with persistent storage via a PVC
6. The system can accept an Anthropic API key (or Mistral/Ollama config) via environment variables for real engagement execution

## Infrastructure Requirements
- **Inference Server:** none — Lavern includes its own Fastify-based API server and calls external LLM APIs (Anthropic, Mistral, or local Ollama)
- **Vector Database:** none — the system uses SQLite for persistence (precedent board, knowledge base, session state)
- **Embedding Model:** none
- **GPU Required:** No — all inference is delegated to external LLM providers or local Ollama
- **Persistent Storage:** 1Gi PVC for SQLite database (`/app/data/lavern.db`) and audit logs (`/app/audit-logs`)
- **Resource Profile:** medium (1Gi RAM, 500m CPU) — Node.js server with native module compilation (better-sqlite3) and potential document processing (pdf-parse, mammoth)
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: Health Check
- **Description:** Verify the Fastify server is running and the /health endpoint responds
- **Type:** http
- **Input:** GET /health
- **Expected:** Returns 200 OK indicating the server is healthy. The existing Dockerfile already includes a HEALTHCHECK using this endpoint.
- **Timeout:** 60 seconds (includes container start-up with native module loading)

### Scenario 2: Capabilities Endpoint
- **Description:** Verify the /api/capabilities endpoint returns the system's agent and workflow capabilities
- **Type:** http
- **Input:** GET /api/capabilities
- **Expected:** Returns 200 OK with JSON listing available agents, workflows, and inference providers
- **Timeout:** 30 seconds

### Scenario 3: Agents List
- **Description:** Verify the /api/agents endpoint returns the full roster of 67 agent definitions
- **Type:** http
- **Input:** GET /api/agents
- **Expected:** Returns 200 OK with a JSON array of agent definitions. The array should contain entries for specialists, orchestrators, and other roles.
- **Timeout:** 30 seconds

### Scenario 4: Well-Known / OpenAPI Metadata
- **Description:** Verify the well-known metadata endpoint responds (the project has a `well-known.ts` routes file)
- **Type:** http
- **Input:** GET /.well-known/ai-plugin.json
- **Expected:** Returns 200 OK with JSON plugin metadata, or a structured error response. Either outcome confirms the routing layer is functional.
- **Timeout:** 15 seconds

### Scenario 5: Frontend Dashboard
- **Description:** Verify the bundled Vite frontend dashboard is served at the root path
- **Type:** http
- **Input:** GET /
- **Expected:** Returns 200 OK with HTML content (the built dashboard from `viz/dist`)
- **Timeout:** 30 seconds

## Dockerfile Considerations

The project already includes a well-structured multi-stage Dockerfile. Key points for the containerize agent:

- **USE the existing Dockerfile as-is or with minimal modifications.** It already handles:
  - Frontend build stage (node:20-slim, `npm run build` in `viz/`)
  - API build stage (TypeScript source copy)
  - Runtime stage with native module build dependencies (python3, make, g++ for better-sqlite3)
  - Production dependency installation with `--omit=dev`
  - SQLite data directory creation (`/app/data`, `/app/audit-logs`)
  - Proper environment defaults
  - HEALTHCHECK directive
  - EXPOSE 3000

- **This is a long-running Fastify web server.** The ENTRYPOINT/CMD should be `npx tsx src/index.ts --serve` (as in the existing Dockerfile's `CMD ["npx", "tsx", "src/index.ts", "--serve"]`).
- **EXPOSE 3000** is correct — the server listens on port 3000.
- **The `.npmrc` file is important** — it contains `legacy-peer-deps=true` to resolve peer dependency conflicts between zod@^4 and openai@^4. Make sure it's copied.
- **SOUL.md must be copied** — the existing Dockerfile copies it; it's likely used by agent prompts at runtime.
- **Puppeteer may need additional system dependencies** — but for the PoC, document rendering features (PDF generation) are secondary and can be skipped. The existing Dockerfile doesn't install Chromium, which means Puppeteer features will degrade gracefully.

## Deployment Considerations

- **Deploy as a Kubernetes Deployment** with 1 replica. This is a long-running Fastify server that listens on port 3000.
- **Create a Service** on port 3000 pointing to the Deployment.
- **Create a PVC** (1Gi) mounted at `/app/data` for SQLite database persistence. Optionally, a second PVC or the same one can serve `/app/audit-logs`.
- **Environment variables:** The most critical is `ANTHROPIC_API_KEY` (required for real LLM-powered engagements). For the PoC, the system should start in a demo/local mode if no API key is provided — the README states "Demo mode runs the dashboard, the Clawern view, and the cinematic guided tour without an API key." The test scenarios (health, capabilities, agents list, frontend) do not require actual LLM calls.
- **Set `SHEM_HOST=0.0.0.0`** so Fastify binds to all interfaces (already in the Dockerfile ENV defaults).
- **Set `SHEM_PORT=3000`** (already default).
- **Set `NODE_ENV=production`** (already default in Dockerfile).
- **Test via HTTP requests** to the Service endpoint. All five scenarios use standard HTTP GET requests.
- **No Ingress/Route is strictly required** for PoC validation — test via cluster-internal Service or port-forward. If exposed externally, set `SHEM_CORS_ORIGINS` appropriately.
- **The docker-compose.yml provides a good reference** for volume mounts and environment variable configuration.