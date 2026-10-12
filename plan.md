# Plan — Gestor de "Americano" para club de pádel

## 1. Objetivo del proyecto

Aplicación para gestionar un torneo "americano" de pádel y mostrar en la pantalla de un TV del club:

- Resultados de los partidos.
- Cancha en la que se juega cada partido.
- Tiempo restante de la ronda en curso.

El organizador carga los resultados desde el celular; el TV solo muestra información (solo lectura).

## 2. Alcance de esta primera tarea

**Solo arquitectura y tecnologías.** Al terminar, debe existir un esqueleto funcional que corra en local y en un VPS, sin lógica de negocio.

**Fuera de alcance (se define después, ver sección 9):** modelo de dominio real (torneos, parejas, canchas, rondas, partidos), algoritmo de emparejamiento y propuesta de cancha, login del organizador, pantalla del TV con datos reales.

## 3. Decisiones de stack

| Capa | Decisión | Motivo |
|---|---|---|
| Backend | NestJS (TypeScript) | Estructura modular, soporte de WebSockets/SSE a futuro |
| Frontend | React + Vite (TypeScript) | Build estático simple, rápido para un MVP |
| Estado (frontend) | Redux Toolkit + RTK Query | Estado global predecible; RTK Query cubre fetching, caché y polling |
| Estilos (frontend) | Tailwind CSS + CSS Modules | Tailwind para layout, espaciado y tokens de diseño; CSS Modules para estilos propios de un componente |
| Base de datos | PostgreSQL 16 | Datos muy relacionales (rankings, partidos con 4 jugadores y cancha) |
| ORM | Prisma | Rápido para MVP, migraciones simples (cambiable a TypeORM si se prefiere) |
| Servidor web / proxy | Caddy 2 | Sirve el build de React, hace proxy a `/api` y gestiona HTTPS automático |
| Contenedores | Docker + Docker Compose | Mismo entorno en local y en VPS |
| Hosting | VPS pequeño (~4-6 USD/mes: Hetzner, DigitalOcean, Vultr) | Costo bajo y sin servicios que se "duermen" durante un evento |
| Repositorio | Monorepo simple (dos carpetas, un solo Git) | Un `git clone`, un `docker-compose.yml`, cambios coordinados |
| CI/CD | GitHub Actions + GHCR (GitHub Container Registry) | Build en CI, el VPS solo hace `pull`; rollback por tag |
| Testing | Jest (backend), Vitest + Testing Library (frontend) | Jest viene con Nest; Vitest se integra con Vite |

### Principios y convenciones

- **SOLID, Clean Architecture, KISS, DRY**, aplicados de forma proporcional: capas completas solo en módulos con lógica de negocio. `health` y la infraestructura se mantienen simples (sin capas vacías).
- **Nombres de archivos en inglés** y según la convención de cada framework:
  - Backend (Nest): `kebab-case` + sufijo de rol: `*.module.ts`, `*.controller.ts`, `*.service.ts`, `*.use-case.ts`, `*.dto.ts`, `*.entity.ts`, `*.repository.ts`, `*.mapper.ts`, `*.spec.ts` (junto al archivo), `*.e2e-spec.ts` (en `backend/test/`). Clases en PascalCase.
  - Frontend (React): componentes y páginas `PascalCase.tsx`; hooks `useXxx.ts`; utilidades/módulos `camelCase.ts`; tests `*.test.tsx` junto al archivo; CSS Modules `PascalCase.module.css` junto a su componente (`RoundCountdown.tsx` + `RoundCountdown.module.css`).
- **Regla de dependencias (backend):** `presentation → application → domain ← infrastructure`. El dominio no importa `@nestjs/*` ni `@prisma/client`.

#### Backend: capas por módulo

```
src/modules/<feature>/
├── domain/            # entidades, value objects, reglas y PUERTOS (interfaces + token de inyección)
├── application/       # casos de uso (`execute()`) y DTOs; dependen solo de domain
├── infrastructure/    # adaptadores que implementan los puertos (Prisma repository, mapper)
├── presentation/      # controllers HTTP
└── <feature>.module.ts  # cableado de DI: { provide: X_REPOSITORY, useClass: PrismaXRepository }
```

