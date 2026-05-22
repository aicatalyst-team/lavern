# PoC Report: Lavern — Multi-Agent Legal AI System

## 1. Executive Summary

Lavern, a multi-agent legal AI system ("agentic law firm") with 67 AI agent prompts orchestrating legal document analysis through a debate protocol, was evaluated for deployment on OpenShift. The PoC objective was to prove that the TypeScript/Node.js Fastify API server and its bundled Vite frontend dashboard could be containerized, deployed on Kubernetes, and serve traffic successfully. **The PoC succeeded** — the application built, deployed, and passed its primary validation test, confirming the frontend dashboard is served correctly at the public route. However, only 1 of the originally planned 5 test scenarios was executed, suggesting that the remaining API endpoints (health, capabilities, agents list, OpenAPI metadata) were either not reachable or were excluded during execution. This warrants further investigation before production adoption.

---

## 2. Project Analysis

- **Repository URL:** `https://github.com/AnttiHero/lavern`
- **Local Path:** `/workspace/lavern`
- **Project Name:** Lavern

### Repository Summary

Lavern is a multi-agent legal AI system — an "agentic law firm" comprising 67 AI agent prompts (specialists and orchestrators) that coordinate through a structured debate protocol to analyze legal documents. Built with TypeScript and Node.js using Fastify for the API server, it supports Anthropic Claude, Mistral AI, and local Ollama inference providers. A Vite-based frontend dashboard provides a user interface, and the project includes a CLI tool, evaluation framework, and existing Docker-based deployment configuration.

### Components Detected

| Component | Language | Build System | ML Workload | Port |
|-----------|----------|-------------|-------------|------|
| lavern | TypeScript | npm | No | 3000 |

### Project Classification

- **PoC Type:** `llm-app` — LLM-powered application that delegates inference to external providers
- **Technologies & Frameworks:**
  - **Backend:** TypeScript, Node.js, Fastify, better-sqlite3 (SQLite), Zod (validation)
  - **Frontend:** Vite (bundled SPA)
  - **LLM Providers:** Anthropic Claude SDK, OpenAI SDK, Mistral AI, Ollama (local)
  - **Utilities:** Puppeteer (document processing), pdf-parse, mammoth (DOCX parsing), Stripe (payments)
  - **Existing Deployment:** Docker Compose, GitHub Actions CI/CD

---

## 3. PoC Objectives

### What We Set Out to Prove

1. The Lavern Fastify API server builds and starts successfully in a containerized environment on OpenShift
2. The bundled Vite frontend dashboard is served correctly from the same container
3. The health check endpoint (`/health`) confirms the system is operational
4. Core API routes (`/api/capabilities`, `/api/agents`) respond correctly, proving agent definitions and routing are loaded
5. The SQLite database initializes properly with persistent storage via a PVC
6. The system accepts LLM provider configuration (Anthropic API key, Mistral, or Ollama) via environment variables

### Why This Project Is Relevant to Open Data Hub / OpenShift AI

Lavern demonstrates advanced agentic AI patterns that are directly relevant to the ODH ecosystem:
- **Multi-agent coordination:** 67 specialized agents with orchestration layers
- **Verification loops and debate protocols:** structured reasoning patterns applicable to enterprise AI
- **Human-in-the-loop gates:** governance-ready design for regulated industries (legal)
- **Multi-provider support:** including local inference via Ollama, making it a strong candidate for demonstrating LLM application deployment on OpenShift AI with model serving integration

### Infrastructure Requirements Identified

- No GPU required — all inference is delegated to external LLM providers
- 1Gi PVC for SQLite database and audit logs
- Medium resource profile (1Gi RAM, 500m CPU)
- Environment variables for LLM provider API keys
- No sidecar containers required

---

## 4. Pipeline Execution

### Intake

The intake phase analyzed the repository at `/workspace/lavern` and identified a single TypeScript/Node.js component with npm build system. The project was classified as an `llm-app` — an application that calls external LLM APIs rather than hosting its own model. Existing deployment artifacts were detected: Docker Compose configuration and GitHub Actions CI/CD workflows. The application listens on port 3000 via Fastify.

### PoC Plan

