# Workspace

## Overview

pnpm workspace monorepo using TypeScript. Each package manages its own dependencies.

This monorepo hosts **Xsus AI** — a production multi-user AI chat web app
(ChatGPT/Perplexity-like) with auth, chat history, streaming responses,
image+vision, web search, and dark mode. Backed by an Express API server and
shared workspace libraries.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec)
- **Build**: esbuild (CJS bundle)
- **Auth**: Clerk (email/password + Google) — `@clerk/express` server,
  `@clerk/react` client. API requests proxied through frontend with
  cookie session (no Bearer needed) via path-based routing.
- **AI**: Replit AI Integrations OpenAI proxy (model `gpt-5.4` for chat,
  `gpt-5-nano` for auto-titles).
- **Web search**: DuckDuckGo Instant Answer API (no API key).
- **Object Storage**: Replit App Storage for image uploads (vision input).
- **Frontend**: React 19 + Vite 7 + Tailwind v4 + shadcn/ui + wouter v3 +
  TanStack Query v5 + react-markdown + highlight.js.

## Artifacts

- `artifacts/api-server` (kind: api, mounted at `/api`, port 8080)
  Express server with routes `/me`, `/chats`, `/chats/:id/messages` (SSE),
  `/storage`. See `artifacts/api-server/src`.
- `artifacts/xsus-ai` (kind: web, mounted at `/`, port 20205)
  Main user-facing web app. Pages: landing, sign-in, sign-up, chat, settings.
- `artifacts/mockup-sandbox` (kind: design) — design playground (not started by default).

## Key Commands

- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- `pnpm --filter @workspace/api-server run dev` — run API server locally
- `pnpm --filter @workspace/xsus-ai run dev` — run web app locally

## Required Environment

Secrets (already configured in this Repl):

- `DATABASE_URL` — Postgres connection string
- `CLERK_SECRET_KEY`, `VITE_CLERK_PUBLISHABLE_KEY` — Clerk auth
- `AI_INTEGRATIONS_OPENAI_*` — OpenAI proxy
- `DEFAULT_OBJECT_STORAGE_BUCKET_ID`, `PRIVATE_OBJECT_DIR`,
  `PUBLIC_OBJECT_SEARCH_PATHS` — Object Storage
- `SESSION_SECRET` — session signing

## Database Schema

`lib/db/src/schema/`:
- `users` — Clerk user id, email, displayName, avatarUrl, **city, country, formattedAddress, latitude, longitude, timezone**
- `chats` (formerly `conversations`) — userId fk, title, pinned, archived
- `messages` — chatId fk, role, content, imageUrl, sources jsonb

## Feature Modules

- **Weather + Time bar** — `WeatherBar.tsx` polls `/api/weather?lat&lon`
  (Open-Meteo, no key, server-side 10-min cache). Emoji + temp + day/time
  in user's timezone. Shows above the chat header once a location is set.
- **3D Earth location picker** — `EarthModal.tsx` lazy-loads `globe.gl`
  (Three.js wrapper). Click to pick coords (reverse-geocoded via
  Nominatim) or search by name (`/api/geocode`, Open-Meteo geocoding).
  Uses `navigator.geolocation` for "use my device location". Saves to
  `/api/user/location`. Location is included as a system message context
  in every chat completion so answers are timezone/region-aware.
- **AI Presentation Generator** — `PresentationModal.tsx` calls
  `/api/generate-presentation/outline` (JSON outline preview) then
  `/api/generate-presentation` (returns `.pptx` blob built with
  `pptxgenjs`, optional AI image generation via `gpt-image-1`).
  Family-safe slide image prompt prefix is enforced server-side.
- **Speed** — `compression` middleware on Express (skipped for SSE so
  chunks flush instantly), SSE primer write to defeat proxy buffering,
  history trimmed to last 20 messages before sending to OpenAI.

## Frontend Notes

- `BASE_URL` is `/` for the web artifact. Routes are wouter v3.
- Sign-in/up routes use both exact (`/sign-in`) and wildcard (`/sign-in/*?`)
  routes so Clerk's nested factor pages work.
- All API calls go through `apiFetch` with `credentials: "include"` (cookie
  session, no Bearer token needed because the API is mounted on the same
  origin behind a path-based proxy).
- Image uploads: client gets a presigned PUT URL, uploads, then converts the
  returned `objectPath` to a public URL the OpenAI vision endpoint can fetch.
- SSE streaming uses native `fetch` + `ReadableStream` with a `data:` parser.

See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details.
