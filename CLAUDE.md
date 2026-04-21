# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run Vitest
npm run db:reset     # Reset and re-migrate the database
```

To run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Environment

- `ANTHROPIC_API_KEY` — Required for real AI generation; omit to use the built-in mock provider (demo mode)
- `JWT_SECRET` — Defaults to `"development-secret-key"` if unset

## Code style

- Use comments sparingly. Only comment complex code.

## Architecture

UIGen is a Next.js 15 (App Router) application that generates React components via Claude AI and renders them in a sandboxed live preview.

### Data flow

```
User prompt → ChatInterface → useChat (AI SDK) → POST /api/chat
  → streamText with tools → tool calls update FileSystemContext
  → PreviewFrame re-renders the iframe
```

### Virtual file system

All generated files live in a client-side `VirtualFileSystem` class (`src/lib/file-system.ts`) — nothing is written to disk. The `FileSystemProvider` context owns this state and exposes file operations to all components.

### AI tool-calling

`/api/chat/route.ts` uses Vercel AI SDK's `streamText` with two tools:
- `str_replace_editor` (`src/lib/tools/`) — creates files, views files, does string-replace edits, line insertions
- `file_manager` — renames and deletes files/directories

File mutations flow: tool call → server action → `FileSystemContext` update.

### Live preview

`PreviewFrame` (`src/components/preview/`) renders an `<iframe>` with a generated HTML document. `jsx-transformer.ts` compiles JSX at runtime via Babel standalone and builds an import map pointing to esm.sh CDN for React and other packages. No build step for generated code.

### Authentication & persistence

JWT sessions are stored in cookies (`src/lib/auth.ts`). Authenticated users persist projects (messages + file system state as JSON) in SQLite via Prisma. Anonymous users have in-session tracking via `anon-work-tracker`.

### Mock provider

When `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` returns a `MockLanguageModel` that replays a fixed 4-step generation sequence — useful for development without API costs.

### Key paths

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | AI streaming endpoint (120s timeout) |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/lib/contexts/` | FileSystemContext, ChatContext |
| `src/lib/tools/` | AI tool definitions |
| `src/lib/transform/` | JSX compiler + import map generator |
| `src/lib/prompts/` | System prompt for component generation |
| `src/actions/` | Server actions for auth and project CRUD |
| `prisma/schema.prisma` | Source of truth for all database models — reference this to understand stored data structure |

### Database models

Defined in `prisma/schema.prisma`.

**User**

| Field | Type | Notes |
|-------|------|-------|
| `id` | `String` | CUID, primary key |
| `email` | `String` | Unique |
| `password` | `String` | |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |
| `projects` | `Project[]` | Relation |

**Project**

| Field | Type | Notes |
|-------|------|-------|
| `id` | `String` | CUID, primary key |
| `name` | `String` | |
| `userId` | `String?` | Optional — null for anonymous projects |
| `messages` | `String` | JSON array, default `"[]"` |
| `data` | `String` | JSON object (file system state), default `"{}"` |
| `createdAt` | `DateTime` | |
| `updatedAt` | `DateTime` | |

### Path alias

`@/*` maps to `src/*` (configured in `tsconfig.json`).
