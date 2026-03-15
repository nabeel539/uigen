# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (uses Turbopack)
npm run dev

# Build
npm build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database
npm run db:reset
```

Set `ANTHROPIC_API_KEY` in `.env` to enable real AI generation. Without it, a `MockLanguageModel` is used that returns static component examples.

## Architecture

### High-Level Flow

1. User sends a message via `ChatInterface` → `ChatProvider` (uses Vercel AI SDK `useChat`) → `POST /api/chat`
2. The API route constructs a `VirtualFileSystem` from the serialized state sent by the client, then calls `streamText` with two tools: `str_replace_editor` and `file_manager`
3. As the AI streams tool calls back, `onToolCall` in `ChatProvider` dispatches them to `FileSystemContext.handleToolCall`, which mutates the in-memory `VirtualFileSystem` and triggers a refresh
4. `PreviewFrame` reacts to `refreshTrigger`, compiles all VFS files via Babel (`@babel/standalone`) into blob URLs, builds an import map, and injects everything into a sandboxed `<iframe>` as `srcdoc`

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` is an in-memory tree stored as a flat `Map<string, FileNode>`. All state lives in the browser; nothing is written to disk. For authenticated users, the serialized FS (`fileSystem.serialize()`) is persisted to the `Project.data` JSON column in SQLite after each AI turn.

### Contexts

- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the `VirtualFileSystem` instance and exposes file CRUD + `handleToolCall` for AI-driven mutations
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK `useChat`, wires `onToolCall` to `FileSystemContext.handleToolCall`, and forwards `projectId` + serialized files to the API

### AI Tools (server-side)

Defined in `src/lib/tools/` and registered in `/api/chat/route.ts`:
- `str_replace_editor` — create/str_replace/insert operations on the virtual FS
- `file_manager` — rename/delete operations

The AI is instructed (via `src/lib/prompts/generation.tsx`) to always create `/App.jsx` as the root entry point and use `@/` import aliases for local files.

### Preview Pipeline

`PreviewFrame` → `createImportMap()` → Babel transforms each `.jsx/.tsx` file to a blob URL → builds native ES module import map → `createPreviewHTML()` injects it all into an `<iframe srcdoc>` with Tailwind CDN loaded. Third-party npm packages are resolved via `esm.sh`. Missing local imports get auto-generated placeholder modules.

### Auth

Custom JWT auth (`src/lib/auth.ts`) using `jose`. Sessions stored in an httpOnly cookie (`auth-token`, 7-day expiry). Middleware (`src/middleware.ts`) only protects `/api/projects` and `/api/filesystem` routes. The `/api/chat` route checks auth itself to decide whether to persist project data.

Anonymous users can work freely; their state is tracked via `src/lib/anon-work-tracker.ts` (localStorage) and offered for migration on sign-up.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email/password with bcrypt) and `Project` (stores `messages` and `data` as JSON strings). Prisma client is generated to `src/generated/prisma`.

### Testing

Vitest with jsdom + `@testing-library/react`. Tests live alongside source in `__tests__/` subdirectories. The `vite-tsconfig-paths` plugin makes `@/` aliases work in tests.

### Provider Selection

`src/lib/provider.ts` exports `getLanguageModel()` — returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that streams static component demos.

