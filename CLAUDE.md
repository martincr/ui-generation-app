# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server with Turbopack at http://localhost:3000
npm run build          # Production build
npm run lint           # ESLint via Next.js
npm test               # Run all tests with Vitest
npm run db:reset       # Reset SQLite database (destructive)
npx prisma migrate dev # Run new migrations after schema changes
npx prisma generate    # Regenerate Prisma client after schema changes
```

To run a single test file:
```bash
npx vitest src/lib/__tests__/file-system.test.ts
```

## Environment

Copy `.env.example` to `.env` (or create `.env`) and set:
```
ANTHROPIC_API_KEY=sk-ant-...   # Optional — app falls back to a mock provider if absent
JWT_SECRET=...                  # Required for auth
```

The `node-compat.cjs` shim is required (auto-applied via `NODE_OPTIONS` in all npm scripts) to polyfill Node.js built-ins for some dependencies.

## Architecture

**UIGen** is an AI-powered React component generator. Users describe components in chat, Claude generates code, and a live preview renders the result — all in a split-panel UI.

### Key Data Flow

1. **Chat** → `POST /api/chat` → `streamText()` with two Claude tools → streaming response
2. **Claude tools** mutate the **virtual file system** (in-memory, never touches disk)
3. **Preview** reads from VFS, transforms JSX via Babel standalone, renders in a sandboxed `<iframe>`
4. **Persistence** serializes VFS + message history as JSON strings in the `Project` model (SQLite via Prisma)

### Virtual File System (`src/lib/file-system.ts`)

The `VirtualFileSystem` class is the central data structure. All file operations go through it. It supports the full CRUD needed for code generation: `createFile`, `updateFile`, `deleteFile`, `renameFile`, `viewFile`, `replaceInFile`, `insertInFile`. It serializes to/from JSON for database storage.

### AI Integration (`src/app/api/chat/route.ts`, `src/lib/provider.ts`)

- Model: Claude Haiku 4.5 via `@ai-sdk/anthropic`
- Two tools exposed to Claude:
  - `str_replace_editor` — file viewing, creation, string replacement, line insertion
  - `file_manager` — file management operations
- Tool implementations are in `src/lib/tools/`
- System prompt is in `src/lib/prompts/generation.tsx`
- When `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` returns a mock provider

### State Management

Two React Contexts (no external state library):
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps `useChat` from Vercel AI SDK; owns message history and submission
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns VFS state and exposes file operations to components

### Live Preview (`src/components/preview/PreviewFrame.tsx`)

Renders an `<iframe>` with:
- Import maps for module resolution (React, etc.)
- Babel standalone for JSX → JS transformation at runtime
- Dynamic entry point detection (`App.jsx`, `App.tsx`, `index.jsx`, etc.)

### Auth (`src/lib/auth.ts`, `src/actions/index.ts`)

JWT-based sessions stored in HTTP-only cookies (7-day expiry). Passwords hashed with bcrypt. The `src/middleware.ts` guards routes. Anonymous use is supported — projects can exist without a user (`userId` is optional in the schema).

### Database Schema (`prisma/schema.prisma`)

SQLite with two models:
- `User` — email/password auth
- `Project` — stores `messages` (JSON array) and `data` (JSON VFS snapshot), both as plain strings

Prisma client is generated into `src/generated/prisma/`.

### Path Alias

`@/*` resolves to `src/*` (configured in `tsconfig.json` and `components.json`).
