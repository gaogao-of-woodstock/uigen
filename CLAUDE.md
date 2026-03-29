# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, and Claude AI generates them with real-time visual feedback. The app works with or without an Anthropic API key (falls back to mock generation).

## Commands

```bash
npm run setup          # Initial setup: install deps + generate Prisma client + run migrations
npm run dev            # Start dev server with Turbopack (uses mock provider if no API key)
npm run build          # Production build
npm run lint           # Run ESLint
npm run test           # Run Vitest unit tests
npm run db:reset       # Reset database (destructive)
```

Environment: Set `ANTHROPIC_API_KEY` in `.env` to use real AI generation.

## Architecture

### Three-Panel Layout
The UI (`src/app/main-content.tsx`) is a resizable three-panel layout:
- **Left (35%):** Chat interface for conversation-driven generation
- **Right top:** Live preview iframe
- **Right bottom:** File tree + Monaco code editor

### AI Generation Pipeline
1. User sends message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. API calls Claude via Vercel AI SDK's `streamText` with two tools:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — create/modify file content
   - `file_manager` (`src/lib/tools/file-manager.ts`) — delete files
3. Tool calls update the virtual file system (in-memory, no disk I/O)
4. File changes stream back to the client and trigger live preview refresh

### Virtual File System
- `src/lib/file-system.ts` — Map-based in-memory file store with path normalization
- `src/lib/contexts/file-system-context.tsx` — React context wrapping the file system, exposes `createFile`, `updateFile`, `deleteFile`
- File state is serialized as JSON for database persistence in `Project.data`

### Preview Execution
`src/lib/transform/jsx-transformer.ts` converts virtual files to runnable HTML:
- Uses Babel standalone to compile JSX
- Builds an import map for React/ReactDOM dependencies
- Injects Tailwind CSS via CDN
- Renders in a sandboxed iframe (`src/components/preview/PreviewFrame.tsx`)
- Auto-detects entry points: `App.jsx`, `index.tsx`, etc.

### State Management
- `src/lib/contexts/chat-context.tsx` — chat messages, streaming state, integration with file system updates
- `src/lib/contexts/file-system-context.tsx` — virtual file system state
- No external state library; React context + hooks throughout

### Authentication
- JWT sessions via `jose` (HS256, 7-day expiry, httpOnly cookies)
- `src/lib/auth.ts` — session creation/verification
- `src/middleware.ts` — protects `/api/projects` and `/api/filesystem` routes
- Passwords hashed with bcrypt
- Server actions in `src/actions/` handle auth and project CRUD

### Database
SQLite via Prisma. Schema at `prisma/schema.prisma`:
- `User` — id, email, password, timestamps
- `Project` — id, name, userId (nullable), `messages` (JSON string), `data` (serialized file system JSON), timestamps

### AI Provider
`src/lib/provider.ts` — Uses `claude-haiku-4-5` by default. Falls back to `MockLanguageModel` when `ANTHROPIC_API_KEY` is absent. Mock generates simple sample components (Counter, Form, Card).

### Path Alias
`@/` maps to `./src/` (configured in `tsconfig.json`).

## Code Style

Only add comments for genuinely complicated code.

## Key Technology Choices
- **Next.js 15 App Router** with Server Actions for auth/data
- **Vercel AI SDK** (`streamText` + tool use) for streaming generation
- **Tailwind CSS v4** + **Radix UI** primitives + **shadcn/ui** (new-york style)
- **Monaco Editor** for in-browser code editing
- **Vitest** + **React Testing Library** + **jsdom** for testing