- **S:** el controller solo traduce HTTP ↔ caso de uso; un caso de uso = una acción; el repositorio solo persiste; el mapper convierte Prisma ↔ dominio.
- **O:** reglas variables (p. ej. rotación de parejas) como estrategias detrás de una interfaz (`PairingStrategy`), sin tocar el caso de uso.
- **L:** toda implementación de un puerto (Prisma, in-memory para tests) es intercambiable.
- **I:** puertos pequeños y específicos (`TournamentRepository`, `Clock`), no un repositorio "dios".
- **D:** los casos de uso dependen de interfaces del dominio; Prisma queda confinado a `infrastructure/`.
- **KISS:** `health` = un controller + un `HealthService` que hace `SELECT 1`. Sin CQRS, eventos ni DDD táctico hasta que el dominio lo pida.
- **DRY:** `shared/` con base entity, `DomainError`, filtro global de excepciones (`DomainError` → HTTP) y un `Clock` inyectable (útil para `endsAt` y tests).

#### Frontend: organización por feature

```
src/
├── app/        # App.tsx, router.tsx (rutas /tv y /admin), store.ts (configureStore), hooks.ts (useAppDispatch/useAppSelector tipados)
├── pages/      # una por ruta; solo componen features (TvPage.tsx, AdminPage.tsx)
├── features/   # <feature>/{api,components,types} + slice: health ahora; tv y admin después
└── shared/     # api/baseApi.ts (RTK Query), components/, lib/
```

- **Redux Toolkit** como gestor de estado global. Dos tipos de estado, cada uno con su herramienta:
  - **Datos del servidor** (health, partidos, rondas, ranking): **RTK Query**. Un único `baseApi` (`createApi` + `fetchBaseQuery({ baseUrl: '/api' })`) y cada feature inyecta sus endpoints (`injectEndpoints`). El polling del TV se resuelve con la opción `pollingInterval`, así que no se escribe un `usePolling` propio (DRY).
  - **Estado de UI/cliente** (sesión del organizador, filtros, formularios en curso): slices con `createSlice`, uno por feature (`features/<feature>/<feature>Slice.ts`).
- Flujo de dependencias: componente → hooks generados por RTK Query / selectores → store. Los componentes solo renderizan y nunca llaman `fetch`; `baseApi` es el único punto de red, y los tests lo mockean con MSW o un `baseQuery` falso.
- KISS: no se normaliza estado a mano ni se agregan middlewares propios; solo lo que traen RTK y RTK Query.
- **Estilos: Tailwind CSS + CSS Modules**, con una regla simple para no mezclarlos al azar:
  - **Tailwind por defecto** para layout, espaciado, tipografía y colores. Los tokens de diseño (colores, escala tipográfica para TV, breakpoints) se definen una sola vez en `styles/index.css` (bloque `@theme` de Tailwind v4) y se reutilizan en todo el proyecto (DRY).
  - **CSS Module** (`Component.module.css`) solo cuando un componente necesita estilos que en utilidades quedan ilegibles: animaciones con `@keyframes` (p. ej. parpadeo del contador), selectores de estado complejos o más de ~8 utilidades repetidas. Dentro del módulo se puede usar `@apply` y los tokens de `@theme`.
  - Un mismo elemento no mezcla ambos para la misma propiedad. Para clases condicionales se usa `clsx`.
  - Componentes reutilizables de `shared/components/` encapsulan los patrones repetidos (`Button`, `Card`) en lugar de copiar cadenas de clases.
- Props pequeñas y específicas; composición sobre flags de configuración.

### Decisiones de diseño ya tomadas (para cuando se defina el dominio)

- **Tiempo restante:** el backend guarda la hora de término (`endsAt`) de cada ronda; la cuenta regresiva se calcula en el navegador. No se emite un evento por segundo.
- **Actualización del TV:** polling cada 5-10 segundos en el MVP. Si más adelante se quiere algo más fluido, migrar a SSE o WebSockets.
- **Acceso:** vista pública de solo lectura (`/tv`) y panel protegido (`/admin`) para el organizador.
- **Mismo dominio para front y API** (`/api/*` va a Nest): sin problemas de CORS.

