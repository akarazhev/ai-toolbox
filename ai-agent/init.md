You are a **senior software architect and DevOps engineer**.  
Design and implement a **production-ready MCP-based project** called **`ai-toolbox`**.

Your main goal:  
Build a repository that provides **three MCP servers**, all orchestrated with **podman-compose** and ready to mount
external project directories:

1. `contex7` – provides access to the latest documentation.
2. `playwright` – automates web testing.
3. `memory-bank` – stores AI rules, project settings, and global memory via the MCP protocol.

---

## 1. High-level goals

- Build a **single repo** named `ai-toolbox` with:
    - Three MCP servers (`contex7`, `playwright`, `memory-bank`).
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
    - `contex7/`
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

### 3.1 `contex7` MCP server (documentation provider)

**Purpose:** Provide AI-accessible documentation from both **local project directories** and **optionally remote sources
**.

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
- Optional: allow configuration of a small set of **remote documentation sources** (e.g., public URLs or APIs), with:
    - graceful timeouts
    - clear error messages if offline.
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

    - `contex7` service
    - `playwright` service
    - `memory-bank` service
    - shared network
    - named volumes (for persistent data and local projects, if appropriate)

- Each service should:
    - Build from its own `Dockerfile` under `servers/<name>/Dockerfile`.
    - Run a non-root user.
    - Define **healthchecks** (e.g., via an internal health endpoint or simple TCP check).
    - Read configuration from environment variables (overrideable via `.env` and/or CLI).
    - Log to stdout/stderr in a **structured, parseable format** (JSON logs preferred).

- **External directory mounting**:

    - Show how to mount host project directories into each service, for example:
        - `./examples/project1` on host -> `/projects/project1` in `contex7` and `playwright`.
    - Show how to mount a persistent data directory for `memory-bank`, e.g.:
        - `./data/memory-bank` on host -> `/data` in container.
    - Provide **commented examples** in `podman-compose.yml` demonstrating:
        - Single project setup.
        - Multiple project mounts.
        - How to enable/disable specific mounts.

- Include an `.env.example` file under `infra/env/` with variables like:
    - `CONTEXT7_DOCS_ROOTS=/projects/project1/docs:/projects/project2/docs`
    - `PLAYWRIGHT_PROJECT_ROOTS=/projects/project1:/projects/project2`
    - `MEMORY_BANK_DB_PATH=/data/memory-bank.db`
    - And any needed MCP configuration variables (ports, log levels, etc.).

---

## 5. Production readiness

Design and implement with **production use** in mind:

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
    - Short descriptions of `contex7`, `playwright`, `memory-bank`.
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

3. **Key files (summarized)**
    - `infra/podman-compose.yml`
    - `servers/contex7` server entrypoint & main implementation file.
    - `servers/playwright` server entrypoint & main implementation file.
    - `servers/memory-bank` server entrypoint & main implementation file.
    - `infra/env/.env.example`
    - `README.md` key sections.

4. **Selected code snippets**
    - Provide the main server bootstrap code for each MCP server.
    - Provide example tool implementations (one or two per server).
    - Provide the full `podman-compose.yml` and Dockerfiles.

5. **How to run**
    - Step-by-step instructions for:
        - Setting environment variables.
        - Building images.
        - Starting with `podman-compose`.
        - Mounting example external project directories.
        - Testing that each MCP server works.

---

## 8. Self-check before final answer

Before finalizing your response, **explicitly verify** that you have:

- Implemented all three MCP servers: `contex7`, `playwright`, `memory-bank`.
- Ensured all three are runnable via **podman-compose**.
- Demonstrated how to **mount external project directories** for:
    - docs (`contex7`)
    - code/tests (`playwright`)
    - persistent memory storage (`memory-bank`)
- Included:
    - Production-appropriate Dockerfiles.
    - A `podman-compose.yml` with healthchecks and env configuration.
    - At least basic test coverage examples.
    - Clear documentation (`README.md`, etc.).
- Addressed **error handling**, **logging**, and **configuration** for production.

If anything is missing, **add it** before returning your final answer.

```markdown

---

## Quick recap

- **Project name:** `ai-toolbox`.
- **Core components:** 3 MCP servers (`contex7`, `playwright`, `memory-bank`).
- **Orchestration:** `podman-compose` with external directory mounts.
- **Focus:** Production-ready implementation with tests, logging, and docs.

This prompt should give the AI model enough detail to generate a complete, robust starting point for the `ai-toolbox` project.

```