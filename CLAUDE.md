# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run start        # Start production server
npm run lint         # ESLint
npm run test         # Run all tests with Vitest
npm run setup        # First-time setup: install + Prisma generate + migrate
npm run db:reset     # Reset database (destructive)
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`

## Architecture

**UIGen** is an AI-powered React component generator with live preview. Users describe components in natural language and Claude generates code in real-time.

### Core Data Flow

1. User sends message in `ChatInterface`
2. `chat-context.tsx` calls `/api/chat/route.ts` via Vercel AI SDK streaming
3. The API sends messages + current virtual file system state to Claude (claude-haiku-4-5)
4. Claude calls tools (`str_replace_editor`, `file_manager`) to create/modify files
5. Tool results update the virtual file system in `file-system-context.tsx`
6. `PreviewFrame` re-renders the component via Babel + esm.sh import maps in an iframe

### Virtual File System (VFS)

All files are in-memory — no disk writes. `src/lib/file-system.ts` manages the tree structure. `src/lib/contexts/file-system-context.tsx` exposes it to React and handles AI tool calls. The VFS serializes to JSON for persistence in the database.

### AI Integration

- `src/app/api/chat/route.ts` — streaming endpoint with tool calling
- `src/lib/provider.ts` — selects real Anthropic provider or mock (when `ANTHROPIC_API_KEY` is absent)
- `src/lib/tools/` — tool definitions for `str_replace_editor` and `file_manager`
- `src/lib/prompts/generation.tsx` — system prompt

### Live Preview

`src/components/preview/PreviewFrame.tsx` renders an iframe where:
- Babel standalone transforms JSX at runtime
- `src/lib/transform/jsx-transformer.ts` generates an import map (local files → blob URLs, npm packages → esm.sh)
- CSS imports are stripped before Babel processing
- Missing imports get placeholder components

### Auth & Persistence

- JWT tokens in httpOnly cookies (7-day expiry), using `jose` + `bcrypt`
- `src/middleware.ts` verifies sessions for API routes
- `src/actions/` — server actions for auth (signUp, signIn, signOut, getUser) and project CRUD
- Prisma + SQLite: `User` and `Project` models (messages and file data stored as JSON strings)
- Anonymous mode works fully without auth; projects are ephemeral

### Key Files

| File | Purpose |
|------|---------|
| `src/lib/file-system.ts` | VFS core — in-memory file tree with CRUD |
| `src/app/api/chat/route.ts` | Streaming AI endpoint with tool calling |
| `src/lib/contexts/file-system-context.tsx` | VFS React state + AI tool dispatch |
| `src/lib/contexts/chat-context.tsx` | Chat state using Vercel AI SDK `useChat` |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX transform + import map |
| `src/components/preview/PreviewFrame.tsx` | iframe rendering logic |
| `src/lib/provider.ts` | Anthropic provider or mock fallback |
| `prisma/schema.prisma` | Data models |

## Environment

Copy `.env.example` to `.env`. `ANTHROPIC_API_KEY` is optional — without it the app uses a mock provider that returns static code.

After cloning: `npm run setup` (runs install + `prisma generate` + `prisma migrate dev`).