## 4. Estructura del repositorio

```
americano/
├── .github/
│   └── workflows/
│       ├── ci.yml            # lint + typecheck + tests + build de prueba (PR y workflow_call)
│       └── deploy.yml        # push a main: build + push a GHCR + deploy por SSH
├── backend/                  # NestJS + Prisma
│   ├── prisma/
│   ├── src/
│   │   ├── main.ts
│   │   ├── app.module.ts
│   │   ├── config/           # app.config.ts, env.validation.ts
│   │   ├── shared/
│   │   │   ├── domain/       # base-entity.ts, domain-error.ts
│   │   │   └── infrastructure/prisma/   # prisma.module.ts, prisma.service.ts
│   │   └── modules/
│   │       └── health/       # health.module.ts, health.controller.ts, health.service.ts
│   ├── test/                 # *.e2e-spec.ts
│   ├── Dockerfile
│   └── package.json
├── frontend/                 # React + Vite
│   ├── src/
│   │   ├── main.tsx
│   │   ├── app/              # App.tsx, router.tsx, store.ts, hooks.ts
│   │   ├── pages/            # TvPage.tsx, AdminPage.tsx
│   │   ├── features/health/  # api/healthApi.ts (RTK Query), components/HealthStatus.tsx, types.ts
│   │   ├── shared/           # api/baseApi.ts
│   │   └── styles/           # index.css: @import "tailwindcss" + @theme (tokens)
│   ├── Dockerfile
│   └── package.json
├── docker-compose.yml          # Producción: imágenes de GHCR (db + api + web)
├── docker-compose.override.yml # Local: añade `build:` para api y web (se aplica solo en local)
├── docker-compose.dev.yml      # Desarrollo: solo Postgres
├── Caddyfile
├── .env.example
├── .gitignore
├── plan.md
└── README.md
```

No se usan workspaces, Turborepo ni Nx. Cada carpeta tiene su propio `package.json`. No se crea paquete `shared/` por ahora (se duplican los pocos tipos necesarios en el front).

## 5. Tareas

### Fase 0 — Repositorio

- [ ] Crear el repositorio y la estructura de carpetas de la sección 4.
- [ ] Crear `.gitignore` en la raíz (`node_modules`, `dist`, `.env`, volúmenes locales).
- [ ] Crear `.env.example` con los nombres de las variables (sección 7). **No subir `.env` a Git.**
- [ ] Crear un `README.md` breve con cómo levantar el proyecto en local.

### Fase 1 — Backend (NestJS)

- [ ] Inicializar NestJS en `backend/` (`nest new` o `@nestjs/cli`), con TypeScript.
- [ ] Configurar prefijo global `api` (`app.setGlobalPrefix('api')`).
- [ ] Instalar y configurar `@nestjs/config` leyendo variables de entorno (sin valores escritos en el código).
- [ ] Activar `ValidationPipe` global (`class-validator` + `class-transformer`).
- [ ] Habilitar `app.enableShutdownHooks()` para cierres limpios en Docker.
- [ ] Crear módulo `health` (`health.module.ts`, `health.controller.ts`, `health.service.ts`) con `GET /api/health` que responda `{ status: 'ok' }` y verifique la conexión a la base de datos (`SELECT 1` vía `PrismaService`).
- [ ] Crear `shared/domain` (`DomainError`) y un filtro global de excepciones que traduzca `DomainError` a respuesta HTTP.
- [ ] Validar las variables de entorno al arrancar (`config/env.validation.ts`); si falta una, la app no inicia.
- [ ] Test unitario de `HealthService` (Prisma mockeado) y un e2e de `GET /api/health`.
- [ ] El puerto de escucha se toma de `PORT` (por defecto 3000) y escucha en `0.0.0.0`.

### Fase 2 — Base de datos (Postgres + Prisma)

