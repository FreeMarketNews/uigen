# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator. Users describe components in natural language, Claude AI generates/edits them via tool calls, and a live preview renders them in a sandboxed iframe. All generated code lives in an in-memory virtual file system (no disk writes) that serializes to a SQLite database for authenticated users.

## Commands

```bash
npm run setup          # Install deps + generate Prisma client + run migrations
npm run dev            # Start Next.js dev server with Turbopack (localhost:3000)
npm run lint           # ESLint
npm test               # Run all Vitest tests
npm test -- --watch    # Watch mode
npm test -- src/lib/__tests__/file-system.test.ts   # Run single test file
npm run db:reset       # Delete and recreate SQLite database
npx prisma studio      # Visual database browser
npx prisma migrate dev # Create new migration
npx prisma generate    # Regenerate Prisma types
```

## Environment

- `ANTHROPIC_API_KEY` — optional; when absent, a mock provider generates demo components
- `JWT_SECRET` — optional; defaults to a dev key

## Architecture

### Core Data Flow

1. User submits prompt → POST to `/api/chat` with messages + serialized virtual FS
2. Claude Haiku 4.5 responds via `streamText()` with tool calls (`str_replace_editor`, `file_manager`)
3. Tool calls manipulate the `VirtualFileSystem` instance in `FileSystemContext`
4. `PreviewFrame` detects changes, runs Babel transform via `jsx-transformer.ts`, renders in sandboxed iframe
5. On chat completion, project is saved to Prisma (if authenticated + has projectId)

### Virtual File System (`src/lib/file-system.ts`)

Central abstraction — a `VirtualFileSystem` class that holds all generated files in memory. Supports create, read, update, delete, rename, and text-editing operations (view, str_replace, insert). Serializes to JSON for database persistence via `serialize()`/`deserialize()`.

### State Management — Two React Contexts

- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): Owns the VirtualFileSystem instance, selected file state, and processes AI tool calls via `handleToolCall()`
- **ChatContext** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat()`, sends file system state with each message, tracks anonymous work in sessionStorage

### AI Tools (what Claude calls during generation)

- **str_replace_editor** (`src/lib/tools/str-replace.ts`): create files, view files, string replacement, line insertion
- **file_manager** (`src/lib/tools/file-manager.ts`): rename and delete files/directories

### Preview System (`src/lib/transform/jsx-transformer.ts`)

Babel transforms JSX/TSX to JS in the browser, creates blob URLs with import maps, loads third-party packages via esm.sh CDN, and injects Tailwind CSS via CDN. Entry point detection: App.jsx → App.tsx → index.jsx → index.tsx.

### Authentication

JWT (HS256, 7-day expiry) stored in HttpOnly cookies. Server actions in `src/actions/index.ts` handle signUp/signIn/signOut. Middleware protects `/api/projects` and `/api/filesystem` routes.

### Database (Prisma + SQLite)

Two models: **User** (email, hashed password) and **Project** (name, JSON-stringified messages and file system data). Projects have optional userId — anonymous projects exist but aren't persisted to DB.

## Key Directories

- `src/app/` — Next.js App Router pages and API routes
- `src/components/` — UI components organized by feature (auth, chat, editor, preview, ui)
- `src/lib/` — Core logic: virtual FS, contexts, AI tools, JSX transformer, auth, prompts
- `src/actions/` — Server actions for auth and project CRUD
- `src/components/ui/` — shadcn/ui components (New York style, Radix-based)
- `prisma/` — Schema and SQLite database

## Tech Stack

- **Next.js 15** (App Router, Turbopack) + **React 19** + **TypeScript 5**
- **Vercel AI SDK** with `@ai-sdk/anthropic` — Claude Haiku 4.5 default model
- **Tailwind CSS v4** + **shadcn/ui** + **Lucide icons**
- **Monaco Editor** for code editing, **react-resizable-panels** for layout
- **Prisma 6** with SQLite, **bcrypt** for passwords, **jose** for JWT
- **Vitest** + **Testing Library** + **JSDOM** for tests
- **@babel/standalone** for client-side JSX transformation

## Testing

Tests live in `__tests__/` directories alongside their source. Uses Vitest with JSDOM environment and React Testing Library. Mock provider (`src/lib/provider.ts`) enables testing without an API key.

## Path Alias

`@/*` maps to `./src/*` (configured in tsconfig.json and vitest.config.mts).