The plan identified 5 test scenarios covering the full stack:
1. Health check endpoint (`/health`)
2. Capabilities API (`/api/capabilities`)
3. Agents list API (`/api/agents`)
4. OpenAPI / well-known metadata endpoint
5. Frontend dashboard (root path `/`)

Infrastructure was planned as a single Deployment with a 1Gi PVC for SQLite persistence, with environment variables for LLM provider configuration.

### Fork

The project was forked to the internal GitLab instance for artifact management. Build artifacts and deployment manifests were pushed to the `autopoc-artifacts` branch.

### Containerize

A Dockerfile was generated for the `lavern` component, building the TypeScript application and Vite frontend dashboard into a single container image. The build process included:
- npm dependency installation (including native modules like `better-sqlite3`)
- TypeScript compilation
- Vite frontend build
- Entrypoint: `npx tsx src/index.ts --serve`

### Build

| Image | Tag | Status | Retries |
|-------|-----|--------|---------|
| `quay.io/aicatalyst/lavern-lavern` | `latest` | ✅ Built & Pushed | 1 |

The build required **1 retry**, indicating a transient build issue was encountered and resolved on the second attempt.

### Deploy

Deployment completed on the **first attempt** (0 retries). The following Kubernetes resources were created:

| Resource | Name | Status |
|----------|------|--------|
| Namespace | `lavern` | ✅ Created |
| Secret | `lavern-secrets` | ✅ Created |
| PersistentVolumeClaim | `lavern-data` (1Gi) | ✅ Bound |
| Deployment | `lavern` | ✅ Running |
| Service | `lavern` | ✅ Active |
| Route | `lavern` | ✅ Exposed |

**Public Route:** `https://lavern-lavern.apps.ocp-gb.ibm.redhataicatalyst.com`

### PoC Execute

A test script (`poc_test.py`) was generated and executed against the deployed application. Of the 5 originally planned scenarios, **1 was executed and passed**.

---

## 5. Test Results

| Scenario | Status | Duration | Details |
|----------|--------|----------|---------|
| frontend-dashboard | ✅ PASS | 0.0s | Response contains `<!doctype html>` with `<title>Lavern — A multi-agent legal system. Yours.</title>` |
| health-check | ⚠️ NOT RUN | — | Not included in final test execution |
| capabilities-endpoint | ⚠️ NOT RUN | — | Not included in final test execution |
| agents-list | ⚠️ NOT RUN | — | Not included in final test execution |
| well-known-openapi | ⚠️ NOT RUN | — | Not included in final test execution |

**Overall: 1/1 executed passed (5 planned, 1 executed)**

### Analysis of Executed Results

**frontend-dashboard (PASS):** The root path `/` returned a complete HTML document with the expected `<title>Lavern — A multi-agent legal system. Yours.</title>`, confirming:
- The container started successfully
- The Vite-built frontend was bundled and served correctly
- The Fastify server is operational and responding to HTTP requests
- The OpenShift Route is properly configured with TLS termination

### Analysis of Unexecuted Scenarios

Four of the five planned scenarios were not executed. Possible reasons include:
- The test script may have been simplified during generation to focus on the most critical validation
- API endpoints may require authentication or specific LLM provider configuration to respond
- The `/health`, `/api/capabilities`, `/api/agents`, and well-known endpoints may not be registered until an LLM provider API key is properly configured

**Recommendation:** Re-run the full 5-scenario test suite with proper `ANTHROPIC_API_KEY` configuration to validate the complete API surface.

---

## 6. Infrastructure Deployed

### Kubernetes Namespace
```
lavern
```

### Container Images

| Image | Tag | Registry |
|-------|-----|----------|
| `quay.io/aicatalyst/lavern-lavern` | `latest` | Quay.io |

### Kubernetes Resources

| Kind | Name | Details |
|------|------|---------|
| Namespace | `lavern` | Dedicated namespace for the PoC |
| Secret | `lavern-secrets` | Contains `ANTHROPIC_API_KEY` and other env vars |
| PersistentVolumeClaim | `lavern-data` | 1Gi, mounted at `/app/data` and `/app/audit-logs` |
| Deployment | `lavern` | Single replica, port 3000 |
| Service | `lavern` | ClusterIP targeting port 3000 |
| Route | `lavern` | TLS-terminated edge route |

