# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates JSX code that renders instantly in a sandboxed iframe. Supports both authenticated users (persistent projects) and anonymous sessions.

## Commands

```bash
npm run dev          # Start dev server (Next.js + Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (all tests)
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Force reset the SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/transform/__tests__/jsx-transformer.test.ts
```

## Architecture

### Multi-Panel Layout

`src/app/main-content.tsx` renders a resizable split view:
- **Left (35%):** Chat interface (`src/components/chat/`)
- **Right (65%):** Togglable Preview iframe or Monaco code editor

### AI Code Generation Pipeline

1. User sends message → `ChatContext` → AI SDK `useChat` hook → `POST /api/chat`
2. `/api/chat/route.ts` calls Claude (Haiku 4.5) or `MockLanguageModel` (if no `ANTHROPIC_API_KEY`) with two tools:
   - `str_replace_editor` — create/view/replace/insert in files
   - `file_manager` — delete/rename files
3. Tool calls stream back to the browser; `handleToolCall()` in `FileSystemContext` mutates the virtual file system
4. Changing `refreshTrigger` causes `PreviewFrame` to re-transpile and re-render

### Virtual File System

`src/lib/file-system.ts` — in-memory tree structure. All file operations (create, update, delete, rename) go through this class. It is serialized to JSON and stored in the `Project.data` DB column for authenticated users.

### JSX Transformation & Preview

`src/lib/transform/jsx-transformer.ts` uses `@babel/standalone` in the browser to transpile JSX → JS and resolve imports. External packages (React 19, React-DOM 19, etc.) are mapped to `esm.sh` CDN via an import map injected into the iframe's `srcdoc`.

### State Management

Two React contexts wire everything together:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the `VirtualFileSystem` instance, exposes file CRUD, triggers preview refresh
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps AI SDK `useChat`, manages message history, invokes file system tool handlers

### Authentication & Persistence

- Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD
- JWT tokens in httpOnly cookies (7-day expiry) via `jose`; passwords hashed with `bcrypt`
- SQLite + Prisma; schema at `prisma/schema.prisma`

**Database models:**

| Model | Key fields |
|-------|-----------|
| `User` | `id` (cuid), `email` (unique), `password` (bcrypt), timestamps |
| `Project` | `id` (cuid), `name`, `userId` (nullable — null = anonymous), `messages` (JSON string), `data` (JSON string — serialized VirtualFileSystem), timestamps |

`Project.userId` is optional — anonymous users can have projects with no associated user. Deleting a `User` cascades to their projects.

### Mock Provider

When `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` returns a `MockLanguageModel` that simulates Claude responses with static Counter/ContactForm/Card component templates — useful for UI development without an API key.

## Key Conventions

- Path alias `@/*` resolves to `src/*` (configured in `tsconfig.json` and used throughout)
- Component entry point for the preview must be `App.jsx`; the system prompt enforces this
- All generated components use Tailwind CSS v4 for styling
- shadcn/ui components live in `src/components/ui/` (New York style, RSC-compatible)
- Tests use Vitest + React Testing Library with jsdom environment
