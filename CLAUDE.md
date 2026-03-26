# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in chat; Claude generates React + Tailwind code using a virtual file system; the preview renders via Babel in an iframe.

## Commands

```bash
# First-time setup (install deps + Prisma generate + migrate)
npm run setup

# Development server (Turbopack, http://localhost:3000)
npm run dev

# Run tests (Vitest + jsdom)
npm run test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Lint
npm run lint

# Production build
npm run build

# Reset database (destructive)
npm run db:reset
```

## Environment

Create `.env` with:
```
ANTHROPIC_API_KEY=your-key   # optional — omit to use MockLanguageModel
JWT_SECRET=your-secret       # optional — defaults to dev value
```

## Architecture

### Data Flow

1. User submits a chat message in `ChatInterface`
2. `chat-context.tsx` sends request to `/api/chat/route.ts`
3. The route invokes Claude (`claude-haiku-4-5`) or `MockLanguageModel` (via `src/lib/provider.ts`) with tools
4. Claude calls `str_replace` / `file_manager` tools to create/edit files in the virtual filesystem
5. `file-system-context.tsx` updates in-memory state; `PreviewFrame` re-renders `App.jsx` via Babel standalone
6. On completion, messages and file system are persisted to SQLite via `Project.data` (JSON)

### Virtual File System

`src/lib/file-system.ts` — pure in-memory `VirtualFileSystem` class, no disk I/O. Serialized to JSON and stored in `Project.data`. AI tools interact with this abstraction.

### AI Tooling

- `src/lib/tools/str-replace.ts` — view/create/edit files (primary tool)
- `src/lib/tools/file-manager.ts` — rename/delete files
- `src/lib/prompts/generation.tsx` — system prompt instructing Claude to generate React + Tailwind components

### State Management

- `src/lib/contexts/chat-context.tsx` — chat messages, input state (Vercel AI SDK)
- `src/lib/contexts/file-system-context.tsx` — virtual filesystem state
- `src/lib/anon-work-tracker.ts` — localStorage tracker for anonymous users

### Authentication

JWT-based (7-day expiry, HTTP-only cookies). Server actions in `src/actions/index.ts` handle signUp/signIn/signOut. `src/middleware.ts` protects routes. `src/lib/auth.ts` provides session helpers.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models:
- `User`: id, email, password, timestamps
- `Project`: id, name, optional userId, `messages` (JSON), `data` (JSON serialized VirtualFileSystem)

### Key Directories

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | Chat API — AI invocation + tool execution |
| `src/lib/provider.ts` | Claude vs Mock model selection |
| `src/lib/file-system.ts` | VirtualFileSystem implementation |
| `src/lib/contexts/` | React context providers |
| `src/lib/tools/` | AI tool definitions |
| `src/lib/prompts/` | Claude system prompt |
| `src/components/preview/PreviewFrame.tsx` | Babel-powered live preview iframe |
| `src/actions/` | Next.js Server Actions (auth + project CRUD) |
| `prisma/` | Schema and migrations |

### Path Alias

`@/*` maps to `src/*` throughout the project.
