# Copilot instructions: Meeting Recorder

Repository-wide rules. They apply to **all** work, backend and frontend.
Area-specific rules live in `.github/instructions/` and apply only to their paths:
- `backend.instructions.md` → `backend/**` (Spring Boot services, layers, JPA, RabbitMQ)
- `frontend.instructions.md` → `frontend/**` (Angular)

Don't apply backend conventions to frontend code, or the other way around.

## Project

Distributed system for a university course (PCBE). Users upload or record meetings, a Python worker transcribes
them and extracts tasks, and an agent proposes task assignments through an MCP server. Humans
approve every proposal. The project is in its **early stages**: most flows in `README.md` are planned, not
implemented yet. `README.md` has the event flow and MCP tools; `AGENTS.md` has commands and layout.

| Path | What |
|------|------|
| `backend/` | Maven reactor: `gateway-api`, `user-api`, `meeting-api`, `task-api`, `mcp-api` (Java 25, Spring Boot 4.1) |
| `frontend/` | Angular 21 client (TypeScript, SCSS) |
| `docker/` | `docker-compose.yml`: PostgreSQL (one schema per service) and RabbitMQ |

## System rules

- **One owner per domain.** users, roles, teams → `user-api`; meetings, audio files, transcripts → `meeting-api`;
  tasks and proposals → `task-api`. A service touches only its own schema. Other domains' data comes from
  the owner's REST API or a RabbitMQ message.
- **The frontend talks only to the gateway** through relative `/api/...` URLs.
- **`mcp-api`, `gateway-api` and the Python services have no database access.**
- **Humans approve every agent proposal.** Proposals stay `PENDING` until approved; at most one open proposal per task.
- **Contracts are shared:** REST endpoints, message schemas and MCP tools are used by teammates. Change both sides
  together and update the docs.

## Naming consistency (required)

Naming consistency is a **review requirement, not a style preference**. One concept has one name across every
layer, service and artifact: Java, database, REST, JSON, RabbitMQ messages, configuration, environment variables,
TypeScript and docs. Inconsistent names are defects.

### Domain glossary (canonical terms)

| Concept | Code name | REST resource | Don't use |
|---------|-----------|---------------|-----------|
| User | `User` | `/api/users` | account, member, person |
| Team | `Team` | `/api/teams` | group |
| Role | `Role` | (part of user) | permission, authority (in domain code) |
| Meeting | `Meeting` | `/api/meetings` | recording, lecture, session |
| Audio file of a meeting | `AudioFile` | under `/api/meetings/{meetingId}` | recording, media, upload |
| Transcript | `Transcript` | `/api/meetings/{meetingId}/transcript` | transcription, text |
| Task | `Task` | `/api/tasks` | action item, item, todo |
| Owner of a task | `owner` / `ownerId` (a `User` id) | | assignee, responsible |
| Assignment proposal | `Proposal` | `/api/proposals` | suggestion, recommendation |
| Workload of an owner | `OwnerWorkload` | | load, stats |

The project plan calls tasks "action items"; in code and docs they are always `Task` (owned by `task-api`).
Status enums are `<Concept>Status` with `UPPER_SNAKE` values (`MeetingStatus.PROCESSING`, `ProposalStatus.PENDING`).
Add new terms to this glossary before using them in code.

### Agreed contract names (kept as-is)

These come from the project plan and are shared with the Python services, so they aren't renamed even where they
deviate from the glossary: exchanges `meetings`, `rpc`; routing keys `meeting.audio.uploaded`,
`meeting.transcript.ready`, `meeting.items.extracted`; queues `audio.process`, `meeting.transcripts`, `task.items`;
MCP tools `get_overdue_items(since)`, `get_owner_workload(days)`, `propose_assignment(item_id, owner, reason)`.
New ones follow the same patterns: routing keys `<domain>.<subject>.<past-tense-event>`, MCP tools `verb_noun` in snake_case.

### Casing per artifact

| Artifact | Convention | Example |
|----------|------------|---------|
| Java/TS types | PascalCase | `Task`, `TaskResponse` |
| Java/TS methods, fields, JSON properties | camelCase | `taskId`, `dueDate` |
| Constants, enum values | UPPER_SNAKE | `MAX_UPLOAD_SIZE`, `PENDING` |
| DB tables and columns | snake_case (Hibernate default naming) | `task`, `owner_id` |
| REST paths | plural, kebab-case nouns | `/api/tasks/{taskId}` |
| Config properties | kebab-case, prefixed by service area | `task.clients.user-api.base-url` |
| Environment variables | UPPER_SNAKE mirroring the property | `TASK_CLIENTS_USER_API_BASE_URL` |
| Maven modules, files in `frontend/` | kebab-case | `meeting-api`, `task-list.ts` |

### Naming rules

- **Same concept, same name** in every place: `Task` entity → `task` table → `TaskResponse`
  record → `/api/tasks` → `taskId` JSON → `TaskResponse` TS interface → "task" in docs.
- **Names state real semantics:** `get<X>` returns `<X>` or fails if missing; `find<X>` returns an optional/empty
  result; collections use the plural (`getTasks`); booleans start with `is`/`has`/`can`. Units go in the name
  (`durationMs`, `maxSizeBytes`). Dates are `<event>At` for instants and `<event>Date` for dates (`createdAt`, `dueDate`).
- **No abbreviations** except `id`, `dto`, `api`, `url`, `jwt`, `mcp`. No vague names (`data`, `info`, `manager`,
  `helper`, `util`, `process`) when a domain term exists.
- **Follow siblings first:** before coining a name, copy the pattern of existing types in the same layer and the
  same concept in other services and in the frontend.
- **Renames are complete:** update every reference (code, config, env vars, REST paths, frontend types, docs) in the same change.

## Custom agents (`.github/agents/`)

`orchestrator` runs the delivery pipeline: clarify → `planner` → `executor` → logging/docs → `pr-reviewer`
(+ `security-reviewer`) → handoff. `explore` answers codebase questions. Only `executor` edits files; all other
agents are read-only. Tests aren't part of the current workflow; the verification gate is a successful build.

## Safety

- Never read, print or commit `.env` or real credentials. `docker/.env.example` holds local placeholders only.
- No pushes, history rewrites or deployments unless explicitly requested.
- Preserve unrelated changes; don't reformat or refactor outside the task scope.
