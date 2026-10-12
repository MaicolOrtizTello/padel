# Frontend — Americano

SPA del gestor de torneos "americano". React + Vite + TypeScript, Redux Toolkit (RTK Query), Tailwind CSS y CSS Modules.

- `/tv`: pantalla pública de solo lectura para el TV del club (resultados, cancha, tiempo restante).
- `/admin`: panel protegido para que el organizador cargue resultados desde el celular.

> Estado: estructura preparada, aún sin código. El proyecto Vite se genera en la Fase 3 de [`plan.md`](../plan.md). Reglas de arquitectura: [`.specify/memory/constitution.md`](../.specify/memory/constitution.md).

## Estructura

```
frontend/src/
├── main.tsx
├── app/            # App.tsx, router.tsx, store.ts, hooks.ts (useAppDispatch/useAppSelector)
├── pages/          # una por ruta; solo componen features (TvPage.tsx, AdminPage.tsx)
├── features/
│   └── <feature>/  # health ahora; tv y admin después
│       ├── api/        # endpoints RTK Query (injectEndpoints)
│       ├── components/ # componentes de la feature
│       ├── <feature>Slice.ts  # estado de UI, si hace falta
│       └── types.ts
├── shared/
│   ├── api/        # baseApi.ts (createApi + fetchBaseQuery con base /api)
│   ├── components/ # Button, Card, Layout…
│   └── lib/        # utilidades puras
└── styles/         # index.css: @import "tailwindcss" + @theme (tokens)
```

## Convenciones

- Archivos en inglés: componentes y páginas `PascalCase.tsx`; hooks `useXxx.ts`; utilidades `camelCase.ts`; tests `*.test.tsx` junto al archivo.
- **Estado:** datos del servidor con RTK Query (el polling del TV usa `pollingInterval`); estado de UI con slices de Redux Toolkit.
- Los componentes solo renderizan: nunca llaman `fetch` ni tienen lógica de negocio.
- **Estilos:** Tailwind por defecto. CSS Module (`Component.module.css`, junto al componente) solo para animaciones, selectores de estado complejos o utilidades ilegibles. Los tokens de diseño se definen una vez en `styles/index.css` (`@theme`). Clases condicionales con `clsx`.
- Pantalla del TV: fondo oscuro y tipografía grande.

## Desarrollo (cuando exista el proyecto)

```bash
cd frontend
npm install
npm run dev          # http://localhost:5173 — /api se proxea a http://localhost:3000
```

Requiere el backend corriendo (ver [`backend/README.md`](../backend/README.md)).

## Scripts previstos

| Comando | Qué hace |
|---|---|
| `npm run dev` | Servidor de desarrollo con hot reload |
| `npm run build` | Build estático a `dist/` |
| `npm test` | Tests (Vitest + Testing Library + MSW) |
| `npm run lint` | Lint |
| `npm run typecheck` | Verificación de tipos |

## Producción

`frontend/Dockerfile` construye con Vite y sirve `dist/` con Caddy, que también hace proxy de `/api/*` a la API y usa `try_files` para que React Router funcione al recargar.
