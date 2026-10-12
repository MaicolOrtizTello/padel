# Americano Constitution

Gestor de torneos "americano" de pádel. El organizador carga resultados desde el celular (`/admin`); el TV del club solo muestra información en modo lectura (`/tv`).

## Core Principles

### I. Clean Architecture proporcional (NON-NEGOTIABLE)
El backend se organiza por módulo de negocio en cuatro capas: `domain`, `application`, `infrastructure`, `presentation`. Las dependencias apuntan hacia adentro (`presentation → application → domain ← infrastructure`).
- `domain` no importa `@nestjs/*`, `@prisma/client` ni ninguna librería de infraestructura.
- Los casos de uso dependen de puertos (interfaces) definidos en `domain`; Prisma solo se usa en `infrastructure`.
- Las capas completas se aplican solo a módulos con lógica de negocio. Módulos simples (p. ej. `health`) no llevan capas vacías.

### II. SOLID, KISS y DRY
- Un caso de uso = una acción (`execute()`). Controllers solo traducen HTTP ↔ caso de uso.
- Reglas variables (p. ej. rotación de parejas) van detrás de una interfaz (`PairingStrategy`); extender sin modificar.
- Puertos pequeños y específicos; nada de repositorios "dios".
- Lo más simple que cumpla el requisito. Prohibido anticipar necesidades (YAGNI): sin CQRS, eventos ni abstracciones genéricas hasta que una spec lo exija.
- Lo compartido vive en `shared/`. Toda complejidad añadida debe justificarse en el plan (sección Complexity Tracking).

### III. Frontend por feature con Redux Toolkit
- Estructura `app/`, `pages/`, `features/<feature>/`, `shared/`. Las páginas solo componen features.
- Datos del servidor con **RTK Query** (un único `baseApi`; endpoints inyectados por feature; polling con `pollingInterval`). Estado de UI con slices de **Redux Toolkit**.
- Los componentes solo renderizan: nunca llaman `fetch` ni contienen lógica de negocio.
- Estilos: **Tailwind CSS** por defecto; **CSS Modules** solo para animaciones, selectores de estado complejos o utilidades ilegibles. Tokens de diseño definidos una sola vez en `@theme`.

### IV. Convenciones de nombres (NON-NEGOTIABLE)
Todo el código y los nombres de archivo están en **inglés** y siguen la convención del framework.
- Backend (Nest): `kebab-case` + sufijo de rol: `*.module.ts`, `*.controller.ts`, `*.service.ts`, `*.use-case.ts`, `*.dto.ts`, `*.entity.ts`, `*.repository.ts`, `*.mapper.ts`, `*.spec.ts`, `*.e2e-spec.ts`.
- Frontend (React): componentes y páginas `PascalCase.tsx`; hooks `useXxx.ts`; utilidades `camelCase.ts`; `Component.module.css` junto a su componente; tests `*.test.tsx`.

### V. Testing en proporción al riesgo
- La lógica de dominio y los casos de uso se prueban con tests unitarios (Jest en backend; Vitest + Testing Library en frontend) usando implementaciones in-memory o mocks de los puertos, nunca la base de datos real.
- Cada endpoint tiene al menos un test e2e. Las reglas del americano (rotación, puntajes, ranking) requieren tests antes de darse por terminadas.
- La red del frontend se mockea en un único punto (`baseApi`, con MSW).

### VI. Seguridad y configuración
- Ningún secreto ni `.env` en Git; todo valor configurable llega por variables de entorno, validadas al arrancar (la app no inicia si falta una).
- `/tv` es público y de solo lectura; `/admin` y toda escritura requieren autenticación del organizador.
- Entrada validada en el borde (`ValidationPipe` con DTOs) y nunca confiar en el cliente.

### VII. Tiempo y actualización del TV
- El backend guarda `endsAt` por ronda; la cuenta regresiva se calcula en el navegador. No se emite un evento por segundo.
- El TV actualiza por polling (5-10 s). SSE/WebSockets solo si una spec lo justifica.
- El tiempo se obtiene por un `Clock` inyectable (nunca `Date.now()` directo en dominio) para poder probarlo.

## Stack y despliegue

- **Backend:** NestJS + TypeScript, Prisma, PostgreSQL 16. API bajo prefijo `/api`.
- **Frontend:** React + Vite + TypeScript, React Router, Redux Toolkit + RTK Query, Tailwind CSS.
- **Infra:** monorepo (`backend/`, `frontend/`, sin workspaces), Docker Compose, Caddy 2 (HTTPS automático, mismo dominio para front y API → sin CORS), VPS pequeño.
- **CI/CD:** GitHub Actions. PR → `ci.yml` (lint, typecheck, tests, build). Push a `main` → `deploy.yml` (tests → build y push de imágenes a GHCR → deploy por SSH → healthcheck a `/api/health`). Rollback por tag con `workflow_dispatch`. El `.env` de producción vive solo en el VPS.
- Las migraciones de Prisma se aplican con `prisma migrate deploy` al arrancar la API.

## Flujo de trabajo (Spec-Driven Development)

1. Todo cambio funcional nace de una spec en `specs/<NNN-feature>/` con el flujo de Spec Kit: `specify → clarify → plan → tasks → implement`.
2. El `plan.md` de cada feature incluye un **Constitution Check** contra estos principios; las violaciones se justifican explícitamente.
3. `plan.md` en la raíz es el documento de arquitectura/tarea inicial; las decisiones allí tomadas son vinculantes mientras no se enmiende esta constitución.
4. Ningún merge a `main` sin CI en verde; `main` protegida y cambios vía PR.
5. Commits pequeños y enfocados, uno por tarea de `tasks.md` cuando sea posible.

## Governance

Esta constitución prevalece sobre cualquier otra práctica del repositorio. Toda enmienda se documenta en este archivo, incrementa la versión (MAJOR: se elimina o redefine un principio; MINOR: se añade uno o se amplía; PATCH: aclaraciones) y actualiza los artefactos afectados (`plan.md`, plantillas en `.specify/templates/`, READMEs). Los PRs y revisiones verifican el cumplimiento; la complejidad adicional debe justificarse.

**Version**: 1.0.0 | **Ratified**: 2026-10-12 | **Last Amended**: 2026-10-12
