# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and an AI (Claude via Vercel AI SDK) generates React code that renders in a sandboxed iframe preview. The app works without an API key using a mock provider that returns static components.

## Commands

```bash
npm run setup          # Install deps + generate Prisma client + run migrations
npm run dev            # Dev server with Turbopack (localhost:3000)
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Vitest (all tests)
npx vitest run src/lib/__tests__/file-system.test.ts  # Single test file
npm run db:reset       # Reset SQLite database
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already configured in scripts).

## Architecture

### Core Data Flow

1. **Chat** → User sends message via `ChatProvider` (`@/lib/contexts/chat-context.tsx`) which uses Vercel AI SDK's `useChat`
2. **API** → `POST /api/chat` (`src/app/api/chat/route.ts`) streams responses using `streamText` with two AI tools: `str_replace_editor` and `file_manager`
3. **Virtual FS** → AI tool calls manipulate a `VirtualFileSystem` (`@/lib/file-system.ts`) — an in-memory file tree with no disk writes
4. **Client sync** → Tool calls are replayed client-side via `FileSystemContext.handleToolCall` to keep the client VFS in sync
5. **Preview** → `PreviewFrame` transforms all VFS files through Babel (`@/lib/transform/jsx-transformer.ts`), creates blob URLs, builds an import map, and renders in a sandboxed iframe

### Key Architectural Decisions

- **Dual VFS instances**: The server-side VFS (in the API route) and client-side VFS (in React context) are kept in sync by replaying tool calls on both sides. The server VFS is reconstructed from serialized data sent with each chat request.
- **Import map-based preview**: Generated components run in an iframe using browser-native import maps. Third-party packages resolve to `esm.sh`. Missing local imports get placeholder modules.
- **Mock provider fallback**: When `ANTHROPIC_API_KEY` is absent, `MockLanguageModel` (`@/lib/provider.ts`) returns canned components so the app works without API access.
- **Entry point convention**: The AI always creates `/App.jsx` as the root component (enforced in the system prompt at `@/lib/prompts/generation.tsx`). Preview searches for `/App.jsx`, `/App.tsx`, `/index.jsx`, etc.

### AI Tools

The AI has two tools available (defined in `src/lib/tools/`):
- **`str_replace_editor`** — view, create, str_replace, insert operations on virtual files
- **`file_manager`** — rename and delete operations

### Authentication

JWT-based with `jose`. Sessions stored in httpOnly cookies. Anonymous users can use the app without auth; authenticated users get project persistence via Prisma/SQLite. Auth logic in `src/lib/auth.ts`, middleware in `src/middleware.ts`.

### UI Components

- shadcn/ui (new-york style) with Radix primitives in `src/components/ui/`
- Monaco editor for code editing (`@monaco-editor/react`)
- Resizable panels via `react-resizable-panels`

### Project Structure

- `src/app/` — Next.js App Router pages (home redirects authenticated users to latest project)
- `src/lib/contexts/` — React contexts (`FileSystemProvider`, `ChatProvider`)
- `src/lib/transform/` — Babel JSX transformation and import map generation
- `src/lib/tools/` — AI tool definitions for the Vercel AI SDK
- `src/lib/prompts/` — System prompt for AI generation
- `src/actions/` — Server actions for project CRUD
- `prisma/` — SQLite schema and migrations

## Tech Stack

- Next.js 15 (App Router, Turbopack), React 19, TypeScript
- Tailwind CSS v4, shadcn/ui (new-york)
- Prisma with SQLite (`prisma/dev.db`)
- Vercel AI SDK (`ai` + `@ai-sdk/anthropic`), model: `claude-haiku-4-5`
- Vitest + Testing Library + jsdom for tests
- Path alias: `@/*` → `./src/*`

## Code Style

- Use comments sparingly — only where the logic isn't self-evident