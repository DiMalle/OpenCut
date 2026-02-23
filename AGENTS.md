# AGENTS.md

## Overview

Privacy-first video editor, with a focus on simplicity and ease of use.

## Lib vs Utils

- `lib/` - domain logic (specific to this app)
- `utils/` - small helper utils (generic, could be copy-pasted into any other app)

## Core Editor System

The editor uses a **singleton EditorCore** that manages all editor state through specialized managers.

### Architecture

```
EditorCore (singleton)
├── playback: PlaybackManager
├── timeline: TimelineManager
├── scene: SceneManager
├── project: ProjectManager
├── media: MediaManager
└── renderer: RendererManager
```

### When to Use What

#### In React Components

**Always use the `useEditor()` hook:**

```typescript
import { useEditor } from '@/hooks/use-editor';

function MyComponent() {
  const editor = useEditor();
  const tracks = editor.timeline.getTracks();

  // Call methods
  editor.timeline.addTrack({ type: 'media' });

  // Display data (auto re-renders on changes)
  return <div>{tracks.length} tracks</div>;
}
```

The hook:

- Returns the singleton instance
- Subscribes to all manager changes
- Automatically re-renders when state changes

#### Outside React Components

**Use `EditorCore.getInstance()` directly:**

```typescript
// In utilities, event handlers, or non-React code
import { EditorCore } from "@/core";

const editor = EditorCore.getInstance();
await editor.export({ format: "mp4", quality: "high" });
```

## Actions System

Actions are the trigger layer for user-initiated operations. The single source of truth is `@/lib/actions/definitions.ts`.

**To add a new action:**

1. Add it to `ACTIONS` in `@/lib/actions/definitions.ts`:

```typescript
export const ACTIONS = {
  "my-action": {
    description: "What the action does",
    category: "editing",
    defaultShortcuts: ["ctrl+m"],
  },
  // ...
};
```

2. Add handler in `@/hooks/use-editor-actions.ts`:

```typescript
useActionHandler(
  "my-action",
  () => {
    // implementation
  },
  undefined,
);
```

**In components, use `invokeAction()` for user-triggered operations:**

```typescript
import { invokeAction } from '@/lib/actions';

// Good - uses action system
const handleSplit = () => invokeAction("split-selected");

// Avoid - bypasses UX layer (toasts, validation feedback)
const handleSplit = () => editor.timeline.splitElements({ ... });
```

Direct `editor.xxx()` calls are for internal use (commands, tests, complex multi-step operations).

## Commands System

Commands handle undo/redo. They live in `@/lib/commands/` organized by domain (timeline, media, scene).

Each command extends `Command` from `@/lib/commands/base-command` and implements:

- `execute()` - saves current state, then does the mutation
- `undo()` - restores the saved state

Actions and commands work together: actions are "what triggered this", commands are "how to do it (and undo it)".

## Cloud-specific instructions

### Prerequisites

- **Bun** (v1.2.18) must be installed. The update script handles this via `~/.bun/bin/bun`.
- **No Docker required** for frontend-only development. Docker (PostgreSQL + Redis) is only needed for auth features, and no `docker-compose.yaml` exists in the repo yet.

### Environment variables

All env vars are validated by Zod at startup (`packages/env/src/web.ts`). Every field is required (no `.optional()`), so `.env.local` must exist under `apps/web/` with valid-looking values for all variables. For local development without external services, placeholder values work for: `MARBLE_WORKSPACE_KEY`, `FREESOUND_CLIENT_ID`, `FREESOUND_API_KEY`, `CLOUDFLARE_ACCOUNT_ID`, `R2_*`, and `MODAL_TRANSCRIPTION_URL` (must be a valid URL like `http://localhost:9999/transcription`).

### Running services

| Service | Command | Notes |
|---------|---------|-------|
| Dev server | `bun run dev` (from `apps/web`) | Runs Next.js 16 with Turbopack on port 3000 |
| Lint | `bun run lint` (from `apps/web`) | Uses Biome. Pre-existing CSS/Tailwind directive warnings are expected. |
| Tests | `bun test` (from repo root) | 60 tests across 7 files (storage migrations + sticker-id) |
| Format | `bun run format` (from `apps/web`) | Biome formatter |

### Gotchas

- `@biomejs/biome` is a root devDependency needed for the lint/format scripts in `apps/web`. The lint script calls `biome` directly.
- The env schema at `packages/env/src/web.ts` uses `z.url()` for several fields (`NEXT_PUBLIC_SITE_URL`, `NEXT_PUBLIC_MARBLE_API_URL`, `UPSTASH_REDIS_REST_URL`, `MODAL_TRANSCRIPTION_URL`), meaning they must be full valid URLs (with protocol). Plain strings will fail validation.
- The repo's README references `docker-compose up -d` but no `docker-compose.yaml` exists. Auth features (login, signup) won't work without PostgreSQL and Redis.
