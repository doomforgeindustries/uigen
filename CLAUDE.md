# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack (http://localhost:3000)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run Vitest test suite
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Architecture

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude generates code via tool calls that mutate an in-memory virtual file system, which is then previewed in a sandboxed iframe via Babel + import maps.

### Key Data Flow

1. User sends message → `POST /api/chat` (streaming via Vercel AI SDK `streamText`)
2. AI calls `str_replace_editor` and `file_manager` tools to create/modify files
3. Tool results update `VirtualFileSystem` (in-memory Map tree, no disk I/O)
4. `FileSystemContext` propagates changes → `PreviewFrame` detects new file content
5. `jsx-transformer.ts` uses `@babel/standalone` to transform JSX → JS, builds an import map pointing `react`, `react-dom`, etc. to `esm.sh`
6. Blob URLs are created for each file and loaded in a sandboxed `<iframe>`
7. On success, project (messages + serialized VFS) is persisted to SQLite via Prisma server action

### State Management

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): Wraps `VirtualFileSystem`, exposes file CRUD, selected file, and refresh trigger
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`, integrates tool call results into the file system, sends VFS state with each request

### AI Provider

`src/lib/provider.ts` — if `ANTHROPIC_API_KEY` is set, uses Claude Haiku 4.5; otherwise falls back to `MockLanguageModel` (generates sample components, no API key needed). The system prompt lives in `src/lib/prompts/generation.tsx`.

### AI Tools

- `str_replace_editor` (`src/lib/tools/str-replace.ts`): Precise string-based file editing (create, overwrite, str_replace, view)
- `file_manager` (`src/lib/tools/file-manager.ts`): Rename and delete files

### Database

SQLite via Prisma. Two models: `User` (email + bcrypt password) and `Project` (stores `messages` and `data` as JSON strings). Anonymous users get projects persisted in browser storage only.

### Authentication

JWT in HTTP-only cookies (7-day expiry). `src/lib/auth.ts` handles token creation/verification using `jose`. Server actions are in `src/actions/index.ts`. `src/middleware.ts` protects routes.

### Preview System

`src/components/preview/PreviewFrame.tsx` — re-runs the full transform pipeline on any VFS change. Entry point auto-detected (`/App.jsx`, `/App.tsx`, etc.). Iframe sandbox flags: `allow-scripts allow-same-origin allow-forms`.

## Path Alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).

## Tailwind & UI

Tailwind CSS v4. shadcn/ui components live in `src/components/ui/` with the "new-york" style preset. Add new shadcn components via `npx shadcn@latest add <component>`.
