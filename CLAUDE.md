# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server (Turbopack) at http://localhost:3000
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Run all tests with Vitest
npm run db:reset    # Reset the SQLite database (destructive)
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

Note: `NODE_OPTIONS='--require ./node-compat.cjs'` is automatically prepended to `dev`, `build`, and `start` scripts via `package.json`. This is required for Next.js to work correctly here — do not remove it.

## Architecture

### Overview

UIGen is a Next.js 15 (App Router) web app where users describe React components in a chat interface and an AI generates them in real time. Generated code lives in an **in-memory virtual file system** and is previewed live in a sandboxed iframe — no files are ever written to disk.

### Core Data Flow

1. User sends a message → **ChatContext** (`src/lib/contexts/chat-context.tsx`) forwards it to `/api/chat` along with the current serialized VFS state.
2. `/api/chat/route.ts` runs `streamText` (Vercel AI SDK) with two tools: `str_replace_editor` and `file_manager`.
3. As the LLM emits tool calls, `onToolCall` in `ChatContext` fires → **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`) applies the mutation to the in-memory `VirtualFileSystem`.
4. Every VFS mutation triggers a `refreshTrigger` counter increment → **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`) detects the change, transpiles all files with Babel, builds an ES module import map (with blob URLs), and sets `iframe.srcdoc` to the generated HTML.

### Key Modules

- **`src/lib/file-system.ts`** — `VirtualFileSystem` class: in-memory tree of `FileNode` objects. Has `serialize()`/`deserializeFromNodes()` for JSON round-trips. Supports create, read, update, delete, rename, and text-editor operations (`replaceInFile`, `insertInFile`, `viewFile`).

- **`src/lib/transform/jsx-transformer.ts`** — Babel-based JSX/TSX transpiler. `createImportMap()` transforms all files in the VFS into blob URLs and builds an ES importmap. Third-party packages are resolved via `https://esm.sh/`. The preview HTML includes Tailwind CSS from CDN.

- **`src/lib/tools/`** — Vercel AI SDK tool builders (`str_replace_editor`, `file_manager`) that wrap the `VirtualFileSystem` API for LLM use.

- **`src/lib/provider.ts`** — Returns either the real `anthropic("claude-haiku-4-5")` model or a `MockLanguageModel` when `ANTHROPIC_API_KEY` is absent. The mock streams canned component code through the same tool-call protocol.

- **`src/lib/prompts/generation.tsx`** — System prompt for the LLM. Key conventions it enforces: always create `/App.jsx` as entry point, use Tailwind for styling, use `@/` alias for all local imports.

- **`src/lib/auth.ts`** — Custom JWT auth via `jose`. Sessions stored in an `httpOnly` cookie (`auth-token`). No external auth provider.

- **`src/lib/prisma.ts`** — Singleton Prisma client. The generated client is output to `src/generated/prisma` (non-default path set in `prisma/schema.prisma`).

### Database Schema

The full schema is defined in [prisma/schema.prisma](prisma/schema.prisma) — reference it whenever you need to understand the structure of data stored in the database.

Two models in SQLite:
- **`User`** — email/password (bcrypt) accounts.
- **`Project`** — belongs to an optional `User`. `messages` (JSON string) stores the chat history; `data` (JSON string) stores the serialized VFS. Anonymous users' work is tracked client-side via `src/lib/anon-work-tracker.ts`.

### UI Layout

`src/app/main-content.tsx` renders a two-panel resizable layout:
- **Left**: `ChatInterface` (message list + input)
- **Right**: tabs switching between `PreviewFrame` (live iframe) and a code view (`FileTree` + `CodeEditor` via Monaco)

Both panels are wrapped in `FileSystemProvider` > `ChatProvider`.

### Path Alias

`@/` maps to `src/` (configured in `tsconfig.json`). The VFS also uses `@/` as a module alias when generating import maps for preview (maps to the virtual root `/`).
