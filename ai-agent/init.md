You are a **senior software architect and DevOps engineer**.
Design and implement a **production-ready MCP-based project** called **`ai-toolbox`**.

Your main goal:  
Build a repository that provides **three MCP servers**, all orchestrated with **podman-compose** and ready to mount
external project directories:

1. `context7` – provides access to documentation.
2. `playwright` – automates web testing.
3. `memory-bank` – stores AI rules, project settings, and global memory via the MCP protocol.

Longer introduction:

`ai-toolbox` is a production-ready toolkit of **Model Context Protocol (MCP)** servers designed to supercharge
AI-assisted development workflows. It bundles three specialized services, all containerized and orchestrated via
**podman-compose**, with first-class support for mounting your existing projects and data from the host:

- **`context7`** – Documentation and context server that indexes and serves project and external docs,
  enabling AIs to search, browse, and retrieve precise snippets from mounted documentation directories.
- **`playwright`** – Web-testing automation server powered by **Playwright**, capable of generating, running, and
  inspecting end-to-end tests against web apps hosted in mounted project directories.
- **`memory-bank`** – Persistent global memory server backed by **SQLite**, used to store AI rules, per-project
  settings, and long-term context via a simple MCP interface.

Each server is implemented with modern, maintainable patterns, featuring structured logging, robust error handling,
configurable environment-based settings, and examples of test coverage. `ai-toolbox` is intended as a plug-and-play
infrastructure layer for advanced AI agents and IDE integrations, making it easy to connect your projects,
documentation, tests, and long-term memory into a cohesive, production-grade AI environment.

---

## 1. High-level goals

- Build a **single repo** named `ai-toolbox` with:
    - Three MCP servers (`context7`, `playwright`, `memory-bank`).
    - `podman-compose` configuration to run all servers together.
    - Clear support for **mounting external project directories** (code, docs, config) into the containers.
- The implementation must be **production-ready**:
    - Robust error handling.
    - Logging and observability.
    - Secure configuration and secrets handling.
    - Reasonable test coverage.
    - Documentation for setup and usage.

Use **TypeScript/Node.js** for the MCP servers unless there is a strong reason to prefer another language. Use modern,
maintainable patterns.

---

## 2. Repository structure

Propose and then implement a structure similar to (you can refine as needed):

- `servers/`
    - `context7/`
        - `src/` (MCP server implementation)
        - `Dockerfile`
        - `package.json`, `tsconfig.json`
    - `playwright/`
        - `src/`
        - `Dockerfile`
        - `package.json`, `tsconfig.json`
    - `memory-bank/`
        - `src/`
        - `Dockerfile`
        - `package.json`, `tsconfig.json`
- `infra/`
    - `podman-compose.yml`
    - `env/`
        - `.env.example`
- `docs/`
    - `README.md`
    - `MCP_SERVERS.md`
    - `DEPLOYMENT.md`

You may adjust the tree, but keep it **clear, modular, and production-oriented**.

---

## 3. MCP servers – functional requirements

## 3.0 MCP protocol & transport (MANDATORY)

These MCP servers MUST be **networked** services that clients can connect to over the container network.

- Each server MUST expose:
    - **MCP Streamable HTTP transport** on a configurable port.
    - MCP endpoints at ` /mcp`:
        - `POST /mcp` (request/response)
        - `GET /mcp` (SSE stream for notifications; required if SSE is enabled)
        - `DELETE /mcp` (session termination)
    - **Health endpoint**: `GET /healthz` returning `200` when ready.
- Each server SHOULD support optional authentication for `/mcp`:
    - If `MCP_AUTH_TOKEN` is set, require `Authorization: Bearer <token>` for all `/mcp` methods.
    - If unset, run without auth but avoid publishing the port to the host by default.
- Prefer implementing the network transport using the **official MCP SDK for Node.js/TypeScript** and its supported
  **Streamable HTTP** transport.
- The Compose setup MUST wire:
    - container ports (internal) for service-to-service connectivity
    - optional host port publishing for local development
- Each server MUST log:
    - one JSON log line per request/tool call (structured logging)
    - `request_id` (if provided) or a generated correlation id

You may expose an additional `GET /version` endpoint for ops/debug.

## 3.0.1 Cross-cutting tool contract conventions (MANDATORY)

All MCP tools across all servers MUST follow these conventions:

- **Project selection**
    - Any tool that touches mounted host paths MUST accept either:
        - `project_id` (string), mapped to a configured mount root, OR
        - `root` (string), selected from a server-provided allowlisted set.
    - The server MUST reject unknown `project_id`/`root` with a clear error.
