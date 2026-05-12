# PoC Plan: Hermes Agent

## Project Classification
- **Type:** llm-app
- **Key Technologies:** Python 3.13, OpenAI SDK, httpx, Rich, Pydantic, prompt_toolkit, Playwright, Node.js (for TUI/web components), Docker
- **ODH Relevance:** Hermes Agent is a self-improving AI agent framework that connects to multiple LLM providers and exposes an OpenAI-compatible API server. It can be deployed on OpenShift as a long-running gateway service, demonstrating how agentic LLM applications can run in the ODH/OpenShift AI ecosystem. Its built-in API server makes it a drop-in replacement for model-serving endpoints, and its multi-platform gateway (Telegram, Discord, Slack, etc.) showcases production agent deployment patterns.

## PoC Objectives
What we want to prove:
1. The Hermes Agent container builds and runs successfully on OpenShift using the existing Dockerfile (or a minimal adaptation of it).
2. The built-in OpenAI-compatible API server starts and accepts HTTP requests on a configured port.
3. Chat completion requests flow through the agent to an upstream LLM provider and return valid responses.
4. The `hermes` CLI is functional inside the container (version check, doctor diagnostics).
5. The application can be deployed as a Kubernetes Deployment with a Service exposing the API server port.

## Infrastructure Requirements
- **Inference Server:** none (Hermes has its own built-in API server that proxies to upstream LLM providers)
- **Vector Database:** none (memory features use local SQLite/FTS5, bundled in the container)
- **Embedding Model:** none
- **GPU Required:** no
- **Persistent Storage:** 5Gi PVC mounted at `/opt/data` for Hermes state, conversation history, skills, and memory databases
- **Resource Profile:** medium (1Gi RAM, 500m CPU) — the agent is primarily I/O-bound proxying to LLM APIs, but needs memory for Node.js TUI components and Playwright browser
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: API Server Health
- **Description:** Verify the Hermes API server is running by hitting the OpenAI-compatible `/v1/models` endpoint.
- **Type:** http
- **Input:** GET /v1/models with Authorization header containing the API_SERVER_KEY
- **Expected:** Returns 200 OK with a JSON list of available models
- **Timeout:** 120 seconds (container startup includes npm installs and Playwright setup)

### Scenario 2: Chat Completion
- **Description:** Send a chat completion request to verify end-to-end LLM proxying works.
- **Type:** http
- **Input:** POST /v1/chat/completions with `{"model": "default", "messages": [{"role": "user", "content": "Say hello in exactly three words."}], "max_tokens": 50}`
- **Expected:** Returns 200 with a JSON response containing `choices[0].message.content` with non-empty text
- **Timeout:** 120 seconds

### Scenario 3: Hermes Version Check
- **Description:** Verify the hermes CLI is installed and reports its version.
- **Type:** cli
- **Input:** `hermes --version`
- **Expected:** Job exits 0, outputs version string containing "0.13"
- **Timeout:** 30 seconds

### Scenario 4: Hermes Doctor
- **Description:** Run the built-in diagnostic command to verify internal health checks.
- **Type:** cli
- **Input:** `hermes doctor`
- **Expected:** Job exits 0, outputs diagnostic information about configuration and connectivity
- **Timeout:** 60 seconds

## Dockerfile Considerations

The project already has a comprehensive Dockerfile based on `debian:13.4` that:
- Installs system dependencies (Node.js, npm, ripgrep, ffmpeg, git, tini, etc.)
- Sets up a non-root `hermes` user (UID 10000)
- Installs Python dependencies via `uv` and Node.js dependencies via `npm`
- Installs Playwright browsers for web research capabilities
- Uses `tini` as the init process to reap zombie subprocesses
- Uses `gosu` to drop privileges at runtime via `docker/entrypoint.sh`

**Use the existing Dockerfile as-is or with minimal modifications.** Key points for the containerize agent:
- The existing Dockerfile is production-ready and well-structured with multi-stage caching.
- The ENTRYPOINT is `docker/entrypoint.sh` which handles UID/GID remapping and then executes the gateway via `hermes gateway`.
- The container listens on a port only when `API_SERVER_HOST` and `API_SERVER_KEY` environment variables are set. The default API server port is 3000 (see `hermes_cli/web_server.py` — the gateway's built-in API server).
- Add `EXPOSE 3000` if not already present.
- The container needs `PYTHONUNBUFFERED=1` (already set in the existing Dockerfile).
- The `/opt/data` directory is the data volume mount point for persistent state.

## Deployment Considerations

**Deploy as a Kubernetes Deployment with 1 replica.** The Hermes gateway is a long-running process that:
1. Starts the multi-platform gateway (manages connections to Telegram, Discord, Slack, etc.)
2. Optionally starts the built-in OpenAI-compatible API server on port 3000

**Create a Service** exposing port 3000 for the API server endpoint. This is how tests will interact with the deployment.

**Environment variables required:**
- `OPENAI_API_KEY` (required, secret) — API key for the upstream LLM provider. Hermes uses OpenAI SDK by default but supports many providers via OpenRouter, etc.
- `API_SERVER_KEY` (required, secret) — Authentication key for the built-in API server. Requests must include this as a Bearer token.
- `API_SERVER_HOST` = `0.0.0.0` — Bind the API server to all interfaces (required for Kubernetes Service routing; default is `127.0.0.1`).
- `HERMES_UID` = `10000` — UID for the runtime user (matches the Dockerfile default).
- `HERMES_GID` = `10000` — GID for the runtime user.

**PVC:** Mount a 5Gi PVC at `/opt/data` for persistent state (conversation history, skills database, memory, config).

**Startup time:** The container may take 30-60 seconds to fully start due to Node.js dependency resolution and Playwright browser verification. Set readiness probe initial delay accordingly.

**Security context:** The container starts as root but drops to the `hermes` user (UID 10000) via `gosu` in the entrypoint script. On OpenShift with restricted SCCs, this will need adjustment — either modify the entrypoint to run directly as non-root, or use a SecurityContext with `runAsUser: 10000`.

**HTTP test requests** to the API server must include the `Authorization: Bearer <API_SERVER_KEY>` header. The deploy agent should configure test scenarios to pass this header.

**CLI test scenarios** should use `kubectl run` with the same container image, mounting the same PVC, to run `hermes --version` and `hermes doctor`. These are run-to-completion Jobs.