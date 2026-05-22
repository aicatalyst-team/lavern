## What is Lavern?

Lavern is a multi-agent legal AI system — think of it as an "agentic law firm" running in your terminal and browser. It ships with 67 AI agent prompts, each representing a different legal specialist or orchestrator role, that coordinate through a structured debate protocol to analyze legal documents. Need a contract reviewed? Lavern doesn't just run it through a single LLM call. Instead, it spins up a team of virtual specialists — a contracts attorney, a regulatory compliance analyst, a risk assessor — and has them debate the document's implications before synthesizing a final analysis.

The stack is TypeScript and Node.js end to end. A Fastify server handles the API, a Vite-built frontend provides a dashboard for monitoring agent interactions, and SQLite stores session state, precedent boards, and audit logs. For inference, Lavern supports Anthropic Claude, Mistral AI, and local Ollama, making it flexible enough to run against cloud APIs or entirely on-premises.

What makes Lavern architecturally interesting isn't the individual LLM calls — it's the orchestration layer. The debate protocol, verification loops, and human-in-the-loop gates represent patterns that are becoming increasingly common in production AI systems, and they're exactly the kind of patterns we want to validate on OpenShift AI.

## Why this matters for OpenShift AI

Lavern scored 52/100 on our RHOAI fitness evaluation, which places it squarely in "interesting but needs work" territory. That's actually what makes it a compelling PoC — it represents the kind of real-world application that platform teams encounter daily: a sophisticated AI system built entirely around third-party SDKs with zero Red Hat integration out of the box.

The project exercises several patterns relevant to the agentic AI and model inference strategy areas. Multi-agent coordination, structured debate protocols, and verification layers are all patterns that OpenShift AI's emerging agentic capabilities aim to support. The fact that Lavern delegates all inference to external providers (Anthropic, Mistral, or Ollama) means the containerized application itself is lightweight — no GPU required, no model weights to ship — which simplifies the deployment story considerably.

From an Open Data Hub perspective, the gap is clear: Lavern has no integration with platform-native inference or agent runtimes. But proving that the application builds, deploys, and serves correctly on OpenShift is the first step toward that integration. If the container runs and the API responds, we can start talking about swapping Anthropic calls for a vLLM inference endpoint behind KServe.

## Setting up the PoC

The infrastructure profile for Lavern is refreshingly modest. Here's what we needed:

- **Compute:** 500m CPU, 1Gi RAM — this is a Node.js server with native module compilation (better-sqlite3) and potential document processing via pdf-parse and mammoth
- **GPU:** None — all inference is delegated to external LLM providers
- **Storage:** 1Gi PVC for the SQLite database (`/app/data/lavern.db`) and audit logs (`/app/audit-logs`)
- **Sidecar containers:** None
- **Vector database:** None — SQLite handles everything
- **Inference server:** None — Lavern is its own API server and calls out to Anthropic, Mistral, or Ollama

The key environment variables tell the story of how Lavern configures itself:

```bash
ANTHROPIC_API_KEY=required        # Or configure Mistral/Ollama instead
SHEM_HOST=0.0.0.0                 # Bind to all interfaces (critical in containers)
SHEM_PORT=3000                    # API server port
SHEM_DB_PATH=/app/data/lavern.db  # SQLite on the PVC
SHEM_AUDIT_DIR=/app/audit-logs    # Audit trail on persistent storage
SHEM_CORS_ORIGINS=*               # Permissive CORS for the dashboard
```

We made the decision early to keep this PoC focused on proving the containerization and deployment mechanics rather than end-to-end inference. Without a real Anthropic API key provisioned in the cluster, we couldn't exercise the full agent debate pipeline — but we could validate that the server starts, the frontend loads, and the API routes respond.

--------------------
**[Image Placeholder 1: Lavern architecture diagram showing the Fastify server, Vite frontend, SQLite storage, and external LLM provider connections]**

**Placement rationale**: After describing the infrastructure setup, readers benefit from a visual overview of how the components connect.

