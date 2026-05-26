# UIGen Agent Guidelines

This document helps AI coding agents quickly understand UIGen's architecture, conventions, and best practices for productive development.

## Project Overview

**UIGen** is an AI-powered React component generator. Users describe components in chat, Claude generates code, and a virtual file system + live preview let users iterate in real-time. Authenticated users can persist projects to a database.

- **Tech Stack**: Next.js 15, React 19, TypeScript, Tailwind CSS, Prisma/SQLite, Anthropic Claude API
- **Run**: `npm run dev` → http://localhost:3000
- **Setup**: `npm run setup` (installs deps, generates Prisma client, runs migrations)

For detailed architecture, see [CODEBASE_ANALYSIS.md](CODEBASE_ANALYSIS.md).

---

## Architecture Quick Reference

### Data Flow Diagram
```
User Chat Input
    ↓
ChatInterface (client component)
    ↓
useChat() hook (Vercel AI SDK) → /api/chat (streaming)
    ↓
Claude with tools: str_replace, file_manager, (more contextually)
    ↓
FileSystemContext (in-memory virtual FS)
    ↓
PreviewFrame (runs JSX in isolated iframe via @babel/standalone)
    ↓
MonacoEditor (code viewing/editing)
    ↓
Prisma (save to SQLite if authenticated)
```

### Key Layers
1. **Frontend**: Chat UI, code editor, file tree, live preview (React components)
2. **API**: `/api/chat` endpoint streams Claude responses + tool calls
3. **Virtual File System**: In-memory tree, persisted as JSON in Prisma
4. **Database**: SQLite with User (auth) and Project (state) models

---

## File Structure & Conventions

### Naming Patterns
- **Components**: PascalCase (`ChatInterface.tsx`, `MessageList.tsx`)
- **Utils/Hooks**: camelCase, dash-separated in filenames (`use-auth.ts`, `file-system.ts`)
- **API Routes**: kebab-case (`/api/chat/route.ts`)
- **Tests**: Component name + `.test.tsx` or `.test.ts` (colocated in `__tests__/`)

### Directory Organization
```
src/
  app/                      # Next.js App Router
    api/chat/route.ts       # Claude integration endpoint
    [projectId]/page.tsx    # Project detail page
  actions/                  # Server actions (auth, CRUD)
  components/
    chat/                   # Chat-related components
    editor/                 # Code editor components
    preview/                # Preview rendering
    ui/                     # Radix-based primitives
    auth/                   # Auth dialogs
  lib/
    contexts/               # React contexts (FileSystemContext, ChatContext)
    tools/                  # Claude tool implementations
    transform/              # JSX transformer for preview
    prompts/                # System prompts for Claude
    (utilities)
  middleware.ts             # Session verification
```

### @/* Path Alias
Use `@/` to import from `src/` root:
```typescript
import { prisma } from '@/lib/prisma';
import ChatInterface from '@/components/chat/ChatInterface';
```

---

## State Management

### Two Core Contexts

**1. FileSystemContext** (`src/lib/contexts/file-system-context.tsx`)
- Manages virtual file tree state
- Tracks file operations (create, update, delete)
- Updates when Claude calls tools
- Subscribe with `useFileSystem()` hook

**2. ChatContext** (`src/lib/contexts/chat-context.tsx`)
- Wraps Vercel AI SDK's `useChat()` hook
- Manages messages, loading state, error handling
- Integrates with FileSystemContext for tool calls
- Subscribe with `useChat()` hook (from context, not Vercel directly)

**When to use each:**
- Need to read/write files or track file operations? → `useFileSystem()`
- Need to send messages or track chat state? → `useChat()` (from ChatContext)

---

## Development Tasks

### Adding a New Feature (Example: "Export as ZIP")
1. **Identify the layer**:
   - Frontend-only (UI)? → Add component
   - Server-side logic? → Add server action or API route
   - Requires Claude tool? → Add to `tools/` and update `promptContext` in `/api/chat`

2. **Follow conventions**:
   - Create files in appropriate subdirectory
   - Use PascalCase for components, camelCase for utilities
   - Add tests colocated in `__tests__/`

3. **Update types** (if needed):
   - Extend TypeScript interfaces in `lib/contexts/`
   - Update Prisma schema if persistence needed

4. **Test locally**:
   - Run `npm run dev` for HMR
   - Run `npm run test` for unit tests
   - Manual testing in browser

5. **Database changes**:
   - Edit `prisma/schema.prisma`
   - Run `npx prisma migrate dev --name <migration_name>`
   - Commit the migration file

### Modifying Claude's Behavior
1. Update system prompt in `src/lib/prompts/generation.tsx`
2. Add/modify tools in `src/lib/tools/`
3. Update tool schema in `/api/chat/route.ts`
4. Test with `npm run dev` (changes auto-reload via HMR)

### Debugging

