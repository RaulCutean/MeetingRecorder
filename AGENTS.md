# AGENTS.md

Entry point for AI coding agents (GitHub Copilot, Copilot cloud agent, others) working in this repository.
The rules live in one place; read them before changing anything:

| File | Applies to | Contents |
|------|------------|----------|
| `.github/copilot-instructions.md` | everything | project, system rules, **naming glossary and naming rules**, agents, safety |
| `.github/instructions/backend.instructions.md` | `backend/**` | modules, layered architecture, per-layer naming, persistence, messaging |
| `.github/instructions/frontend.instructions.md` | `frontend/**` | Angular structure, naming, gateway communication |
| `README.md` | | architecture, event flow, MCP tools, work split |

## Layout

| Path | What |
|------|------|
| `backend/pom.xml` | Parent/reactor POM (groupId `ro.pcbe.meetingrecorder`): all versions and the module list |
| `backend/gateway-api` | Spring Cloud Gateway (WebMVC), port 8080, routes in `application.properties` |
| `backend/user-api` | Users, roles, teams, login/JWT. Port 8081, schema `user_schema` |
| `backend/meeting-api` | Meetings, audio files, transcripts, full-text search. Port 8082, schema `meeting_schema` |
| `backend/task-api` | Tasks and proposals. Port 8083, schema `task_schema` |
| `backend/mcp-api` | MCP server (Spring AI MCP, to be added). Port 8084, **no database** |
| `frontend/` | Angular 21 standalone app, dev proxy `/api` → `:8080` |
| `docker/` | `docker-compose.yml` (PostgreSQL + RabbitMQ) and `.env.example` |

Each service is a single Maven module with only its `*ApiApplication` class to start. Packages
(`ro.pcbe.meetingrecorder.<service>.<layer>`) are created when the first real class needs them; don't pre-create
empty packages or `package-info.java` files. Component scanning picks up everything under the application package.

## Commands

```bash
cd docker && docker compose up -d                        # PostgreSQL + RabbitMQ (needed to run services)
cd backend && ./mvnw -q compile                           # build gate, all modules (JDK 25)
cd backend && ./mvnw -q -pl task-api -am compile          # build gate, one module
cd backend && ./mvnw -pl user-api spring-boot:run         # run one service
cd frontend && npm start                                  # dev server on :4200
cd frontend && npm run build                              # frontend build gate
```

On macOS with several JDKs: `export JAVA_HOME=$(/usr/libexec/java_home -v 25)`.
Tests aren't part of the current workflow: don't add them unless explicitly asked.

## Custom agents (`.github/agents/`)

| Agent | Role |
|-------|------|
| `orchestrator` | Runs the pipeline: clarify → plan → implement → logging/docs → review → handoff |
| `planner` | Turns a user story into a per-module, per-layer Scope Manifest (read-only) |
| `executor` | The only agent that edits files |
| `explore` | Codebase Q&A with file references (read-only) |
| `pr-reviewer` | Reviews diffs, including the naming-consistency check (read-only) |
| `security-reviewer` | Security audit (read-only) |