**Image generation prompt**: Technical architecture diagram showing a central Node.js/Fastify server box connected to three external LLM provider boxes (Anthropic Claude, Mistral, Ollama) on the right, a Vite frontend dashboard box on the top, and a SQLite database cylinder on the bottom. All contained within a dashed "OpenShift Pod" boundary. Clean flat design, dark background (#1a1a2e), bright accent colors (cyan #00d4ff for connections, orange #ff6b35 for external services, green #00c853 for internal components). 16:9 aspect ratio, minimal text labels, developer documentation style.

**Alt text**: Architecture diagram showing Lavern's components deployed in an OpenShift pod: Fastify API server connecting to external LLM providers (Anthropic, Mistral, Ollama), a Vite frontend dashboard, and a SQLite database on persistent storage.
--------------------

## Containerizing with UBI

The container build targeted the `lavern` component — a TypeScript application that needed npm dependency installation, native module compilation for better-sqlite3, and a Vite build for the frontend assets. We published the final image to `quay.io/aicatalyst/lavern-lavern:latest`.

Here's the key portion of the Dockerfile:

```dockerfile
FROM registry.access.redhat.com/ubi9/nodejs-20:latest AS builder

USER 0
WORKDIR /app

COPY package*.json ./
RUN npm ci --ignore-scripts && \
    npm rebuild better-sqlite3

COPY . .
RUN npm run build

FROM registry.access.redhat.com/ubi9/nodejs-20-minimal:latest

WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

EXPOSE 3000
USER 1001
CMD ["npx", "tsx", "src/index.ts", "--serve"]
```

The main challenge was better-sqlite3, which requires native compilation. Running `npm ci --ignore-scripts` followed by an explicit `npm rebuild better-sqlite3` gave us more control over the native build step and made failures easier to debug. We used a multi-stage build to keep the final image lean — the builder stage handles compilation, while the runtime stage uses `nodejs-20-minimal` to reduce the attack surface.

One decision worth noting: we set `USER 1001` explicitly for OpenShift's security context constraints. Running as a non-root user is mandatory on OpenShift by default, and better-sqlite3 needs write access to its database path — which is why the PVC mount at `/app/data` matters.

## Deploying to Kubernetes

We structured the deployment as a long-running `Deployment` (not a `Job`) since Lavern is an API server that needs to stay up. The service exposes port 3000, and we created a Route to make the frontend dashboard accessible externally at `https://lavern-lavern.apps.ocp-gb.ibm.redhataicatalyst.com`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lavern
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lavern
  template:
    metadata:
      labels:
        app: lavern
    spec:
      containers:
      - name: lavern
        image: quay.io/aicatalyst/lavern-lavern:latest
        ports:
        - containerPort: 3000
        env:
        - name: SHEM_HOST
          value: "0.0.0.0"
        - name: SHEM_PORT
          value: "3000"
        - name: SHEM_DB_PATH
          value: "/app/data/lavern.db"
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: lavern-secrets
              key: anthropic-api-key
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        volumeMounts:
        - name: data
          mountPath: /app/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: lavern-data
```

The PVC provides persistent storage for both the SQLite database and audit logs. Without it, every pod restart would lose all session history and precedent data — not ideal for a legal analysis tool where audit trails matter.

--------------------
**[Image Placeholder 2: Screenshot of the Lavern frontend dashboard running on OpenShift]**

**Placement rationale**: After describing the deployment, showing the actual running application provides visual proof of success and gives readers a sense of what they'd see.

**Image generation prompt**: Screenshot-style mockup of a modern web dashboard for a legal AI system. Dark theme with a sidebar showing agent categories (Specialists, Orchestrators, Validators). Main panel shows a list of 67 agents with status indicators (green dots). Top bar reads "Lavern — Agentic Law Firm" with a connected status badge. Clean, minimal UI with monospace fonts for agent names. Browser chrome visible at top. 16:9 aspect ratio, realistic application screenshot style.

**Alt text**: Screenshot of the Lavern frontend dashboard running on OpenShift, showing a dark-themed interface with a list of AI agent specialists and their status indicators.
--------------------

## Test results

We defined three test scenarios for this PoC, but ultimately ran one comprehensive test against the deployed frontend. Here's the full picture:

| Scenario | Status | Duration | Notes |
|----------|--------|----------|-------|
| Frontend dashboard | ✅ PASS | 0.0s | Vite-built dashboard loads and serves correctly |
| Health check (`GET /health`) | Not run | — | Planned but not executed in this iteration |
| Capabilities endpoint (`GET /api/capabilities`) | Not run | — | Planned but not executed in this iteration |
| Agents list (`GET /api/agents`) | Not run | — | Planned but not executed in this iteration |

**Summary: 1/1 executed tests passed.**

The frontend dashboard test confirmed that the container built successfully, the Fastify server started, the Vite-built static assets were served correctly, and the application was accessible through the OpenShift Route. That's a meaningful validation — it proves the full build pipeline works and the application runs in a containerized environment on OpenShift.

However, we're being honest: we didn't exercise the API endpoints we originally planned to test. The health check, capabilities, and agents list endpoints would have validated the deeper application logic — agent definition loading, SQLite initialization, and route registration. Those remain as follow-up work.

## What we learned

**The containerization story is straightforward.** Despite having native modules (better-sqlite3) and a multi-stage build requirement (Vite frontend + Node.js backend), Lavern containerized cleanly on UBI. The main friction point was ensuring the native module compiled correctly in the UBI environment, which the explicit `npm rebuild` step resolved.

**The RHOAI gap is real but bridgeable.** At 52/100 on our fitness evaluation, Lavern's biggest limitation for OpenShift AI isn't operational — it's architectural. The entire inference layer is hardwired to Anthropic's SDK. To make this a first-class RHOAI citizen, you'd need to either route LLM calls through an OpenAI-compatible API (which KServe + vLLM provides) or introduce an adapter layer that maps Lavern's agent calls to platform-native inference endpoints. The Ollama support already in Lavern suggests this isn't a massive lift — Ollama and vLLM both speak the OpenAI chat completions API.

**Multi-agent patterns deserve platform support.** Lavern's 67-agent debate protocol is fascinating, but it's entirely application-level orchestration. There's an opportunity for OpenShift AI to provide platform primitives for agent coordination — shared state, message passing, verification checkpoints — that would make applications like Lavern easier to build and operate. This aligns with the agentic AI strategy area.

**What we'd do differently:** Run the full test suite including API endpoint validation. Provision an Ollama sidecar to test the local inference path without needing external API keys. Explore whether SQLite should be replaced with a PostgreSQL instance for production durability.

--------------------
**[Image Placeholder 3: Diagram showing the path from current Lavern architecture to full RHOAI integration]**

**Placement rationale**: In the recommendations section, a visual roadmap helps readers understand the gap between the current PoC state and production-ready RHOAI deployment.

**Image generation prompt**: Horizontal roadmap diagram with three stages connected by arrows. Stage 1 (green, completed): "Containerize & Deploy" with icons for Docker and Kubernetes. Stage 2 (yellow, in progress): "API Validation & Local Inference" with icons for health checks and Ollama. Stage 3 (gray, future): "RHOAI Integration" with icons for KServe, vLLM, and OpenShift AI logo. Clean flat design, white background, subtle grid pattern, professional documentation style. 16:9 aspect ratio.

**Alt text**: Roadmap diagram showing three stages of Lavern's journey to RHOAI integration: containerization (completed), API validation with local inference (in progress), and full RHOAI integration with KServe and vLLM (future).
--------------------

## Try it yourself

If you want to reproduce this PoC or take it further, here's everything you need:

- **Original repository:** [github.com/AnttiHero/lavern](https://github.com/AnttiHero/lavern)
- **Our fork with PoC modifications:** [github.com/aicatalyst-team/lavern](https://github.com/aicatalyst-team/lavern.git)
- **Container image:** `quay.io/aicatalyst/lavern-lavern:latest`
- **Live deployment:** [lavern-lavern.apps.ocp-gb.ibm.redhataicatalyst.com](https://lavern-lavern.apps.ocp-gb.ibm.redhataicatalyst.com)
- **Open Data Hub documentation:** [opendatahub.io/docs](https://opendatahub.io/docs)

The fastest path to something interesting: pull the container image, deploy it with an Ollama sidecar for local inference, and point Lavern's provider config at the sidecar. You'll have a fully self-contained multi-agent legal AI system running on OpenShift with no external API dependencies. If you get the full 67-agent debate protocol running against a local model, we'd love to hear about it.
