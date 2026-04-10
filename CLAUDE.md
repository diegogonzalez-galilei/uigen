# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server with Turbopack
npm run dev:daemon     # Start dev server in background (logs to logs.txt)
npm run build          # Production build
npm run lint           # ESLint via Next.js
npm run test           # Run all tests with Vitest
npm run db:reset       # Reset SQLite database to initial state
```

To run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

Environment: copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. Without it, the app falls back to `MockLanguageModel` (returns static examples).

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates and edits code via tool calls; a browser iframe renders the output in real-time.

### Request Flow

1. User sends a message in `ChatInterface`
2. POST to `/api/chat` (src/app/api/chat/route.ts)
3. Server reconstructs `VirtualFileSystem` from serialized project data stored in SQLite
4. Streams Claude completion with two tools: `str_replace_editor` and `file_manager`
5. On completion, persists updated files + messages back to the `Project` row
6. Client updates the in-memory `VirtualFileSystem` via `FileSystemContext`

### Virtual File System

`VirtualFileSystem` (src/lib/file-system.ts) is a pure in-memory file store — no disk I/O. It serializes/deserializes to JSON for database persistence. The React context (`FileSystemContext`) provides it to all components.

### AI Tools

The app uses the **Vercel AI SDK** (`ai`, `@ai-sdk/anthropic`, `@ai-sdk/react`) — not the raw Anthropic SDK. The active model is `claude-haiku-4-5` (set in `src/lib/provider.ts`).

Two tools are given to Claude at inference time:
- `str_replace_editor` (src/lib/tools/str-replace.ts) — view/create/edit files via string replacement
- `file_manager` (src/lib/tools/file-manager.ts) — rename or delete files

The system prompt lives in `src/lib/prompts/generation.tsx`.

Client-side tool calls are intercepted via `onToolCall` in `ChatContext` → `FileSystemContext.handleToolCall`, which applies changes to the in-memory VFS so the preview updates in real-time without waiting for the server round-trip to complete.

### Live Preview

`PreviewFrame` (src/components/preview/PreviewFrame.tsx) renders an iframe sandbox. `jsx-transformer.ts` (src/lib/transform/) uses Babel standalone to transform JSX → JS in the browser, builds an import map, and assembles the full HTML string injected into the iframe. Auto-detects entry points (App.jsx, index.jsx, etc.).

### Authentication

JWT sessions via `jose`, stored in cookies. `src/lib/auth.ts` is `server-only`. Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD. `src/middleware.ts` verifies JWTs on protected routes. Anonymous users get localStorage-backed work tracking via `anon-work-tracker.ts`.

### Database

Prisma with SQLite. Client singleton at `src/lib/prisma.ts`. The Prisma client is generated to `src/generated/prisma` (non-default). Always reference `prisma/schema.prisma` for the authoritative database structure.

### UI Layout

Three-panel layout in `src/app/main-content.tsx` using `@radix-ui/react-resizable`:
- **Left**: Chat (ChatInterface + MessageList + MessageInput)
- **Right**: Tabs — Preview (iframe) or Code (FileTree + Monaco editor)

`src/components/ui/` contains shadcn/ui primitives (New York style, Lucide icons, CSS variables).

## Path Aliases

`@/*` maps to `src/*` (configured in `tsconfig.json`).
