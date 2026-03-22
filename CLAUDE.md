# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server at localhost:3000 (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Force reset database migrations
```

Set `ANTHROPIC_API_KEY` in `.env` — without it, the app runs with a mock LLM provider.

## Architecture

**UIGen** is a Next.js 15 (App Router) full-stack app where users describe React components in natural language and Claude generates them with live preview.

### Request Flow

1. User submits a prompt in the chat UI
2. `POST /api/chat` streams a response from Claude Haiku 4.5 (via Vercel AI SDK)
3. Claude calls two tools: `str_replace_editor` (edit files) and `file_manager` (create/delete files)
4. Tool calls update an **in-memory virtual file system** (never writes to disk)
5. The preview iframe compiles JSX at runtime using Babel Standalone and renders `/App.jsx`

### Key Modules

- **`src/lib/file-system.ts`** — Virtual file system class. Serializable to/from JSON for DB persistence. Entry point is always `/App.jsx`.
- **`src/lib/provider.ts`** — Selects real Claude API or mock provider based on `ANTHROPIC_API_KEY`.
- **`src/lib/tools/`** — `str-replace.ts` and `file-manager.ts` implement the two AI tools.
- **`src/lib/prompts/generation.tsx`** — System prompt for component generation.
- **`src/lib/auth.ts`** — JWT sessions stored in httpOnly cookies (jose + bcrypt). 7-day expiry.
- **`src/lib/contexts/`** — React contexts for file system state and chat state (using Vercel's `useChat` hook).

### Main UI (`src/app/main-content.tsx`)

Split-panel layout:
- Left 35%: Chat interface
- Right 65%: Toggle between **Preview** (live iframe) and **Code** (file tree + Monaco editor)

### Database

Schema defined in `prisma/schema.prisma` — reference it for the authoritative database structure.

SQLite via Prisma. Schema: `User` (email/password) → `Project` (name, messages JSON, data JSON). Projects are only persisted for authenticated users; anonymous work is tracked in localStorage.

### Routes

- `/` — Redirects authenticated users to their latest project (or creates one); shows auth UI otherwise
- `/[projectId]` — Project workspace
- `/api/chat` — Streaming AI endpoint (max 40 tool steps for real API, 4 for mock)
