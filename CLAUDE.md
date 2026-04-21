# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator. Users describe components in natural language via a chat interface; Claude generates code that renders live in a sandboxed iFrame preview alongside a Monaco code editor.

## Commands

```bash
npm run setup          # First-time setup: install deps, Prisma generate, run migrations
npm run dev            # Start dev server with Turbopack on port 3000
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Run all Vitest tests
npm run db:reset       # Reset SQLite database

# Run a single test file
npx vitest run src/components/__tests__/ChatInterface.test.tsx
```

Environment variables are in `.env`. The app works without `ANTHROPIC_API_KEY` — it falls back to a `MockLanguageModel`.

## Architecture

### Request Flow

1. User submits a message in `ChatInterface` → `MessageInput` POSTs to `/api/chat`
2. The request includes the full chat history plus the serialized `VirtualFileSystem` (a Map of file paths to content, JSON-stringified)
3. `src/app/api/chat/route.ts` reconstructs the VirtualFileSystem, applies a system prompt with ephemeral prompt cache, and calls Claude (`claude-haiku-4-5`) via Vercel AI SDK
4. Claude streams responses and invokes two tools: `str_replace_editor` (create/edit/view files) and `file_manager` (rename/delete)
5. Tool call results are handled in `FileSystemContext.handleToolCall`, updating the in-memory VirtualFileSystem
6. `PreviewFrame` detects file changes and re-transpiles JSX using Babel Standalone with a dynamic import map (blob URIs), rendering into a sandboxed iFrame
7. On stream finish, if the user is authenticated, the project (messages + serialized files) is saved to SQLite via Prisma

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`) — All files live in memory as a `Map<string, FileNode>`. There is no disk I/O for user files. The FS is serialized to JSON when sent to the API and reconstructed on the server side.

**Language Model Selection** (`src/lib/provider.ts`) — Returns Claude or `MockLanguageModel` depending on whether `ANTHROPIC_API_KEY` is set. Mock is used during development/tests.

**JSX Transform** (`src/lib/transform/jsx-transformer.ts`) — Converts the virtual file tree into a runnable browser page using Babel Standalone. Generates blob URIs for each file as ES modules and assembles an HTML import map for the iFrame.

**Context Providers** — `FileSystemContext` owns the virtual FS state and tool call handler; `ChatContext` wraps Vercel AI SDK's `useChat` hook. Both are provided in `src/app/main-content.tsx`.

### Auth & Persistence

JWT tokens in httpOnly cookies (7-day TTL). Passwords hashed with bcrypt. Anonymous users get full functionality but no persistence. The Prisma schema (`prisma/schema.prisma`) has `User` and `Project` models; `Project.data` stores the serialized virtual FS as a JSON string.

### Routing

- `/` — Home; redirects authenticated users to their first project or creates one
- `/[projectId]` — Loads an existing project and passes data to `MainContent`
- `/api/chat` — Streaming POST endpoint for AI generation
- `/api/auth/*` — Sign-in, sign-up, sign-out, session check

## Path Aliases

`@/*` maps to `src/*` (configured in `tsconfig.json`). Generated components use `@/` imports, which the JSX transformer resolves against the virtual FS.

## Testing

Tests live in `src/components/__tests__/` and `src/lib/__tests__/`. Vitest runs in a jsdom environment. The `MockLanguageModel` and a mock `VirtualFileSystem` are used throughout tests — no real API calls or disk access.

## Code Style

Use comments sparingly. Only comment complex code.