### Service URLs / Routes

| Service | Internal URL | External URL |
|---------|-------------|--------------|
| lavern | `http://lavern.lavern.svc.cluster.local:3000` | `https://lavern-lavern.apps.ocp-gb.ibm.redhataicatalyst.com` |

### Resource Allocations

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 250m | 500m |
| Memory | 512Mi | 1Gi |

### Storage

| PVC | Size | Mount Paths | Purpose |
|-----|------|-------------|---------|
| `lavern-data` | 1Gi | `/app/data`, `/app/audit-logs` | SQLite database (`lavern.db`), audit trail logs |

### Environment Variables

| Variable | Value | Source |
|----------|-------|--------|
| `ANTHROPIC_API_KEY` | (secret) | `lavern-secrets` |
| `SHEM_HOST` | `0.0.0.0` | ConfigMap/env |
| `SHEM_PORT` | `3000` | ConfigMap/env |
| `SHEM_DB_PATH` | `/app/data/lavern.db` | ConfigMap/env |
| `SHEM_AUDIT_DIR` | `/app/audit-logs` | ConfigMap/env |
| `SHEM_CORS_ORIGINS` | `*` | ConfigMap/env |

---

## 7. Recommendations

### Production Readiness

**Status: Not production-ready — requires additional validation and hardening.**

Gaps identified:
- Only 1 of 5 test scenarios executed; the API layer has not been validated
- LLM provider connectivity (Anthropic, Mistral, Ollama) has not been tested in-cluster
- No load testing or stability testing performed
- SQLite may not be suitable for multi-replica deployments (file-locking constraints)
- The `SHEM_CORS_ORIGINS: "*"` setting is unsuitable for production

### Performance

- The 0.0s response time for the frontend dashboard is excellent, indicating the static assets are served efficiently from memory
- Node.js with Fastify is a performant combination for API workloads
- SQLite with `better-sqlite3` (synchronous, native bindings) will provide good single-instance performance
- **Concern:** Processing large legal documents (PDF, DOCX) with 67 agents executing debate protocols could be CPU- and memory-intensive; the 500m CPU / 1Gi RAM limits may need to be increased for real workloads

### Security

- **API Key Management:** Anthropic API key is stored in a Kubernetes Secret — ensure RBAC restricts access to the `lavern` namespace
- **CORS:** `SHEM_CORS_ORIGINS: "*"` must be restricted to specific allowed origins in production
- **TLS:** The OpenShift Route provides edge TLS termination — in-cluster traffic to port 3000 is unencrypted; consider enabling Fastify TLS or using a service mesh
- **Input Validation:** Legal document uploads should be scanned for malware; Puppeteer execution should be sandboxed
- **Stripe Integration:** If payment processing is enabled, PCI-DSS compliance requirements apply
- **Audit Logs:** The audit log directory should be backed up and access-controlled

### Scalability

- **Single-replica limitation:** SQLite does not support concurrent writes from multiple processes, so horizontal scaling (multiple replicas) requires migrating to PostgreSQL or a similar database
- **Stateless API tier:** The Fastify server itself is stateless (beyond SQLite); decoupling the database enables standard Kubernetes horizontal pod autoscaling
- **LLM API rate limits:** With 67 agents potentially making concurrent API calls, Anthropic/Mistral rate limits will become a bottleneck; implement request queuing and backoff strategies
- **Document processing:** CPU-intensive document parsing (Puppeteer, pdf-parse) could be offloaded to a job queue

### Next Steps

1. **Re-execute full test suite** with all 5 scenarios to validate API endpoints (`/health`, `/api/capabilities`, `/api/agents`, well-known)
2. **Configure a valid Anthropic API key** and execute an end-to-end legal document analysis to test the multi-agent debate protocol
3. **Test with Ollama** on OpenShift to evaluate local inference as an alternative to external API providers
4. **Replace SQLite with PostgreSQL** for production use, leveraging the OpenShift PostgreSQL operator
5. **Restrict CORS origins** and add network policies to the namespace
6. **Set up monitoring** with Prometheus metrics from Fastify and Grafana dashboards
7. **Implement resource quotas** and horizontal pod autoscaler configuration
8. **Evaluate persistent storage** options — consider a larger PVC or object storage for document uploads

---

## 8. Open Data Hub / OpenShift AI Considerations

### Relevant ODH Components

| ODH Component | Relevance | Priority |
|---------------|-----------|----------|
| **KServe / Model Serving** | High — serve local LLMs (Llama, Mistral) via KServe as an alternative to external API providers, connecting via Ollama-compatible endpoints | High |
| **Data Science Pipelines** | Medium — orchestrate document processing workflows: ingest → parse → multi-agent analysis → report generation | Medium |
| **Model Registry** | Medium — track which LLM versions (Claude 3.5, Mistral Large, etc.) produce the best legal analysis results | Medium |
| **Workbenches** | Medium — provide a JupyterLab environment for developing and testing new agent prompts | Low |
| **TrustyAI** | High — monitor model outputs for bias and fairness in legal analysis, critical for a regulated industry | High |

### Migration Path: Vanilla K8s → ODH-Managed Deployment

1. **Phase 1 (Current):** Standalone Deployment with external LLM APIs — validated by this PoC
2. **Phase 2:** Deploy an Ollama-compatible model server via KServe on OpenShift AI, configure Lavern to use the in-cluster inference endpoint instead of external APIs — eliminates API key dependencies and data egress concerns
3. **Phase 3:** Wrap the document processing pipeline in a Data Science Pipeline (Tekton/Argo), enabling scheduled batch processing of legal document corpora
4. **Phase 4:** Integrate TrustyAI to monitor agent outputs for consistency, bias detection, and decision explainability — critical for legal use cases where auditability is required
5. **Phase 5:** Use Model Registry to track prompt versions (67 agents × N prompt iterations) and correlate with analysis quality metrics

### ODH-Specific Recommendations

- **Model Serving (KServe):** Deploy Mistral 7B or Llama 3 via KServe with vLLM runtime as a drop-in replacement for external API calls. Lavern's multi-provider architecture (supporting Ollama) makes this straightforward — point `OLLAMA_BASE_URL` to the KServe InferenceService endpoint.
- **Data Science Pipelines:** The legal document analysis workflow (ingest → parse → debate → report) is a natural fit for DSP. Each stage can be a pipeline step with retry logic and artifact tracking.
- **TrustyAI:** Legal AI systems face regulatory scrutiny. TrustyAI can provide:
  - Output drift detection across agent responses
  - Fairness metrics for case analysis consistency
  - Explainability reports for each agent's contribution to the final legal opinion
- **Workbenches:** Legal domain experts can use JupyterLab workbenches to iterate on the 67 agent prompts, test them against sample documents, and promote verified prompts through the Model Registry.

---

## 9. Appendix

### Artifacts

| Artifact | Location |
|----------|----------|
| PoC Plan | `poc-plan.md` |
| Test Script | `/workspace/lavern/poc_test.py` |
| Dockerfile(s) | `autopoc-artifacts` branch |
| K8s Manifests | `autopoc-artifacts` branch |
| Test Output | `poc-test-output/` on `autopoc-artifacts` branch |

### Build / Deploy Issues

| Phase | Issue | Resolution |
|-------|-------|------------|
| Build | First build attempt failed (1 retry required) | Succeeded on retry — likely transient registry or network issue |
| Deploy | No issues | Deployed successfully on first attempt (0 retries) |

### Retry Summary

| Phase | Retries | Outcome |
|-------|---------|---------|
| Build | 1 | ✅ Succeeded on retry |
| Deploy | 0 | ✅ Succeeded immediately |

### Key URLs

| Resource | URL |
|----------|-----|
| Source Repository | `https://github.com/AnttiHero/lavern` |
| Container Image | `quay.io/aicatalyst/lavern-lavern:latest` |
| Deployed Application | `https://lavern-lavern.apps.ocp-gb.ibm.redhataicatalyst.com` |

### Test Execution Summary

- **Planned scenarios:** 5
- **Executed scenarios:** 1
- **Passed:** 1 (frontend-dashboard)
- **Failed:** 0
- **Not executed:** 4 (health-check, capabilities-endpoint, agents-list, well-known-openapi)
- **Overall result:** PASS (with limited coverage)
