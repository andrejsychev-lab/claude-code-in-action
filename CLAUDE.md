# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run Vitest test suite
npm run db:reset     # Reset database (destructive)
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

Environment: copy `.env` and add `ANTHROPIC_API_KEY` (optional — falls back to MockLanguageModel generating demo components).

## Architecture

UIGen is an AI-powered React component generator with a live preview. The core innovation is a **virtual file system** (in-memory, no disk writes) that the AI manipulates through tools, with client-side JSX transformation for live preview.

### Data Flow

```
User message → /api/chat (streaming) → Claude (claude-haiku-4-5) or MockLanguageModel
    → AI calls str_replace_editor / file_manager tools
    → FileSystemContext receives tool results, updates VirtualFileSystem
    → PreviewFrame re-renders (Babel transpiles JSX → iframe srcdoc)
    → Project auto-saved to SQLite via Prisma (authenticated users)
```

### Key Modules

- **`src/lib/file-system.ts`** — In-memory virtual file system. Core data structure mapping paths to `FileNode` objects. Serialized as JSON for database persistence.
- **`src/lib/tools/str-replace.ts`** — AI editor tool implementing `view`, `create`, `str_replace`, `insert`, `undo_edit` commands on the virtual FS.
- **`src/lib/tools/file-manager.ts`** — AI tool for directory operations.
- **`src/lib/transform/jsx-transformer.ts`** — Client-side Babel transpilation (`@babel/standalone`) that converts virtual FS files into an iframe `srcdoc` with import maps.
- **`src/lib/provider.ts`** — LLM initialization; uses Anthropic claude-haiku-4-5 or `MockLanguageModel` (no API key).
- **`src/lib/prompts/generation.tsx`** — System prompt for component generation.
- **`src/lib/auth.ts`** — JWT-based auth with `jose`; sessions stored in HttpOnly cookies (7-day expiry).

### Context / State

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — Owns virtual FS state; processes AI tool calls; triggers preview refresh.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — Wraps Vercel AI SDK's `useChat`; manages messages and submission.

### Component Structure

- `src/app/[projectId]/page.tsx` — Project page; main workspace
- `src/app/main-content.tsx` — Resizable panel layout (chat | preview/code editor)
- `src/components/chat/` — Chat UI (ChatInterface, MessageList, MessageInput, MarkdownRenderer)
- `src/components/editor/` — FileTree + Monaco-based CodeEditor
- `src/components/preview/PreviewFrame.tsx` — Iframe preview with hot reload
- `src/components/ui/` — Radix UI wrappers (shadcn/ui pattern)

### Database

The database schema is defined in the `prisma/schema.prisma` file. Reference it anytime you need to understand the structure of data stored in the database.

SQLite via Prisma. Two models:
- **User**: email, bcrypt-hashed password, projects relation
- **Project**: name, userId, `messages` (JSON), `data` (JSON for serialized file system)

Anonymous work is tracked in localStorage via `src/lib/anon-work-tracker.ts` and can be claimed on sign-up.

### Path Alias

`@/*` maps to `src/*` (configured in `tsconfig.json`).
