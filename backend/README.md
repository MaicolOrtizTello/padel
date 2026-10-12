# Backend — Americano API

API REST del gestor de torneos "americano". NestJS + TypeScript, Prisma y PostgreSQL 16. Todas las rutas cuelgan de `/api`.

> Estado: estructura preparada, aún sin código. El proyecto Nest se genera en la Fase 1 de [`plan.md`](../plan.md). Reglas de arquitectura: [`.specify/memory/constitution.md`](../.specify/memory/constitution.md).

## Estructura

```
backend/
├── prisma/                       # schema.prisma y migraciones
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── config/                   # configuración y validación de variables de entorno
│   ├── shared/
│   │   ├── domain/               # DomainError, base entity (kernel compartido)
│   │   └── infrastructure/prisma/  # PrismaModule (global) y PrismaService
│   └── modules/
│       └── <feature>/            # health ahora; tournament, round, match después
│           ├── domain/           # entidades, reglas y puertos (interfaces)
│           ├── application/      # casos de uso y DTOs
│           ├── infrastructure/   # adaptadores (repositorios Prisma, mappers)
│           ├── presentation/     # controllers
│           └── <feature>.module.ts
├── test/                         # tests e2e (*.e2e-spec.ts)
└── Dockerfile                    # multi-stage
```

Regla de dependencias: `presentation → application → domain ← infrastructure`. El dominio no importa `@nestjs/*` ni `@prisma/client`.

## Convenciones

- Archivos en inglés, `kebab-case` + sufijo de rol: `*.module.ts`, `*.controller.ts`, `*.service.ts`, `*.use-case.ts`, `*.dto.ts`, `*.entity.ts`, `*.repository.ts`, `*.mapper.ts`.
- Tests unitarios `*.spec.ts` junto al archivo; e2e en `test/`.
- Un caso de uso = una acción (`execute()`). Los controllers solo traducen HTTP.
- `health` se mantiene simple (controller + service), sin capas vacías.

## Desarrollo (cuando exista el proyecto)

```bash
# Desde la raíz del repo: levantar solo Postgres
docker compose -f docker-compose.dev.yml up -d

cd backend
cp .env.example .env        # ajustar DATABASE_URL
npm install
npx prisma migrate dev
npm run start:dev           # http://localhost:3000/api/health
```

## Scripts previstos

| Comando | Qué hace |
|---|---|
| `npm run start:dev` | Servidor con hot reload |
| `npm run build` | Compila a `dist/` |
| `npm test` | Tests unitarios (Jest) |
| `npm run test:e2e` | Tests e2e |
| `npm run lint` | Lint |
| `npx prisma migrate dev` | Crea/aplica migraciones en local |

## Variables de entorno

`PORT`, `DATABASE_URL`, `JWT_SECRET`. Se validan al arrancar: si falta una, la app no inicia. Nunca se sube un `.env` a Git.
