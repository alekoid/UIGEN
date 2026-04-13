# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates code that runs instantly in a sandboxed iframe using a virtual file system (no disk writes).

## Commands

```bash
# First-time setup
npm run setup          # install deps + prisma generate + db migrate

# Development
npm run dev            # Next.js dev server with Turbopack

# Build & lint
npm run build
npm run lint

# Tests
npm test               # run all Vitest tests
npm test -- src/lib/__tests__/file-system.test.ts  # single file

# Database
npx prisma studio      # browse DB
npm run db:reset       # reset DB (--force, destructive)
```

## Architecture

### Request Flow

```
User chat message
  → POST /api/chat  (src/app/api/chat/route.ts)
  → Vercel AI SDK streamText with Claude
  → AI calls tools: str_replace_editor | file_manager
  → VirtualFileSystem updated in-memory
  → On finish: project + messages persisted to SQLite
  → FileSystemContext triggers re-render
  → PreviewFrame re-runs JSX transform pipeline → iframe
```

### Key Abstractions

**VirtualFileSystem** (`src/lib/file-system.ts`)
In-memory Map-based file tree. Supports CRUD + rename + list. Serializes to JSON for DB storage. Never writes to disk.

**FileSystemContext** (`src/lib/contexts/file-system-context.tsx`)
React context wrapping VirtualFileSystem. Any mutation triggers a version counter increment to force re-renders in Preview and Editor.

**ChatContext** (`src/lib/contexts/chat-context.tsx`)
Wraps Vercel AI SDK's `useChat`. Serializes current VirtualFileSystem files into the request body so the API route can reconstruct state.

**JSX Transformer** (`src/lib/transform/jsx-transformer.ts`)
Pipeline: Babel standalone → collect imports → blob URLs for each file → build import map JSON → assemble iframe HTML. React 19 and third-party packages resolve from `esm.sh`. Path alias `@/` is mapped to blob URLs.

**Claude Tools** (`src/lib/tools/`)
- `str_replace_editor`: Text-based editor tool (view, create, str_replace, insert operations)
- `file_manager`: File-level operations (move, copy, delete, list)

### Pages & Routing

- `/` — Home: unauthenticated users get MainContent; authenticated users redirect to their latest project
- `/[projectId]` — Project page: loads saved messages + file state, renders MainContent

### Auth

JWT sessions (7-day, via `jose`) stored in httpOnly cookies. Passwords hashed with bcrypt. Server actions in `src/actions/index.ts`. Middleware (`src/middleware.ts`) protects `/[projectId]` routes.

### Mock Provider

When `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` falls back to a mock that returns static component templates (Counter, Form, Card) to demonstrate multi-step tool use without an API key.

## Database

SQLite via Prisma. Two models:
- **User**: email + hashed password
- **Project**: name, optional userId (null = anonymous), messages (JSON), data (JSON serialized VirtualFileSystem)

## Testing

Vitest with jsdom. Tests cover: VirtualFileSystem, JSX transformer, FileSystemContext, ChatContext, and chat UI components. Test files live alongside source in `__tests__/` subdirectories.

## Environment

`ANTHROPIC_API_KEY` in `.env` is the only required env var for full AI functionality. Without it the mock provider activates automatically.
