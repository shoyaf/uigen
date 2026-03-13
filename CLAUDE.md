# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest
npm run db:reset     # Reset SQLite database
```

To run a single test file:
```bash
npx vitest run src/lib/transform/__tests__/jsx-transformer.test.ts
```

## Architecture

**UIGen** is an AI-powered React component generator. Users describe components in a chat interface; Claude generates them as files in a virtual (in-memory) file system; Babel transpiles JSX client-side for live preview in an iframe.

### Key Data Flow

1. User types in `ChatInterface` → sent to `/api/chat` via Vercel AI SDK `useChat()`
2. `/api/chat/route.ts` calls Claude with two tools: `str_replace_editor` (create/modify files) and `file_manager` (rename/delete)
3. Tool calls update the virtual file system, streamed back to client
4. `FileSystemProvider` receives updates → `PreviewFrame` re-renders the iframe
5. On completion, project state (messages + file system) is saved to SQLite via Prisma

### Virtual File System

All generated files live in memory — nothing is written to disk. The file system is serialized as JSON and stored in the `Project.data` column. `src/lib/file-system.ts` implements the core structure; `src/lib/contexts/file-system-context.tsx` manages React state.

### AI Provider

- **Real:** `claude-haiku-4-5` via `@ai-sdk/anthropic` (requires `ANTHROPIC_API_KEY` in `.env`)
- **Mock:** Returns static sample components when no API key is set
- Provider selection logic: `src/lib/provider.ts`
- System prompt: `src/lib/prompts/generation.tsx`
- Max 40 AI steps (4 for mock), 10,000 token limit

### JSX Transformation

`src/lib/transform/jsx-transformer.ts` uses `@babel/standalone` to transpile TypeScript/JSX client-side. It strips CSS imports and creates placeholder modules for missing imports, generating blob URLs for ES module loading in the preview iframe.

### Authentication

JWT tokens in HTTP-only cookies (7-day expiry) via `jose`. `src/lib/auth.ts` handles session creation/validation. Server actions in `src/actions/` handle sign-up/sign-in/sign-out. Anonymous users can generate components but projects aren't persisted.

### Database

SQLite via Prisma. Schema: `User` (id, email, password) → `Project` (id, name, userId, messages JSON, data JSON). Prisma client is generated to `src/generated/prisma/`.

### Path Alias

`@/*` maps to `src/*` throughout the codebase.