- **Pagination**
    - For list/search tools, support `limit` (default 20, max 100) and `cursor` (opaque string).
- **Errors**
    - Use a consistent error shape:
      `{ code: string, message: string, details?: object, retryable?: boolean }`.
    - Do not leak secrets or host filesystem details in `details`.
- **Limits**
    - Enforce configurable maximums (content size, snippet size, search results, timeouts).
    - Return a clear error when limits are exceeded.

Default recommended limits (overrideable via env):

- `MAX_REQUEST_BYTES` default `1_000_000`
- `MAX_TOOL_RUNTIME_MS` default `30_000`
- `MAX_SEARCH_RESULTS` default `50`
- `MAX_SNIPPET_LINES` default `200`
- `MAX_MEMORY_CONTENT_BYTES` default `200_000`
- `PLAYWRIGHT_TIMEOUT_MS` default `300_000`
- `PLAYWRIGHT_MAX_WORKERS` default `2`

### 3.1 `context7` MCP server (documentation provider)

**Purpose:** Provide AI-accessible documentation from **local mounted directories** and (optionally) **remote sources**.

**Requirements:**

- Implement as an MCP server exposing tools such as (names can be refined):
    - `list_doc_sources` – list available documentation sources (local paths and configured remote docs).
    - `search_docs` – full-text search within mounted documentation (e.g., Markdown, text) with:
        - query string
        - optional filters by project / path
    - `get_doc_snippet` – retrieve specific file sections by path + line range or heading.
- Support **mounted local directories**:
    - Read-only access to docs mounted from the host (e.g., `/projects/<project>/docs`).
    - Paths controlled via environment variables and/or configuration files.
- Remote documentation sources MUST be **opt-in via allowlist**:
    - Provide an allowlist configuration (environment variable and/or config file) of permitted base URLs.
    - Reject any remote fetch not matching the allowlist.
    - Enforce timeouts and response size limits.
    - Provide clear error messages when offline or blocked by policy.
- Robust safety:
    - Do not allow arbitrary path traversal outside configured mount roots.
    - Clear validation and sanitization of file paths.

### 3.2 `playwright` MCP server (web testing automation)

**Purpose:** Provide tools to **generate, run, and inspect Playwright-based web tests** for externally mounted projects.

**Requirements:**

- Use **Playwright** with Node.js in a container-friendly setup.
- Expose tools like:
    - `generate_test` – generate a basic Playwright test file for:
        - target URL
        - test description / scenario
        - output path inside the mounted project tests directory.
    - `run_tests` – run tests in a specified directory (e.g., `tests/e2e`) and return:
        - summary (passed/failed, duration)
        - concise logs or links to log files.
    - `inspect_page` (optional) – open a page and capture metadata (title, key elements, etc.).
- All project code under test must be **mounted from the host** into the container:
    - Example: host project at `/home/user/my-app` mounted to `/projects/my-app` in the container.
    - Tests should read/write only within those mounted paths.
- Container should include:
    - Necessary Playwright browsers installed.
    - Proper non-root user.
    - Easy way to extend config for new host projects (via env and volumes).

### 3.3 `memory-bank` MCP server (global AI memory)

**Purpose:** Provide a **persistent, queryable memory store** via MCP to keep:

- AI rules and conventions.
- Per-project settings.
- Cross-project/global memory entries.

**Requirements:**

- Backed by a **persistent store**:
    - Prefer a simple **SQLite** database stored on a mounted volume (e.g., `/data/memory-bank.db`).
- Expose tools like:
    - `create_or_update_memory` – store or update a memory record with:
        - `id` (optional, for update)
        - `type` (rule, setting, context, note, etc.)
        - `scope` (global, project, user, etc.)
        - `tags` (list of strings)
        - `content` (text/json blob)
    - `get_memory` – fetch by ID.
    - `search_memories` – filter by `scope`, `type`, `tags`, and/or full-text content match.
    - `delete_memory` – remove by ID.
    - `list_memories` – paginate results.
- Design a **simple, normalized schema** (`memories` table, indices on tags/scope where useful).
- Ensure:
    - Input validation.
    - Basic rate limiting or safe guards against huge payloads (e.g., max content size).
    - Concurrency-safe DB access.

---

## 4. Podman Compose & containerization

Use **podman-compose** (compose v3 syntax compatible with Docker Compose).

**Requirements:**

- A top-level `infra/podman-compose.yml` that defines at least:

    - `context7` service
    - `playwright` service
    - `memory-bank` service
    - shared network
    - named volumes (for persistent data and local projects, if appropriate)

