# Americano — gestor de torneos de pádel

Aplicación para gestionar un torneo "americano" en un club de pádel y mostrar en la pantalla del TV los resultados, la cancha de cada partido y el tiempo restante de la ronda. El organizador carga los resultados desde el celular; el TV solo muestra información.

- `/tv`: vista pública de solo lectura.
- `/admin`: panel protegido del organizador.

> Estado: arquitectura y estructura listas; el código se construye con Spec-Driven Development (Spec Kit).

## Documentación

| Documento | Para qué |
|---|---|
| [`plan.md`](plan.md) | Arquitectura, stack, tareas del esqueleto y criterios de aceptación |
| [`.specify/memory/constitution.md`](.specify/memory/constitution.md) | Principios y reglas vinculantes del proyecto |
| [`backend/README.md`](backend/README.md) | API NestJS: estructura por capas y convenciones |
| [`frontend/README.md`](frontend/README.md) | SPA React: estructura por feature y convenciones |
| `specs/` | Una carpeta por feature (spec, plan, tasks) |

## Stack

- **Backend:** NestJS + TypeScript, Prisma, PostgreSQL 16.
- **Frontend:** React + Vite + TypeScript, Redux Toolkit (RTK Query), Tailwind CSS y CSS Modules.
- **Infra:** Docker Compose, Caddy 2 (HTTPS automático), VPS, GitHub Actions + GHCR.

## Estructura

```
.
├── backend/      # NestJS + Prisma
├── frontend/     # React + Vite
├── specs/        # Specs por feature (Spec Kit)
├── scripts/      # Scripts de despliegue / operación
├── .github/workflows/   # ci.yml y deploy.yml
├── .specify/     # Spec Kit: constitución y plantillas
├── docker-compose.yml, docker-compose.override.yml, docker-compose.dev.yml
├── Caddyfile
└── .env.example
```

## Desarrollo local (cuando exista el código)

```bash
cp .env.example .env
cp .env.example backend/.env
docker compose -f docker-compose.dev.yml up -d   # solo Postgres

cd backend  && npm install && npx prisma migrate dev && npm run start:dev   # http://localhost:3000/api/health
cd frontend && npm install && npm run dev                                   # http://localhost:5173/tv
```

Stack completo en contenedores: `docker compose up --build` y abrir `http://localhost/tv`.

## Despliegue

Un push a `main` ejecuta `deploy.yml`: tests, build y push de imágenes a GHCR, deploy por SSH al VPS y verificación de `/api/health`. Rollback: ejecutar el workflow manualmente (`workflow_dispatch`) indicando el tag anterior. El `.env` de producción vive solo en el VPS. Detalles en [`plan.md`](plan.md) (Fase 5b).

## Flujo de trabajo (Spec Kit)

Cada feature sigue: `/speckit-specify` → `/speckit-clarify` → `/speckit-plan` → `/speckit-tasks` → `/speckit-implement`. Los planes se validan contra la constitución.