**Chat Context Issues:**
- Check `src/lib/contexts/chat-context.tsx` for message handling
- Verify Vercel AI SDK version in `package.json`
- Check browser DevTools → Network tab for `/api/chat` streaming

**File System Issues:**
- Inspect `useFileSystem()` state in React DevTools
- Verify file operations in tool implementations (`src/lib/tools/`)
- Check JSON serialization/deserialization logic

**Database:**
- View schema: `npx prisma studio` (opens browser UI)
- Reset database: `npm run db:reset` (⚠️ deletes all data)
- Check migrations: `ls prisma/migrations/`

**Prisma Client:**
- Must regenerate after schema changes: `npx prisma generate`
- Check generated files exist: `src/generated/prisma/` directory
- Clear Node cache if stale: `rm -r node_modules/.prisma`

---

## Testing

### Test Structure
- **Frameworks**: Vitest (runner) + React Testing Library (UI) + jsdom (DOM)
- **Location**: Colocated in `__tests__/` subdirectories
- **Run**: `npm run test`

### Test Patterns to Follow

**Component Testing:**
```typescript
import { render, screen } from '@testing-library/react';
import MessageList from '../MessageList';

test('renders messages', () => {
  const messages = [{ id: '1', role: 'user', content: 'Hello' }];
  render(<MessageList messages={messages} />);
  expect(screen.getByText('Hello')).toBeDefined();
});
```

**Context Testing:**
```typescript
import { render, screen } from '@testing-library/react';
import { ChatProvider } from '@/lib/contexts/chat-context';

test('ChatProvider initializes', () => {
  render(
    <ChatProvider initialMessages={[]}>
      <div>Test</div>
    </ChatProvider>
  );
  expect(screen.getByText('Test')).toBeDefined();
});
```

---

## Common Pitfalls & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "Can't resolve '@/generated/prisma'" | Prisma client not generated | Run `npx prisma generate` |
| Stale Prisma types | Schema changed but client not regenerated | Restart dev server or run `npx prisma generate` |
| Chat messages not persisting | User not authenticated or Project ID invalid | Check middleware → auth context → Prisma save logic |
| File tree not updating | FileSystemContext not subscribed or tool not integrated | Verify tool calls trigger context update |
| Preview broken after code change | JSX transformation error in iframe | Check browser console for @babel/standalone errors |
| Tests fail with "document is not defined" | jsdom not configured | Verify `vitest.config.mts` has `environment: 'jsdom'` |

---

## Key Files Reference

| File | Purpose | When to Edit |
|------|---------|--------------|
| `src/lib/prompts/generation.tsx` | Claude system prompt | Modify AI behavior, add instructions |
| `src/lib/tools/str-replace.ts` | Tool for code modification | Adjust how files are edited |
| `src/lib/tools/file-manager.ts` | Tool for file operations | Add file system capabilities |
| `src/lib/contexts/file-system-context.tsx` | File state management | Update file tree structure or operations |
| `src/lib/contexts/chat-context.tsx` | Chat state + Vercel AI SDK integration | Modify message handling, tool calls |
| `src/app/api/chat/route.ts` | Claude API endpoint | Add tools, modify streaming, update model |
| `prisma/schema.prisma` | Database schema | Add/modify entities or relationships |
| `src/lib/transform/jsx-transformer.ts` | Preview rendering | Adjust sandbox setup or imports |
| `src/middleware.ts` | Session/auth verification | Modify protected routes |

---

## Environment & Setup

### Environment Variables
- **ANTHROPIC_API_KEY**: Claude API key (optional; uses mock provider if not set)
  - Get from: https://console.anthropic.com/settings/keys
  - Can be placeholder for development

### First Time Setup
```bash
npm run setup        # Installs deps, generates Prisma, runs migrations
npm run dev          # Start dev server
npm run test         # Run all tests
```

### Database Reset
```bash
npm run db:reset     # ⚠️ Deletes all data, reapplies migrations
```

---

## Performance & Best Practices

- **Use `@/*` path aliases** instead of relative paths for consistency
- **Colocate tests** in `__tests__/` near components
- **Avoid circular imports** by organizing contexts and utilities clearly
- **Don't run `npm audit fix`** — dependencies are pinned for compatibility
- **Turbopack watches changes** — no manual restart needed during development
- **JSON serialization for Prisma**: Messages and file system state stored as JSON strings (see `CODEBASE_ANALYSIS.md` for details)

---

## Related Resources

- [CODEBASE_ANALYSIS.md](CODEBASE_ANALYSIS.md) — Detailed architecture, pain points, debug tips
- [README.md](README.md) — User-facing project overview
- [Next.js 15 Docs](https://nextjs.org/docs)
- [Vercel AI SDK](https://sdk.vercel.ai/)
- [Prisma Docs](https://www.prisma.io/docs/)
- [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/)