- [ ] Instalar Prisma en `backend/` y ejecutar `prisma init` con proveedor `postgresql`. Seguir la documentación vigente de Prisma para la versión instalada.
- [ ] Configurar `DATABASE_URL` desde variables de entorno.
- [ ] Crear `PrismaService` y `PrismaModule` (global) en `src/shared/infrastructure/prisma/` que conecten y desconecten correctamente. Prisma solo se usa desde `infrastructure/`.
- [ ] Crear una migración inicial mínima para verificar que el flujo de migraciones funciona (el dominio real se modela después).
- [ ] Levantar Postgres en local con `docker-compose.dev.yml` y comprobar que `/api/health` confirma la conexión.

### Fase 3 — Frontend (React + Vite)

- [ ] Crear el proyecto en `frontend/` con Vite (plantilla `react-ts`).
- [ ] Instalar React Router y definir dos rutas placeholder en `app/router.tsx`: `/tv` (`TvPage.tsx`) y `/admin` (`AdminPage.tsx`).
- [ ] Configurar el proxy de desarrollo en `vite.config.ts` para que `/api` apunte a `http://localhost:3000`.
- [ ] Instalar `@reduxjs/toolkit` y `react-redux`. Crear `app/store.ts` (`configureStore`), `app/hooks.ts` (`useAppDispatch` y `useAppSelector` tipados) y envolver la app con `<Provider>`.
- [ ] Crear `shared/api/baseApi.ts`: `createApi` con `fetchBaseQuery({ baseUrl: '/api' })` y registrar su reducer y middleware en el store.
- [ ] Crear `features/health/` (`api/healthApi.ts` con `injectEndpoints`, `components/HealthStatus.tsx`, `types.ts`).
- [ ] En `/tv`, mostrar `HealthStatus` usando `useGetHealthQuery(undefined, { pollingInterval })` para validar el ciclo completo front → proxy → API → DB.
- [ ] Configurar Vitest + Testing Library + MSW; test de `HealthStatus` renderizado con un store de prueba y la API mockeada.
- [ ] Instalar Tailwind CSS (v4 con `@tailwindcss/vite`) e importarlo en `styles/index.css`; definir en `@theme` los tokens base (fondo oscuro, colores, tamaños de tipografía grandes para TV).
- [ ] Confirmar que CSS Modules funciona (viene soportado por Vite): crear un `HealthStatus.module.css` mínimo (p. ej. indicador de estado) para validar la convención. Instalar `clsx`.
- [ ] Estilos base pensados para pantalla grande (fondo oscuro, tipografía grande) con Tailwind. Detalle visual queda para la fase de dominio.

### Fase 4 — Docker y Caddy

- [ ] `backend/Dockerfile` multi-stage (ver referencia en sección 6).
- [ ] `frontend/Dockerfile`: build de Vite y servido con Caddy (ver sección 6).
- [ ] `docker-compose.yml` con tres servicios: `db`, `api`, `web`, usando `image:` de GHCR (`TAG` por defecto `latest`).
- [ ] `docker-compose.override.yml` con `build:` para `api` y `web`: en local `docker compose up --build` construye desde el código; en el VPS (donde no existe este archivo) se usan las imágenes de GHCR.
- [ ] `docker-compose.dev.yml` solo con `db`, exponiendo el puerto 5432 únicamente en `127.0.0.1`.
- [ ] `Caddyfile` con proxy a `/api/*` y `try_files` para que React Router funcione al recargar.
- [ ] Volumen persistente para Postgres (`pgdata`) y para Caddy (`caddy_data`, `caddy_config`).
- [ ] La API ejecuta `prisma migrate deploy` al arrancar.
- [ ] Probar el stack completo en local con `docker compose up --build` y abrir `http://localhost/tv`.

### Fase 5 — Despliegue en VPS

