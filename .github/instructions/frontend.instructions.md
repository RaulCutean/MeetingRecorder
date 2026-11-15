---
applyTo: "frontend/**"
---

# Frontend (Angular) instructions

Applies only to `frontend/**`. Repository-wide rules, including the **naming glossary and naming rules**, are in
`.github/copilot-instructions.md` and apply here too. Backend conventions (Spring layers, JPA) don't apply.

## Stack and structure
- Angular 21 standalone components only (no NgModules). Signals for state, `inject()` for DI, built-in control
  flow (`@if`, `@for`, `@switch`). TypeScript strict; avoid `any`.
- Structure by feature: `src/app/features/<feature>/` (`meetings`, `tasks`, `proposals`, `auth`); shared
  code in `src/app/core` (API services, interceptors, guards, models) and `src/app/shared` (reusable UI).
- Lazy-load feature routes in `app.routes.ts`.
- Styles in SCSS, component-scoped.

## Naming
- Use `ng generate` so file and class names follow the Angular CLI 21 defaults; files are kebab-case.
- Feature folders, routes and UI wording use the glossary terms: `tasks` (never `action-items`), `proposals`, `meetings`.
- API model types **mirror the backend DTO names and fields exactly**: `TaskResponse`, `taskId`,
  `dueDate`. Don't rename or reshape fields on the client.
- Status unions mirror backend enums: `type MeetingStatus = 'PROCESSING' | 'READY'`.
- Observables end with `$` only when they are observables; signals have no suffix.

## Backend communication
- All HTTP calls go to relative `/api/...` URLs (the gateway). Never call a service port (8081 to 8084) directly.
  In development `proxy.conf.json` forwards `/api` to the gateway on `:8080`.
- Auth: a JWT HTTP interceptor adds `Authorization: Bearer <token>`; route guards protect authenticated pages.
  Never log tokens.

## Build gate
`cd frontend && npm run build`.
