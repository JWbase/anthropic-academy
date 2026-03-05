# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Use comments sparingly. Only comment complex code.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and Claude generates React code that renders in a sandboxed iframe preview. Components live in a virtual file system (no disk writes). Authenticated users get project persistence via SQLite/Prisma.

## Commands

- `npm run setup` — Install deps, generate Prisma client, run migrations (first-time setup)
- `npm run dev` — Start dev server with Turbopack (http://localhost:3000)
- `npm run build` — Production build
- `npm run lint` — ESLint
- `npm run test` — Run all tests (vitest, jsdom environment)
- `npx vitest run src/path/to/test.test.ts` — Run a single test file
- `npx prisma migrate dev` — Run database migrations
- `npx prisma generate` — Regenerate Prisma client after schema changes
- `npm run db:reset` — Reset database (destructive)

## Architecture

### Data Flow

1. User sends message via chat → `ChatProvider` (uses Vercel AI SDK's `useChat`) → POST `/api/chat/route.ts`
2. Server reconstructs `VirtualFileSystem` from serialized file data sent with each request
3. Server streams Claude responses using `streamText` with two AI tools: `str_replace_editor` and `file_manager`
4. Tool calls arrive on client via `onToolCall` → `FileSystemContext.handleToolCall` applies file operations to the client-side VFS
5. `PreviewFrame` watches `refreshTrigger` from FileSystemContext, transforms files with Babel (`jsx-transformer.ts`), builds import maps with blob URLs, and renders in a sandboxed iframe

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`): In-memory tree-based filesystem. Serialized as `Record<string, FileNode>` for client-server transport. Provides text editor operations (view, create, str_replace, insert) used by AI tools.
- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Client-side Babel transform that converts JSX/TSX to ES modules, creates blob URLs, builds import maps. Third-party packages resolve via esm.sh. Local imports use `@/` alias mapping to root.
- **Mock Provider** (`src/lib/provider.ts`): When `ANTHROPIC_API_KEY` is unset, a `MockLanguageModel` returns static component code (counter/form/card). Uses claude-haiku-4-5 when key is present.

### AI Tools (Server-side)

The AI has two tools to manipulate the virtual filesystem:
- `str_replace_editor`: create files, view files, string replace, insert at line
- `file_manager`: rename/move and delete files

### Context Providers (Client)

`FileSystemProvider` wraps `ChatProvider`. Chat sends serialized file state with every message. Tool calls flow: ChatProvider.onToolCall → FileSystemContext.handleToolCall → VFS mutation → refreshTrigger increment → PreviewFrame re-renders.

### Routing

- `/` — Anonymous users see MainContent directly; authenticated users redirect to most recent project (or create one)
- `/[projectId]` — Authenticated project page, loads saved messages and file data
- `/api/chat` — Streaming chat endpoint (unprotected, but project saving requires auth)

### Auth

JWT sessions stored in httpOnly cookies (`jose` library). Server actions in `src/actions/index.ts` handle signUp/signIn/signOut. Middleware protects `/api/projects` and `/api/filesystem` routes.

### Database

SQLite via Prisma. Two models: `User` and `Project`. Projects store `messages` (JSON stringified array) and `data` (JSON stringified VFS state). Prisma client output is at `src/generated/prisma/`.

## Conventions

- Path alias: `@/*` maps to `./src/*`
- UI components: shadcn/ui (new-york style) in `src/components/ui/`
- Styling: Tailwind CSS v4 with CSS variables for theming
- Tests colocated in `__tests__/` directories next to source files
- The generated AI components use `@/` imports relative to the virtual filesystem root (not the Next.js project root)
- Dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (handled by npm scripts)