- [ ] Crear el VPS (Ubuntu LTS, 1-2 GB de RAM) y acceso por **llave SSH** (deshabilitar login por contraseña).
- [ ] Firewall (ufw): permitir solo 22, 80 y 443.
- [ ] Instalar Docker y el plugin de Compose.
- [ ] Comprar/configurar el dominio y apuntar el registro DNS tipo A a la IP del VPS.
- [ ] Crear en el VPS el directorio de despliegue (p. ej. `/opt/americano`) con el `.env` de producción (contraseñas y secretos fuertes). No hace falta clonar el repo: `docker-compose.yml` y `Caddyfile` los copia el pipeline.
- [ ] `docker login ghcr.io` una vez en el VPS con un PAT de solo lectura (`read:packages`).
- [ ] Primer despliegue (manual o vía pipeline) con `DOMAIN` definido en el `.env`.
- [ ] Verificar HTTPS automático y que `https://tu-dominio/tv` y `/api/health` respondan.
- [ ] Documentar en el README el despliegue automático y el rollback (ver Fase 5b).

### Fase 5b — CI/CD (GitHub Actions)

Flujo: PR → `ci.yml`; push a `main` → `deploy.yml` (test → build y push a GHCR → deploy por SSH → healthcheck).

- [ ] `.github/workflows/ci.yml`: triggers `pull_request` y `workflow_call`; jobs `backend` y `frontend` en paralelo (`setup-node` con cache de npm: lint, typecheck, tests, build) y build de prueba de las imágenes Docker.
- [ ] `.github/workflows/deploy.yml`: triggers `push` a `main` y `workflow_dispatch` (re-deploy o rollback indicando un tag).
  - Job `test`: reusa `ci.yml`.
  - Job `build`: construye y publica `ghcr.io/<usuario>/americano-api` y `americano-web` con tags `<sha>` y `latest` (autenticación con `GITHUB_TOKEN`).
  - Job `deploy` (`environment: production`, `concurrency: deploy`): copia `docker-compose.yml` y `Caddyfile` al VPS, ejecuta por SSH `TAG=<sha> docker compose pull && docker compose up -d --remove-orphans && docker image prune -f`.
  - Paso final: `curl -f https://$DOMAIN/api/health`; si falla, el job falla.
- [ ] Configurar en GitHub (Environment `production`) los secrets `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY` y `VPS_KNOWN_HOSTS`. La llave SSH es dedicada solo para deploy y el usuario no es root (pertenece al grupo `docker`).
- [ ] Proteger la rama `main`: exigir PR con `ci.yml` en verde.
- [ ] Probar un rollback: `workflow_dispatch` con el tag de una versión anterior.

### Fase 6 — Operación mínima

- [ ] Backup diario de Postgres con `pg_dump` en un cron del VPS (guardar fuera del contenedor y, si es posible, copiar a un bucket gratuito o a otra máquina).
- [ ] Probar una restauración del backup al menos una vez.
- [ ] Activar actualizaciones de seguridad automáticas del sistema operativo (`unattended-upgrades`).
- [ ] (Opcional) Snapshots del proveedor.

## 6. Referencias de configuración

> Son puntos de partida. Ajustar versiones y rutas al generar los proyectos reales (por ejemplo, confirmar que el build de Nest deja el archivo en `dist/main.js`).

### `backend/Dockerfile`

```dockerfile
FROM node:22-slim AS build
RUN apt-get update -y && apt-get install -y openssl && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate && npm run build
RUN npm prune --omit=dev

FROM node:22-slim
RUN apt-get update -y && apt-get install -y openssl && rm -rf /var/lib/apt/lists/*
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/prisma ./prisma
COPY --from=build /app/package*.json ./
EXPOSE 3000
CMD ["sh", "-c", "npx prisma migrate deploy && node dist/main.js"]
```

Multi-stage desde el inicio: como las imágenes se publican en GHCR, la imagen final no debe llevar devDependencies ni código fuente. `prisma` (CLI) debe estar en `dependencies` para que `migrate deploy` funcione tras `npm prune --omit=dev`. Confirmar la ruta real de `dist/main.js` al generar el proyecto.

