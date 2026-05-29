# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server at http://localhost:3000 (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run all tests (Vitest + jsdom)
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx  # Run a single test file
npx prisma migrate dev   # Apply new migrations after schema changes
npx prisma generate      # Regenerate client after schema changes (output: src/generated/prisma/)
npm run db:reset     # Wipe and re-seed the database
```

**Do not run `npm audit fix`** — dependencies are pinned to specific versions that work together.

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it the app falls back to a `MockLanguageModel` that returns canned components (counter, form, or card). The mock is detected by `src/lib/provider.ts:getLanguageModel()`. The live model is `claude-haiku-4-5`.

`JWT_SECRET` defaults to `"development-secret-key"` when unset.

## Architecture

### Request flow

```
Browser chat → POST /api/chat → streamText (Vercel AI SDK)
                                  ├── str_replace_editor tool → VirtualFileSystem mutations
                                  └── file_manager tool      → VirtualFileSystem mutations
                                  ↓ onFinish
                              prisma.project.update (if authenticated)
```

The chat route (`src/app/api/chat/route.ts`) deserializes the VFS sent from the client, runs the model with up to 40 tool-call steps, and on finish persists the updated VFS + messages to the DB.

### Virtual file system

`src/lib/file-system.ts` — an in-memory tree (`Map<string, FileNode>`). No files are ever written to disk. The client serializes the VFS to JSON and sends it with every chat request; the server deserializes it, mutates it via tool calls, then the client applies the same mutations via `handleToolCall` in `FileSystemContext`.

### Context layer

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — owns the VFS instance, exposes CRUD helpers and `handleToolCall`, tracks `refreshTrigger` (an incrementing int) to re-render consumers.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, wires `onToolCall` to `handleToolCall`, and passes the serialized VFS in the request body. Tracks anonymous work in `sessionStorage` via `src/lib/anon-work-tracker.ts`.

### Preview pipeline

`src/lib/transform/jsx-transformer.ts`:
1. Babel transforms each `.jsx/.tsx/.ts/.js` file from the VFS to plain JS (removes TypeScript, converts JSX to `React.createElement`).
2. Transformed files become blob URLs.
3. An ES module import map is built mapping file paths and `@/` aliases to those blob URLs; unknown third-party packages map to `https://esm.sh/<pkg>`.
4. `createPreviewHTML` produces a full HTML document with the import map, Tailwind CDN, and a `<script type="module">` that dynamically imports the entry point (prefers `/App.jsx`).
5. `PreviewFrame` sets this as `iframe.srcdoc` on every `refreshTrigger` change.

### Routing and auth

- `/` — anonymous or authenticated. Authenticated users are immediately redirected to their latest project via server-side redirect.
- `/[projectId]` — requires auth (server-side check); loads project messages and VFS data from DB and passes them to `MainContent`.
- Auth is JWT in an `httpOnly` cookie, implemented in `src/lib/auth.ts` using `jose`. Middleware (`src/middleware.ts`) guards only `/api/projects` and `/api/filesystem`.

### Data model

SQLite via Prisma (schema: `prisma/schema.prisma`). Generated client lives at `src/generated/prisma/` (not `node_modules`). Key models: `User` (email + bcrypt password) and `Project` (stores `messages` and `data` as JSON strings).

### AI tools

- `str_replace_editor` — create/str_replace/insert/view commands on VFS files (mirrors Anthropic's standard editor tool).
- `file_manager` — rename and delete commands.

Both tools are built in `src/lib/tools/` and operate on the server-side VFS instance; the client mirrors the same mutations via `handleToolCall`.
