---
applyTo: "backend/**"
---

# Backend (Spring Boot) instructions

Applies only to `backend/**`. Repository-wide rules, including the **naming glossary and naming rules**, are in
`.github/copilot-instructions.md` and apply here too.

## Stack
Java 25, Spring Boot 4.1, Maven. Spring Data JPA on PostgreSQL (one schema per service), Spring AMQP (RabbitMQ),
Spring Cloud Gateway (WebMVC) in `gateway-api`, Spring AI MCP in `mcp-api` (to be added).

## Module structure
`backend/pom.xml` is the single parent/reactor POM: it centralizes all versions (Spring Boot parent, Spring Cloud
and Spring AI BOMs) and lists the modules. Each service is **one Maven module**; don't create nested sub-modules.
Child POMs declare dependencies **without versions**. If no BOM manages a dependency, add a version property and a
`dependencyManagement` entry in `backend/pom.xml`.

| Module | Port | Database schema | Packages |
|--------|------|--------------------------|----------|
| `gateway-api` | 8080 | none | `config`, `filter` (routes in `application.properties`) |
| `user-api` | 8081 | `user_schema` | controller, service, repository, client, model, dto, config |
| `meeting-api` | 8082 | `meeting_schema` | controller, service, repository, client, model, dto, config |
| `task-api` | 8083 | `task_schema` | controller, service, repository, client, model, dto, config |
| `mcp-api` | 8084 | none | controller, service, client, dto, config |

## Layered architecture
Packages: `ro.pcbe.meetingrecorder.<service>.<layer>`. Calls flow **one way only**:

```
controller  ->  service  ->  repository   (own database schema)
                        \->  client       (other services over REST, RabbitMQ publishers)
```

- **controller**: entrypoints: REST controllers and RabbitMQ listeners (MCP tool definitions in `mcp-api`).
  Validates input (`@Valid`), maps to/from DTOs, delegates to a service. No business logic, no `@Transactional`,
  never calls repositories or clients. `@RestControllerAdvice` returns `ProblemDetail` (RFC 9457) errors.
- **service**: business logic and `@Transactional`. Orchestrates repositories and clients. Converts entities to DTOs
  before returning to controllers.
- **repository**: Spring Data JPA interfaces for this service's own entities only. Absent in `mcp-api` and `gateway-api`.
- **client**: all outbound integration: `RestClient`-based clients (base URL from config) and `RabbitTemplate`
  publishers. Translates transport errors into meaningful exceptions. In services without a database it is the only data source.
- **model**: JPA entities. Never returned from controllers or sent in messages.
- **dto**: Java records for requests, responses and messages.
- **config**: beans, `@ConfigurationProperties`, RabbitMQ exchanges/queues/bindings/dead-letter queues, CORS.

## Class and method naming per layer
Use the glossary term as `<Concept>` (e.g. `Task`, never `ActionItem`).

| Layer | Class name | Methods |
|-------|------------|---------|
| controller | `<Concept>Controller`, `<Event>Listener` | HTTP-verb oriented: `getTask`, `createTask`, `onTranscriptReady` |
| service | `<Concept>Service` | `get<X>` (throws not-found), `find<X>` (Optional), `create<X>`, `update<X>`, `delete<X>`, domain verbs (`approveProposal`) |
| repository | `<Concept>Repository` | Spring Data derived queries: `findBy…`, `existsBy…`, `countBy…` |
| client | `<TargetService>Client` (`UserApiClient`), `<Event>Publisher` | mirror the remote operation: `getUser`, `publishAudioUploaded` |
| model | `<Concept>`, `<Concept>Status` | |
| dto | `<Concept>Request`, `<Concept>Response`, `<Subject><Event>Message` (`TranscriptReadyMessage`) | |
| config | `<Area>Config`, `<Area>Properties` | |

Exceptions: `<Concept>NotFoundException`, `<Reason>Exception`. Bean names and constants follow the same terms.

## Persistence
- Entities map to the service's own schema (`hibernate.default_schema` is set). Tables are managed by Hibernate
  `ddl-auto: update` during these early stages (no migration tool yet). `update` adds tables and columns but
  doesn't rename or drop them; reset local data with `cd docker && docker compose down -v`.
- Never add a datasource for another schema, map another service's tables or join across services.
- `open-in-view` is disabled: load what you need inside the service transaction.

## Messaging
- Declare exchanges, queues, bindings and one dead-letter queue per consumer queue as `@Bean`s in `config`.
- Register a JSON `MessageConverter` bean (Jackson 3, as shipped with Boot 4). Listeners must be idempotent and
  must not requeue forever.

## General
- Constructor injection only (no field `@Autowired`). Java records for DTOs and messages.
- Configuration in `application.properties`; secrets via `${ENV_VAR:local-default}` matching `docker/.env.example`.
- Logging: SLF4J; log ids, counts, status, duration and exception type. Never log passwords, JWTs, `Authorization`
  headers, cookies, credentials, audio or transcript contents, or full payloads.
- Build gate: `cd backend && ./mvnw -q -pl <module> -am compile` (JDK 25). Running a service needs `cd docker && docker compose up -d`.