### `frontend/Dockerfile`

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM caddy:2-alpine
COPY --from=build /app/dist /srv
```

El `Caddyfile` se monta desde la raíz mediante el compose.

### `Caddyfile`

```
{$DOMAIN:localhost} {
    encode gzip

    handle /api/* {
        reverse_proxy api:3000
    }

    handle {
        root * /srv
        try_files {path} /index.html
        file_server
    }
}
```

Con `DOMAIN` sin definir usa `localhost` (sin HTTPS público). Con un dominio real, Caddy obtiene y renueva el certificado solo.

### `docker-compose.yml`

> En esta versión `api` y `web` usan `image:` de GHCR (producción). El archivo `docker-compose.override.yml` (solo local) añade `build: ./backend` y `build: ./frontend`.

```yaml
services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    image: ghcr.io/<usuario>/americano-api:${TAG:-latest}
    restart: unless-stopped
    environment:
      NODE_ENV: production
      PORT: 3000
      DATABASE_URL: postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      JWT_SECRET: ${JWT_SECRET}
    depends_on:
      db:
        condition: service_healthy

  web:
    image: ghcr.io/<usuario>/americano-web:${TAG:-latest}
    restart: unless-stopped
    environment:
      DOMAIN: ${DOMAIN:-localhost}
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - api

volumes:
  pgdata:
  caddy_data:
  caddy_config:
```

### `docker-compose.override.yml` (solo local)

```yaml
services:
  api:
    build: ./backend
  web:
    build: ./frontend
```

Docker Compose lo carga automáticamente junto a `docker-compose.yml`. En el VPS este archivo no existe, así que se usan las imágenes de GHCR.

### `docker-compose.dev.yml`

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - pgdata_dev:/var/lib/postgresql/data

volumes:
  pgdata_dev:
```

Desarrollo diario: `docker compose -f docker-compose.dev.yml up -d`, luego `npm run start:dev` en `backend/` y `npm run dev` en `frontend/` (con hot reload).

## 7. Variables de entorno (`.env.example`)

```
# Base de datos
POSTGRES_USER=americano
POSTGRES_PASSWORD=cambiar_esto
POSTGRES_DB=americano

# Para desarrollo local fuera de Docker (backend/.env)
DATABASE_URL=postgresql://americano:cambiar_esto@localhost:5432/americano

# Backend
PORT=3000
JWT_SECRET=cambiar_por_un_secreto_largo_y_aleatorio

# Producción
DOMAIN=americano.tuclub.cl
```

Secrets del pipeline (se configuran en GitHub → Environment `production`, **no** van en `.env` ni en el repo): `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, `VPS_KNOWN_HOSTS`. El `.env` de producción vive solo en el VPS.

## 8. Criterios de aceptación

La primera tarea está terminada cuando:

1. `docker compose up --build` levanta `db`, `api` y `web` sin errores en local.
2. `http://localhost/tv` carga la app React y muestra el estado de salud obtenido de `GET /api/health` (que consulta la base de datos).
3. Al recargar `http://localhost/tv` o `/admin` la página sigue funcionando (el `try_files` está bien configurado).
4. En desarrollo, `npm run start:dev` + `npm run dev` + Postgres en Docker funcionan con hot reload y el proxy `/api`.
5. No hay secretos ni `.env` en el repositorio.
6. El mismo stack está desplegado en el VPS con HTTPS válido sobre el dominio.
7. Existe al menos un backup de Postgres probado.
8. El README explica cómo levantar el proyecto en local, cómo funciona el despliegue automático y cómo hacer rollback.
9. `ci.yml` pasa en verde en un PR (lint, typecheck, tests y build).
10. Un push a `main` ejecuta `deploy.yml` y termina con `https://<dominio>/api/health` respondiendo OK.
11. El código sigue la estructura y las convenciones de nombres de la sección 3 (nombres en inglés, capas por módulo, `health` sin sobreingeniería).

## 9. Requisitos de dominio (borrador para Spec Kit)

Fuera del alcance de la tarea de esqueleto. Cada punto se convierte en una spec con `/speckit-specify` (ver el orden sugerido al final). Aquí solo se fija el *qué*; el *cómo* se decide en `/speckit-plan`.

### 9.1 Flujo del torneo

1. **Inscripción de parejas.** El organizador agrega participantes **en parejas** (dos jugadores por pareja, nombre o alias). La pareja se mantiene fija durante todo el torneo.
2. **Canchas.** El organizador crea de **1 a n canchas** (nombre o número). Pueden agregarse o desactivarse antes de iniciar.
3. **Duración de sesión.** Un único tiempo **general** para todas las rondas, configurable entre **15 y 20 minutos** (validado en backend).
4. **Ronda.** Al iniciar una ronda, el backend guarda `startsAt` y `endsAt` (`startsAt` + duración). El TV muestra la cuenta regresiva calculada en el navegador a partir de `endsAt`.
5. **Registro de resultados.** Al terminar el tiempo, el organizador registra el resultado de cada partido (puntaje de cada pareja) desde el celular.
6. **Propuesta de la siguiente ronda.** Con los resultados cargados, el sistema **propone** los cruces de la ronda siguiente: contra qué pareja juega cada una (**que no haya enfrentado antes**) y en **qué cancha**. El organizador puede aceptar o ajustar la propuesta antes de iniciar la ronda.
7. **Tabla de posiciones.** Se actualiza con cada resultado y se muestra en el TV junto con los partidos en curso (cancha, parejas, tiempo restante).

### 9.2 Reglas de emparejamiento (propuesta inicial)

El algoritmo vive en una estrategia (`PairingStrategy`) detrás de una interfaz del dominio, invocada desde un caso de uso (por ejemplo `propose-next-round.use-case.ts`). Criterios en orden de prioridad:

1. No repetir rival: dos parejas no se enfrentan de nuevo mientras existan alternativas sin repetir.
2. Descanso justo: si hay más parejas que cupos (`2 × canchas`) o cantidad impar, descansa (bye) la pareja que menos haya descansado; nadie descansa dos rondas seguidas si se puede evitar.
3. Rival de nivel parecido: entre las opciones sin repetir, se prefiere cruzar parejas cercanas en la tabla de posiciones.
4. Rotación de cancha: se prefiere una cancha distinta a la de la ronda anterior para cada pareja.
5. Si ya no quedan rivales nuevos (se jugó contra todas), se permite repetir, empezando por el enfrentamiento más antiguo.

Debe ser determinista dado el mismo estado, y probado con tests unitarios (varias parejas, cantidades impares, pocas canchas).

### 9.3 Entidades previstas

`Tournament` (estado, duración de sesión), `Team` (dos jugadores), `Court`, `Round` (número, `startsAt`, `endsAt`, estado), `Match` (ronda, cancha, pareja A y B, puntajes; o bye). Los resultados y la tabla de posiciones se derivan de `Match`.

### 9.4 Preguntas abiertas (resolver con `/speckit-clarify`)

- Sistema de puntos de la tabla: ¿victoria/empate/derrota (3-1-0) o suma de juegos/puntos anotados? ¿Criterio de desempate?
- ¿Cuántas rondas tiene el torneo: fijas, hasta que todas se hayan enfrentado, o las decide el organizador?
- ¿Qué pasa con un bye: cuenta como victoria, como empate o no puntúa?
- ¿Se puede editar un resultado ya cargado y recalcular la propuesta de la siguiente ronda?
- Número esperado de parejas y canchas (para dimensionar el algoritmo y la pantalla del TV).
- Nota de terminología: en el "americano" clásico se rotan los compañeros; aquí las parejas son fijas y rotan los rivales. Confirmar que es lo deseado.

### 9.5 Orden sugerido de specs (carpeta `specs/`)

1. `001-project-skeleton`: esqueleto actual (Fases 0-6 de este plan).
2. `002-teams-and-courts`: inscripción de parejas y alta de canchas.
3. `003-tournament-and-rounds`: torneo, duración de sesión (15-20 min), iniciar ronda con `endsAt`.
4. `004-match-results`: registro de resultados y tabla de posiciones.
5. `005-next-round-proposal`: algoritmo de emparejamiento y propuesta de cancha.
6. `006-organizer-auth`: login del organizador y protección de `/admin`.
7. `007-tv-display`: pantalla `/tv` con partidos por cancha, resultados, posiciones y cuenta regresiva.
8. Evaluar paso de polling a SSE/WebSockets si hace falta.
