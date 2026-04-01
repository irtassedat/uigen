# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface, and Claude generates JSX code that renders in a sandboxed iframe preview. Components live in a virtual filesystem (nothing written to disk). Registered users get project persistence via SQLite.

## Commands

- `npm run setup` — install deps, generate Prisma client, run migrations (first-time setup)
- `npm run dev` — start dev server with Turbopack (http://localhost:3000)
- `npm run build` — production build
- `npm run lint` — ESLint
- `npm test` — run all tests with Vitest (jsdom environment)
- `npm test -- src/components/chat/__tests__/MessageInput.test.tsx` — run a single test file
- `npx prisma migrate dev` — apply schema changes
- `npm run db:reset` — reset the database

All `npm run dev/build/start` commands use `NODE_OPTIONS='--require ./node-compat.cjs'` for Node compatibility shims.

## Architecture

### AI Chat → Tool Calls → Virtual FS → Live Preview

The core loop:

1. **Chat** (`src/lib/contexts/chat-context.tsx`) sends messages to `/api/chat` via Vercel AI SDK's `useChat`
2. **API route** (`src/app/api/chat/route.ts`) streams responses from Claude using `streamText` with two tools:
   - `str_replace_editor` — create/view/replace/insert in files (modeled after Claude's text editor tool)
   - `file_manager` — rename/delete files
3. **Tool calls update the VirtualFileSystem** on both server (for the AI conversation) and client (for UI reactivity via `file-system-context.tsx`)
4. **PreviewFrame** (`src/components/preview/PreviewFrame.tsx`) transforms JSX files with Babel standalone, creates blob URLs, builds an ES module import map, and renders in a sandboxed iframe with Tailwind CDN

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` class is the central data structure. It's an in-memory tree of `FileNode` objects. Used on both server (chat route reconstructs from serialized data) and client (context provider). Files are serialized as JSON for persistence.

### Provider / Mock Mode

`src/lib/provider.ts` — if `ANTHROPIC_API_KEY` is set, uses Claude (claude-haiku-4-5). If not, a `MockLanguageModel` returns static component code so the app runs without an API key.

### Auth

JWT-based sessions via `jose`. Cookie-based (`auth-token`). Middleware protects `/api/projects` and `/api/filesystem`. Anonymous users can use the app but can't persist projects.

### Key Contexts (Client State)

- `FileSystemProvider` — owns the VirtualFileSystem instance, handles tool call side-effects on the client
- `ChatProvider` — wraps Vercel AI SDK's `useChat`, wires tool calls to the file system context

### Preview Pipeline

`src/lib/transform/jsx-transformer.ts` transforms JSX/TSX via Babel standalone → blob URLs → ES module import map. Third-party imports resolve to `esm.sh`. Missing local imports get placeholder modules. CSS files are injected as `<style>` tags.

### Database

Prisma + SQLite. Two models: `User` and `Project`. Projects store messages and virtual FS data as JSON strings. Schema at `prisma/schema.prisma`, Prisma client generated to `src/generated/prisma/`.

### UI Components

shadcn/ui (new-york style) in `src/components/ui/`. Icon library: lucide-react.

## Path Aliases

`@/*` maps to `./src/*` (tsconfig paths). This is also used in the virtual filesystem: generated components use `@/` imports that resolve at preview time.
