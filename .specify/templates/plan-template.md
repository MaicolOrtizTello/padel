# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]

**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: TypeScript (Node 22) in backend and frontend

**Primary Dependencies**: Backend: NestJS, Prisma, class-validator. Frontend: React, Vite, React Router, Redux Toolkit (RTK Query), Tailwind CSS. List only feature-specific additions below.

**Storage**: PostgreSQL 16 via Prisma (N/A if the feature has no persistence)

**Testing**: Backend: Jest (unit + e2e). Frontend: Vitest + Testing Library + MSW

**Target Platform**: Linux VPS with Docker Compose + Caddy; browsers (phone for `/admin`, TV for `/tv`)

**Project Type**: web application (monorepo: `backend/` + `frontend/`)

**Performance Goals**: [domain-specific, e.g., 1000 req/s, 10k lines/sec, 60 fps or NEEDS CLARIFICATION]

**Constraints**: [domain-specific, e.g., <200ms p95, <100MB memory, offline-capable or NEEDS CLARIFICATION]

**Scale/Scope**: [domain-specific, e.g., 10k users, 1M LOC, 50 screens or NEEDS CLARIFICATION]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Verify against `.specify/memory/constitution.md`; mark each as PASS or justify it in Complexity Tracking:

- [ ] **I. Clean Architecture**: new backend logic sits in `domain` / `application` / `infrastructure` / `presentation`; `domain` imports no `@nestjs/*` or `@prisma/client`; use cases depend on ports.
- [ ] **II. SOLID/KISS/DRY**: one action per use case, small ports, variable rules behind an interface, no speculative abstractions.
- [ ] **III. Frontend**: feature folder layout, server data via RTK Query, UI state via slices, components never call `fetch`; Tailwind first, CSS Modules only where justified.
- [ ] **IV. Naming**: English file names following the framework conventions.
- [ ] **V. Testing**: unit tests for domain/use cases with mocked ports, e2e per endpoint, network mocked only at `baseApi`.
- [ ] **VI. Security/config**: no secrets in Git, env validated at startup, writes require organizer auth, input validated with DTOs.
- [ ] **VII. Time/refresh**: `endsAt` stored in backend, countdown computed client-side, polling for the TV, time via injectable `Clock`.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: This is the fixed project layout. Keep only the files this
  feature adds or changes, replacing <feature> with the real name (e.g.,
  tournament). Do not invent new top-level folders.
-->

```text
backend/
├── prisma/                          # schema.prisma, migrations
├── src/
│   ├── config/
│   ├── shared/                      # domain/ (DomainError, base entity), infrastructure/prisma/
│   └── modules/<feature>/
│       ├── domain/                  # entities/, ports/ (interfaces + injection tokens)
│       ├── application/             # use-cases/ (*.use-case.ts), dto/ (*.dto.ts)
│       ├── infrastructure/          # persistence/ (prisma-*.repository.ts, *.mapper.ts)
│       ├── presentation/            # *.controller.ts
│       └── <feature>.module.ts
└── test/                            # *.e2e-spec.ts (unit *.spec.ts live next to the code)

frontend/
└── src/
    ├── app/                         # router.tsx, store.ts
    ├── pages/                       # <Name>Page.tsx
    ├── features/<feature>/          # api/ (RTK Query), components/, <feature>Slice.ts, types.ts
    └── shared/                      # api/baseApi.ts, components/, lib/
```

**Structure Decision**: [List the real paths this feature touches in `backend/` and `frontend/`, and say if a layer is intentionally skipped (KISS) and why]

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |
