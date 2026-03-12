# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe React components in natural language; Claude AI generates JSX code rendered live in an iframe preview panel. Authenticated users get project persistence via SQLite.

## Commands

```bash
npm run setup        # First-time setup: install deps + prisma generate + migrate
npm run dev          # Start dev server (Turbopack) at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run Vitest tests
npm run db:reset     # Reset SQLite database
```

**Run a single test file:**
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

**Environment:** Add `ANTHROPIC_API_KEY` to `.env`. Without it, the app falls back to a `MockLanguageModel` that returns static demo components.

## Architecture

### Data Flow

```
User (chat message)
  → /api/chat (route.ts)
  → streamText() via Vercel AI SDK + Claude
  → Tool calls: str_replace_editor / file_manager
  → FileSystemContext (in-memory virtual FS)
  → JSX transformer (Babel standalone) + import maps
  → PreviewFrame (sandboxed iframe)
```

### Core Modules

| Module | Path | Purpose |
|---|---|---|
| Chat API | `src/app/api/chat/route.ts` | Main AI endpoint — streams Claude responses, executes tool calls, auto-saves to DB |
| Virtual FS | `src/lib/file-system.ts` | In-memory hierarchical file system (not disk) |
| FileSystemContext | `src/lib/contexts/file-system-context.tsx` | React context managing FS state and tool call handling |
| ChatContext | `src/lib/contexts/chat-context.tsx` | Message history, streaming state, input |
| AI Provider | `src/lib/provider.ts` | Anthropic Claude or MockLanguageModel fallback |
| AI Tools | `src/lib/tools/str-replace.ts`, `file-manager.ts` | `str_replace_editor` (create/view/edit/insert) and `file_manager` (rename/delete) |
| System Prompt | `src/lib/prompts/generation.tsx` | Instructions Claude receives for component generation |
| JSX Transform | `src/lib/transform/` | Babel-based browser-side JSX transpilation + import map resolution |
| Auth | `src/lib/auth.ts`, `src/actions/index.ts` | JWT sessions (httpOnly cookies, 7-day expiry), bcrypt passwords |
| Main Layout | `src/app/main-content.tsx` | Resizable split: Chat (35%) left, Preview/Code (65%) right |

### AI Tool Calling

The `/api/chat` endpoint uses `streamText()` with max 40 steps and two tools:
- **`str_replace_editor`** — view, create, overwrite, str-replace, or insert text in virtual files
- **`file_manager`** — rename or delete virtual files

Tool results are handled client-side in `FileSystemContext`, which updates the in-memory FS and triggers preview re-renders.

### Preview iframe

The preview is a sandboxed iframe. The JSX transformer (Babel standalone) compiles component code in the browser; import maps redirect bare module specifiers (e.g. `react`, component paths) to CDN URLs via esm.sh.

### Database Schema (Prisma/SQLite)

```prisma
User    { id, email, password, createdAt, updatedAt }
Project { id, name, userId, messages (JSON), data (JSON), createdAt, updatedAt }
```

Projects store the full serialized chat history and virtual filesystem as JSON blobs. Anonymous users can generate components (optionally cached in localStorage); authenticated users get full persistence.

### State Management

No Redux/Zustand — pure React Context:
- `FileSystemProvider` wraps the app for FS state
- `ChatProvider` wraps for chat/streaming state
- Server actions handle auth and project CRUD securely

### shadcn/ui

Component library config at `components.json`. New shadcn components go in `src/components/ui/`.