- Each service should:
    - Build from its own `Dockerfile` under `servers/<name>/Dockerfile`.
    - Run a non-root user.
    - Define **healthchecks** using `GET /healthz`.
    - Read configuration from environment variables (overrideable via `.env` and/or CLI).
    - Log to stdout/stderr in a **structured, parseable format** (JSON logs preferred).

- **External directory mounting**:

    - Show how to mount host project directories into each service, for example:
        - `./examples/project1` on host -> `/projects/project1` in `context7` and `playwright`.
    - Show how to mount a persistent data directory for `memory-bank`, e.g.:
        - `./data/memory-bank` on host -> `/data` in container.
    - Provide **commented examples** in `podman-compose.yml` demonstrating:
        - Single project setup.
        - Multiple project mounts.
        - How to enable/disable specific mounts.

- Include an `.env.example` file under `infra/env/` with variables like:
    - `CONTEXT7_PORT=7001`
    - `PLAYWRIGHT_PORT=7002`
    - `MEMORY_BANK_PORT=7003`
    - `CONTEXT7_DOCS_ROOTS=/projects/project1/docs:/projects/project2/docs`
    - `CONTEXT7_REMOTE_ALLOWLIST=https://docs.example.com,https://developer.mozilla.org`
    - `PLAYWRIGHT_PROJECT_ROOTS=/projects/project1:/projects/project2`
    - `MEMORY_BANK_DB_PATH=/data/memory-bank.db`
    - And any needed MCP configuration variables (ports, log levels, etc.).

---

## 5. Production readiness

Design and implement with **production use** in mind:

- **Baseline choices (MANDATORY)**
    - Node.js: use an LTS release (recommend Node 20).
    - TypeScript: `strict` enabled.
    - Validation: use a schema validator (e.g., Zod) for all tool inputs.
    - Logging: JSON structured logging.
    - Testing: a modern test runner (e.g., Vitest/Jest) plus at least one integration test per server.

- **Configuration & security**
    - All secrets and sensitive values come from environment variables or mounted secrets, not hardcoded.
    - Sensible defaults but safe-by-default behavior.
    - No broad host filesystem access; only explicitly configured mount points.
- **Logging & observability**
    - Structured logs (JSON) with correlation IDs where relevant.
    - At least minimal metrics hooks or clear extension points (even if just documented).
- **Error handling**
    - Clear error messages with user-facing detail but no sensitive leakage.
    - Consistent error schema for MCP tools.
- **Testing**
    - Unit tests for core logic in each server.
    - At least one integration test per server that:
        - Starts the server (or mocks its entrypoint).
        - Exercises a key tool end-to-end.
    - Instructions in the README for running tests.

---

## 6. Documentation & developer experience

Produce clear, concise documentation:

- Top-level `README.md`:
    - Overview of `ai-toolbox`.
    - Short descriptions of `context7`, `playwright`, `memory-bank`.
    - Quickstart: prerequisites, how to build and run via `podman-compose`.
- `MCP_SERVERS.md`:
    - For each server:
        - Purpose.
        - List of tools with input/output schemas and example invocations.
- `DEPLOYMENT.md`:
    - How to deploy to:
        - Local dev environment.
        - A generic production host (e.g., a VM or server) using Podman.
    - How to configure and mount external project directories.
    - How to perform backups/restores for `memory-bank`.

Include comments in configuration files where it aids understanding, but keep them concise and practical.

---

## 7. Output format

When you respond, structure your answer as follows:

1. **Architecture overview**
    - Short description of each MCP server and how they interact.
    - High-level explanation of podman-compose setup and mounts.

2. **Repository tree**
    - A directory tree of the `ai-toolbox` project.

3. **Files (FULL CONTENTS, REQUIRED)**
    - Provide the **full contents** of every created or modified file, each preceded by its relative path.
    - Do not summarize key files; include them completely so the repository is runnable.

4. **How to run**
    - Step-by-step instructions for:
        - Setting environment variables.
        - Building images.
        - Starting with `podman-compose`.
        - Mounting example external project directories.
        - Testing that each MCP server works.

---

## 8. Self-check before final answer

Before finalizing your response, **explicitly verify** that you have:

- Implemented all three MCP servers: `context7`, `playwright`, `memory-bank`.
- Ensured all three are runnable via **podman-compose**.
- Demonstrated how to **mount external project directories** for:
    - docs (`context7`)
    - code/tests (`playwright`)
    - persistent memory storage (`memory-bank`)
- Included:
    - Production-appropriate Dockerfiles.
    - A `podman-compose.yml` with healthchecks and env configuration.
    - At least basic test coverage examples.
    - Clear documentation (`README.md`, etc.).
- Addressed **error handling**, **logging**, and **configuration** for production.

If anything is missing, **add it** before returning your final answer.