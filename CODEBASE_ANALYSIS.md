# UIGen Codebase Analysis Report

## Executive Summary
UIGen is a sophisticated AI-powered React component generator built with Next.js 15, React 19, and the Vercel AI SDK. It uses Claude via Anthropic API to generate React components based on user descriptions, with a virtual file system, live preview, and optional persistence for authenticated users.

---

## 1. Architecture Overview

### High-Level Structure
UIGen follows a **client-server architecture** with clear separation:

```
┌─────────────────────────────────────────────────────┐
│  Frontend (Client Components)                        │
│  - Chat Interface (messaging with Claude)            │
│  - Code Editor (Monaco with syntax highlighting)     │
│  - File Tree (virtual FS navigation)                 │
│  - Live Preview (iframe with JSX transformer)        │
└─────────────────────────────────────────────────────┘
                        ↕ (Websocket/HTTP)
┌─────────────────────────────────────────────────────┐
│  Backend (Next.js 15 App Router)                     │
│  - /api/chat → Anthropic Claude integration          │
│  - Server Actions (auth, projects)                   │
│  - Virtual File System (in-memory)                   │
│  - Tool Execution (str_replace, file_manager)        │
└─────────────────────────────────────────────────────┘
                        ↕ (Prisma ORM)
┌─────────────────────────────────────────────────────┐
│  Database (SQLite)                                   │
│  - User (authentication & sessions)                  │
│  - Project (chat history & file system state)        │
└─────────────────────────────────────────────────────┘
```

### Key Components & Their Interactions

**Frontend Layers:**
- **Chat Context** → Manages conversation state, integrates Vercel AI SDK `useChat` hook
- **FileSystem Context** → Tracks virtual file tree, handles tool calls from Claude
- **Components**:
  - `ChatInterface` → Message display + input
  - `FileTree` → Hierarchical file navigation
  - `CodeEditor` → Monaco editor for code viewing/editing
  - `PreviewFrame` → Renders App.jsx in isolated iframe

**Backend Layers:**
- **Server Actions** (`/actions/`) → Async operations (auth, project CRUD)
- **API Routes** (`/api/chat/`) → Streaming endpoint for Claude conversation
- **Middleware** → Session verification for protected paths
- **Virtual File System** → In-memory representation of generated code

**Database:**
- `User` model → Email/password auth with bcrypt hashing
- `Project` model → Stores chat messages + file system state as JSON strings

---

## 2. Tech Stack & Key Dependencies

### Core Framework
- **Next.js 15.3.3** → Full-stack React framework with Turbopack
- **React 19.0.0** → Latest with automatic batching
- **TypeScript 5.x** → Strict type checking

### AI Integration
- **@ai-sdk/anthropic 1.2.12** → Anthropic provider
- **ai 4.3.16** → Vercel AI SDK (streamText, useChat, tools)
- **claude-haiku-4-5** → Default Claude model

### UI Components & Styling
- **@radix-ui/** → Accessible component library
  - dialog, tabs, popover, scroll-area, separator, label
- **Tailwind CSS 4** → Utility-first CSS framework
- **lucide-react 0.517.0** → Icon library
- **class-variance-authority** → Component variant patterns
- **cmdk** → Command menu/palette

### Code Editing & Preview
- **@monaco-editor/react 4.7.0** → VS Code editor
- **react-markdown 10.1.0** → Markdown rendering in chat
- **@babel/standalone 7.27.6** → JSX transformation (browser-based)

### Database & Authentication
- **@prisma/client 6.10.1** → Database ORM
- **sqlite** (implicit) → Lightweight database
- **bcrypt 6.0.0** → Password hashing
- **jose 6.0.11** → JWT token generation/verification

### Layout & Interactions
- **react-resizable-panels 3.0.3** → Draggable panel resizing
- **react-dom 19.0.0** → React DOM rendering

### Testing
- **vitest 3.2.4** → Fast unit test runner
- **@testing-library/react 16.3.0** → React testing utilities
- **@testing-library/dom 10.4.0** → DOM testing
- **jsdom 26.1.0** → DOM simulation for Node

### Development
- **Turbopack** → Next.js bundler (enables fast dev mode)
- **eslint** → Code linting
- **vite-tsconfig-paths** → Path alias resolution in tests

---

## 3. Build & Development Commands

### Available npm Scripts (from `package.json`)

| Command | Purpose | Notes |
|---------|---------|-------|
| `npm run dev` | Start dev server with Turbopack | Uses `next dev --turbopack` for fast HMR |
| `npm run dev:daemon` | Background dev server | Writes logs to `logs.txt`, useful for long sessions |
| `npm run build` | Production build | Optimized bundle for deployment |
| `npm run start` | Start production server | Requires `npm run build` first |
| `npm run lint` | ESLint code quality | Checks against Next.js config |
| `npm run test` | Run vitest tests | Uses jsdom environment for DOM tests |
| `npm run setup` | Initial project setup | Installs deps, generates Prisma, runs migrations |
| `npm run db:reset` | Hard reset database | ⚠️ Deletes all data, resets migrations |

### Typical Development Workflow
```bash
npm run setup          # First-time setup
npm run dev            # Start local dev server @ http://localhost:3000
# Edit code, auto-refresh via Turbopack HMR
npm run test           # Run unit tests (during development)
npm run build          # Before deployment
npm run start          # Test production build locally
```

### Key Configuration Files
- **next.config.ts** → Disables dev indicators, pins Turbopack workspace root
- **vitest.config.mts** → jsdom environment, React plugin, tsconfig path resolution
- **tsconfig.json** → ES2017 target, `@/*` path alias, strict mode enabled
- **postcss.config.mjs** → Tailwind CSS processing pipeline

---

## 4. Testing Setup

### Test Organization
```
src/components/
├── chat/__tests__/
│   ├── ChatInterface.test.tsx
│   ├── MarkdownRenderer.test.tsx
│   ├── MessageInput.test.tsx
│   └── MessageList.test.tsx
├── editor/__tests__/
│   └── file-tree.test.tsx
src/lib/
├── __tests__/
│   └── file-system.test.ts
├── contexts/__tests__/
│   ├── chat-context.test.tsx
│   └── file-system-context.test.tsx
├── transform/__tests__/
│   └── (JSX transformer tests)
```

### Testing Framework
- **Test Runner**: Vitest 3.2.4 (fast, Vite-native)
- **DOM Environment**: jsdom 26.1.0
- **UI Testing Library**: @testing-library/react 16.3.0
- **User Interaction**: @testing-library/user-event 14.6.1
- **Mocking**: vitest's built-in `vi` object

### Test Patterns Observed

**1. Component Testing**
```typescript
// Example: MessageList.test.tsx
test("MessageList shows empty state when no messages", () => {
  render(<MessageList messages={[]} />);
  expect(screen.getByText("Start a conversation...")).toBeDefined();
});

test("MessageList renders user messages", () => {
  const messages: Message[] = [{ id: "1", role: "user", content: "..." }];
  render(<MessageList messages={messages} />);
  expect(screen.getByText("Create a button component")).toBeDefined();
});
```

**2. Context Testing**
```typescript
// Example: chat-context.test.tsx
vi.mock("../file-system-context");
vi.mock("@ai-sdk/react");
vi.mock("@/lib/anon-work-tracker");

test("ChatProvider initializes with messages", () => {
  const initialMessages = [{ id: "1", role: "user", content: "Hello" }];
  render(<ChatProvider initialMessages={initialMessages}><TestComponent /></ChatProvider>);
  // Assertions...
});
```

**3. Key Testing Utilities**
- `render()` → Mount component in jsdom
- `screen.getByText()` → Query rendered text
- `waitFor()` → Wait for async updates
- `cleanup()` → Reset DOM after each test
- `vi.mock()` → Mock external modules
- `vi.fn()` → Create mock functions

### Running Tests
```bash
npm run test              # Run all tests
npm run test -- --ui     # Interactive UI mode (if installed)
npm run test -- src/     # Test specific directory
```

---

## 5. Database Schema

### Prisma Schema Overview (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../src/generated/prisma"  // Generated client location
}

datasource db {
  provider = "sqlite"
  url      = "file:./dev.db"  // Local SQLite file
}

model User {
  id        String    @id @default(cuid())      // UUID-like ID
  email     String    @unique                    // Email login key
  password  String                               // bcrypt hash
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  projects  Project[]                            // 1:N relationship
}

model Project {
  id        String    @id @default(cuid())
  name      String                               // User-friendly name (e.g., "Design #42069")
  userId    String?                              // Nullable for anonymous projects (not enforced)
  messages  String    @default("[]")             // JSON array of chat messages
  data      String    @default("{}")             // JSON virtual file system state
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  user      User?     @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

### Data Model Details

**User Table:**
- Stores email/password credentials
- Uses bcrypt for secure password hashing (cost factor 10)
- Sessions managed via JWT in httpOnly cookies
- Cascade delete → Removing user deletes their projects

**Project Table:**
- `messages` → JSON-serialized array of `Message` objects (Vercel AI SDK format)
  ```json
  [
    { "id": "1", "role": "user", "content": "Create a button" },
    { "id": "2", "role": "assistant", "content": "..." }
  ]
  ```
- `data` → Serialized virtual file system (FileNode tree)
  ```json
  {
    "files": {
      "/App.jsx": { "type": "file", "content": "export default..." },
      "/components": { "type": "directory", "children": {...} }
    }
  }
  ```

### Migration History
```
migrations/
├── migration_lock.toml          // Prisma lock file
├── 20250619172131_init/         // Initial schema
│   └── migration.sql            // User + Project tables
├── 20250619174023_optional_userid/
│   └── migration.sql            // Made userId optional for anonymous projects
└── 20250619174322_remove_filesystem_add_data_to_project/
    └── migration.sql            // Changed storage structure
```

---

## 6. State Management & Contexts

### Context Architecture

UIGen uses **React Context API** with Provider pattern for state management (no Redux/Zustand needed).

#### 1. **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`)
Manages the virtual file system in-memory state.

**Type Definition:**
```typescript
interface FileSystemContextType {
  fileSystem: VirtualFileSystem;
  selectedFile: string | null;
  setSelectedFile: (path: string | null) => void;
  createFile: (path: string, content?: string) => void;
  updateFile: (path: string, content: string) => void;
  deleteFile: (path: string) => void;
  renameFile: (oldPath: string, newPath: string) => boolean;
  getFileContent: (path: string) => string | null;
  getAllFiles: () => Map<string, string>;
  refreshTrigger: number;
  handleToolCall: (toolCall: ToolCall) => void;
  reset: () => void;
}
```

**Key Features:**
- Encapsulates `VirtualFileSystem` class
- Auto-selects `/App.jsx` if it exists (or first .jsx/.tsx file)
- Handles tool calls from Claude (str_replace_editor, file_manager)
- `refreshTrigger` → Forces re-renders when files change
- Initializes from `initialData` (from persisted projects)

**Usage:**
```typescript
const { fileSystem, selectedFile, updateFile, handleToolCall } = useFileSystem();
```

#### 2. **ChatContext** (`src/lib/contexts/chat-context.tsx`)
Bridges AI SDK with file system updates.

**Type Definition:**
```typescript
interface ChatContextType {
  messages: Message[];
  input: string;
  handleInputChange: (e: React.ChangeEvent<HTMLTextAreaElement>) => void;
  handleSubmit: (e: React.FormEvent<HTMLFormElement>) => void;
  status: string;  // "idle" | "submitted" | "streaming"
}
```

**Key Features:**
- Wraps Vercel AI SDK's `useChat` hook
- Passes file system to Claude API (`body: { files, projectId }`)
- `onToolCall` → Delegates to `fileSystem.handleToolCall()`
- Tracks anonymous work via `setHasAnonWork()` (sessionStorage)
- Integrates with `/api/chat` endpoint

**Usage:**
```typescript
const { messages, input, handleInputChange, handleSubmit, status } = useChat();
```

#### 3. **Provider Hierarchy**
```
RootLayout (root page)
└── MainContent
    ├── FileSystemProvider
    │   └── ChatProvider
    │       ├── ChatInterface (uses both contexts)
    │       ├── CodeEditor (uses FileSystemContext)
    │       ├── FileTree (uses FileSystemContext)
    │       └── PreviewFrame (uses FileSystemContext)
```

### State Flow Diagram
```
User Input (ChatInterface)
    ↓
handleSubmit() → POST /api/chat
    ↓
Claude receives messages + files
    ↓
Claude calls tools (str_replace_editor, file_manager)
    ↓
onToolCall() fires on client
    ↓
fileSystem.handleToolCall() updates VirtualFileSystem
    ↓
refreshTrigger increments → Re-renders (FileTree, CodeEditor, PreviewFrame)
    ↓
All children components re-read from FileSystemContext
```

### Anonymous Work Tracking
Located in `src/lib/anon-work-tracker.ts`:
```typescript
// Stores session-only data for unauthenticated users
setHasAnonWork(messages, fileSystem)  // Save to sessionStorage
getHasAnonWork()                       // Check if work exists
getAnonWorkData()                      // Retrieve saved state
clearAnonWork()                        // Clean up
```

---

## 7. Key Conventions

### File Naming Patterns

| Pattern | Usage | Examples |
|---------|-------|----------|
| `ComponentName.tsx` | React components (PascalCase) | `ChatInterface.tsx`, `FileTree.tsx` |
| `utility-name.ts` | Utilities & helpers (kebab-case) | `anon-work-tracker.ts`, `file-system.ts` |
| `action-name.ts` | Server actions (kebab-case) | `create-project.ts`, `get-projects.ts` |
| `ComponentName.test.tsx` | Unit tests (mirror filename) | `MessageList.test.tsx` |
| `__tests__/` | Test directory structure | Colocated with components |
| `/api/route-name/route.ts` | API endpoints (lowercase) | `/api/chat/route.ts` |
| `XContext.tsx` | Context providers | `ChatContext.tsx`, `FileSystemContext.tsx` |
| `use-hook-name.ts` | Custom hooks (kebab-case) | `use-auth.ts` |

### Component Structure Patterns

**Functional Client Component:**
```typescript
"use client";  // Client-side marker for Next.js

import { useState, useEffect } from "react";
import { useFileSystem } from "@/lib/contexts/file-system-context";

export function MyComponent() {
  const [state, setState] = useState(null);
  const { selectedFile } = useFileSystem();

  useEffect(() => {
    // Side effects
  }, [selectedFile]);

  return <div>...</div>;
}
```

**Server Component (Actions):**
```typescript
"use server";  // Server-only marker

import { getSession } from "@/lib/auth";
import { prisma } from "@/lib/prisma";

export async function myAction(input: Input): Promise<Output> {
  const session = await getSession();
  if (!session) throw new Error("Unauthorized");
  
  const result = await prisma.model.operation({...});
  return result;
}
```

**Provider Wrapper:**
```typescript
interface ProviderProps {
  children: React.ReactNode;
  initialData?: T;
}

export function MyProvider({ children, initialData }: ProviderProps) {
  const [state, setState] = useState(initialData);
  
  return (
    <MyContext.Provider value={{ state, setState }}>
      {children}
    </MyContext.Provider>
  );
}
```

### API Routes Organization

All API routes use Next.js App Router pattern:

```
src/app/api/
└── chat/
    └── route.ts  // POST /api/chat
                  // Streaming endpoint for Claude conversation
                  // Receives: messages[], files, projectId
                  // Returns: Server-sent events (SSE) stream
```

**Route Pattern:**
```typescript
export async function POST(req: Request) {
  const { messages, files, projectId } = await req.json();
  
  // Add system prompt with caching
  messages.unshift({
    role: "system",
    content: generationPrompt,
    providerOptions: {
      anthropic: { cacheControl: { type: "ephemeral" } },
    },
  });

  const result = streamText({
    model: getLanguageModel(),
    messages,
    maxTokens: 10_000,
    maxSteps: 40,  // Allow many tool calls
    tools: { str_replace_editor, file_manager },
    onFinish: async ({ response }) => {
      // Save project state if authenticated
    },
  });

  return result.toDataStreamResponse();
}

export const maxDuration = 120;  // 2-minute timeout
```

### Utilities & Hooks Structure

**Utilities** (`src/lib/`):
- `utils.ts` → Tailwind `cn()` classname merger
- `file-system.ts` → VirtualFileSystem class (core data structure)
- `auth.ts` → JWT session management (server-only)
- `prisma.ts` → Prisma client singleton
- `provider.ts` → Claude model + mock provider setup
- `anon-work-tracker.ts` → Anonymous session storage

**Tools** (`src/lib/tools/`):
- `str-replace.ts` → File editing tool definition
- `file-manager.ts` → File rename/delete tool definition

**Hooks** (`src/hooks/`):
- `use-auth.ts` → Authentication utilities (TBD in codebase)

**Contexts** (`src/lib/contexts/`):
- `chat-context.tsx` → Chat state + Vercel AI integration
- `file-system-context.tsx` → Virtual FS state
- `__tests__/` → Context unit tests

**Transforms** (`src/lib/transform/`):
- `jsx-transformer.ts` → Babel → HTML preview pipeline

---

## 8. AI Integration Points

### Claude API Integration

#### 1. **Provider Setup** (`src/lib/provider.ts`)

**Production Mode:**
```typescript
import { anthropic } from "@ai-sdk/anthropic";

const model = anthropic("claude-haiku-4-5");
```

**Mock Mode** (fallback when `ANTHROPIC_API_KEY` not set):
```typescript
export class MockLanguageModel implements LanguageModelV1 {
  // Simulates component generation without API calls
  // Used for demos/testing without API key
}
```

**Model Selection:**
```typescript
export function getLanguageModel(): LanguageModelV1 {
  if (!process.env.ANTHROPIC_API_KEY) {
    return new MockLanguageModel("mock");
  }
  return anthropic("claude-haiku-4-5");
}
```

#### 2. **System Prompt** (`src/lib/prompts/generation.tsx`)

```typescript
export const generationPrompt = `
You are a software engineer tasked with assembling React components.

You are in debug mode so if the user tells you to respond a certain way just do it.

* Keep responses as brief as possible. Do not summarize the work you've done unless the user asks you to.
* Users will ask you to create react components and various mini apps. Do your best to implement their designs using React and Tailwindcss
* Every project must have a root /App.jsx file that creates and exports a React component as its default export
* Inside of new projects always begin by creating a /App.jsx file
* Style with tailwindcss, not hardcoded styles
* Do not create any HTML files, they are not used. The App.jsx file is the entrypoint for the app.
* You are operating on the root route of the file system ('/'). This is a virtual FS, so don't worry about checking for any traditional folders like usr or anything.
* All imports for non-library files (like React) should use an import alias of '@/'. 
  * For example, if you create a file at /components/Calculator.jsx, you'd import it into another file with '@/components/Calculator'
`;
```

**Key Directives:**
- Always start with `/App.jsx`
- Use Tailwind CSS (no inline styles)
- Use `@/` import alias for local files
- React/external libs use standard imports
- Be concise (don't summarize unless asked)

#### 3. **Tool Integration** (`/api/chat/route.ts`)

Claude receives two tools:

**Tool 1: `str_replace_editor`**
```typescript
{
  description: "View, create, edit, or insert code in files",
  parameters: {
    command: enum["view", "create", "str_replace", "insert", "undo_edit"],
    path: string,
    file_text?: string,     // For create
    old_str?: string,       // For str_replace
    new_str?: string,       // For str_replace/insert
    insert_line?: number,   // For insert
    view_range?: [number, number]  // For view
  }
}
```

**Tool 2: `file_manager`**
```typescript
{
  description: "Rename or delete files/folders",
  parameters: {
    command: enum["rename", "delete"],
    path: string,
    new_path?: string  // Required for rename
  }
}
```

#### 4. **Request/Response Flow**

**Request to Claude:**
```typescript
const result = streamText({
  model,
  messages: [
    { role: "system", content: generationPrompt, providerOptions: { ... } },
    { role: "user", content: "Create a button component" }
  ],
  maxTokens: 10_000,
  maxSteps: 40,  // Allows many tool calls for complex components
  tools: { str_replace_editor, file_manager },
  onFinish: async ({ response }) => {
    // Save to database after completion
  }
});

return result.toDataStreamResponse();  // Server-sent events stream
```

**Client-Side Streaming:**
```typescript
// From @ai-sdk/react
const { messages, input, handleSubmit } = useChat({
  api: "/api/chat",
  body: { files: fileSystem.serialize(), projectId },
  onToolCall: ({ toolCall }) => {
    fileSystem.handleToolCall(toolCall);  // Update virtual FS
  }
});
```

#### 5. **Prompt Caching** (Performance Optimization)

```typescript
messages.unshift({
  role: "system",
  content: generationPrompt,
  providerOptions: {
    anthropic: { 
      cacheControl: { type: "ephemeral" }  // Cache system prompt
    },
  },
});
```

Uses Anthropic's prompt caching to reduce latency and cost for repeated system prompts.

#### 6. **Environment Configuration**

```bash
# .env
ANTHROPIC_API_KEY=sk-ant-...  # From console.anthropic.com

# If not set: uses MockLanguageModel (for development/demos)
```

### Vercel AI SDK Integration

**Package:** `ai@4.3.16`

**Key Functions:**
- `streamText()` → Stream Claude responses with tool calling
- `useChat()` → React hook for chat UI integration
- `appendResponseMessages()` → Merge assistant responses with chat history
- Tool definitions via `tool()` and `z` (Zod validation)

---

## 9. Common Pain Points & Potential Issues

### 1. **Data Serialization/Deserialization**
**Issue:** Messages and file system stored as JSON strings in Prisma
```typescript
// Causes type safety issues
messages: JSON.stringify(input.messages),  // String in DB
data: JSON.stringify(input.data),          // String in DB

// Must deserialize on load
const messages = JSON.parse(project.messages);
const data = JSON.parse(project.data);
```
**Impact:** Bugs if JSON parsing fails silently; no compile-time safety
**Workaround:** Add Zod validation schema for JSON payloads

### 2. **Virtual File System State Management**
**Issue:** In-memory file system not persisted to disk
- Loss on server restart
- No backup mechanism
- Sync issues between client/server
**Impact:** Data loss in development if not saved to project
**Mitigation:** Always call `onFinish` to save project state

### 3. **Anonymous User Session Fragility**
**Issue:** Anonymous work stored in sessionStorage (browser-only)
- Lost on tab close
- Not synced across tabs
- No server-side backup
**Impact:** User loses work if browser crashes
**Potential Fix:** Add localStorage fallback + periodic autosave

### 4. **Tool Execution Timing**
**Issue:** `onToolCall` fires before tool executes
```typescript
onToolCall: ({ toolCall }) => {
  handleToolCall(toolCall);  // Updates UI immediately
  // But Claude hasn't confirmed success yet
}
```
**Impact:** UI shows changes before Claude tool result returned
**Better Pattern:** Show optimistic UI, revert on error

### 5. **Claude Model Limitations**
- **Model Used:** claude-haiku-4-5 (fast but less capable)
- **maxSteps: 40** → May not be enough for very complex components
- **maxTokens: 10,000** → Long components might be truncated
- **Fallback to Mock:** If API key missing, returns canned responses (not ideal for unknown requests)

**Impact:** Complex components may fail to generate or be incomplete

### 6. **File Tree Refresh Inefficiency**
**Issue:** Uses `refreshTrigger` counter + key to force re-renders
```typescript
const [refreshTrigger, setRefreshTrigger] = useState(0);
// ...
<div key={refreshTrigger}>  {/* Full tree re-renders */}
```
**Impact:** Larger file trees become slow
**Better:** Use React.memo + useMemo for tree nodes

### 7. **Middleware Protection Gaps**
**Issue:** Middleware only protects `/api/projects` and `/api/filesystem` routes
- `/api/chat` is publicly accessible
- Anonymous users can make unlimited requests
**Impact:** Potential abuse/API quota exhaustion
**Fix:** Add rate limiting + anonymous user quotas

### 8. **No Error Recovery Strategy**
**Issue:** If Claude tool call fails, state might become inconsistent
```typescript
// If str_replace fails midway:
// - Chat shows it worked
// - File system doesn't update
// - Preview breaks silently
```
**Impact:** Confusing user experience
**Solution:** Add transaction-like rollback for failed tool calls

### 9. **Preview Frame Sandbox Limitations**
**Issue:** Preview uses iframe sandbox with `allow-scripts allow-same-origin`
- Can't directly communicate with parent window
- Limited error visibility
- Import Map limitations
**Impact:** Complex components may fail silently in preview

### 10. **No Automated Testing for E2E**
**Issue:** Unit tests exist but no E2E tests for:
- Chat → Code generation → Preview flow
- Project save/load cycle
- Multi-step component iterations
**Impact:** Regressions in core workflow not caught

---

## 10. Development Workflow: Adding a New Feature

### Example: "Add Component Favorites" Feature

#### Phase 1: Database Schema
```typescript
// 1. Add to Prisma schema
model Favorite {
  id        String    @id @default(cuid())
  projectId String
  project   Project   @relation(fields: [projectId], references: [id])
  createdAt DateTime  @default(now())
}

// 2. Create migration
npx prisma migrate dev --name add_favorites
```

#### Phase 2: Backend (Server Actions)
```typescript
// src/actions/favorites.ts
"use server";

export async function addFavorite(projectId: string) {
  const session = await getSession();
  if (!session) throw new Error("Unauthorized");
  
  return await prisma.favorite.create({
    data: {
      projectId,
      // Validate project ownership
    },
  });
}

export async function removeFavorite(projectId: string) {
  // Similar pattern
}
```

#### Phase 3: State Management
```typescript
// src/lib/contexts/favorites-context.tsx
"use client";

interface FavoritesContextType {
  favorites: string[];  // projectIds
  addFavorite: (projectId: string) => Promise<void>;
  removeFavorite: (projectId: string) => Promise<void>;
}

export function FavoritesProvider({ children }: { children: ReactNode }) {
  const [favorites, setFavorites] = useState<string[]>([]);
  
  // Load on mount
  useEffect(() => {
    getFavorites().then(setFavorites);
  }, []);
  
  return (
    <FavoritesContext.Provider value={{...}}>
      {children}
    </FavoritesContext.Provider>
  );
}
```

#### Phase 4: UI Components
```typescript
// src/components/FavoriteButton.tsx
"use client";

export function FavoriteButton({ projectId }: { projectId: string }) {
  const { favorites, addFavorite, removeFavorite } = useFavorites();
  const isFavorited = favorites.includes(projectId);
  
  return (
    <button
      onClick={() => isFavorited ? removeFavorite(projectId) : addFavorite(projectId)}
    >
      {isFavorited ? <StarFilled /> : <Star />}
    </button>
  );
}
```

#### Phase 5: Update Layout
```typescript
// src/app/main-content.tsx
export function MainContent({ user, project }: MainContentProps) {
  return (
    <FavoritesProvider>
      <FileSystemProvider>
        <ChatProvider>
          {/* Existing JSX with FavoriteButton added */}
        </ChatProvider>
      </FileSystemProvider>
    </FavoritesProvider>
  );
}
```

#### Phase 6: Tests
```typescript
// src/components/__tests__/FavoriteButton.test.tsx
import { test, expect, vi } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { FavoriteButton } from "../FavoriteButton";

vi.mock("@/lib/contexts/favorites-context");

test("toggles favorite on click", async () => {
  const user = userEvent.setup();
  const mockAdd = vi.fn();
  
  render(<FavoriteButton projectId="123" />);
  
  const button = screen.getByRole("button");
  await user.click(button);
  
  expect(mockAdd).toHaveBeenCalledWith("123");
});
```

#### Phase 7: Deployment Checklist
```bash
npm run test              # ✓ All tests pass
npm run build             # ✓ Build succeeds
npm run lint              # ✓ No linting errors
npx prisma migrate deploy # ✓ DB migrations applied
npm run start             # ✓ Production build starts
```

### Typical Developer Tasks

| Task | Location | Pattern |
|------|----------|---------|
| **Add UI Component** | `src/components/` | Create `.tsx`, co-locate `.test.tsx` |
| **Create API Endpoint** | `src/app/api/*/route.ts` | Use `streamText()` for streaming, verify auth |
| **Add Server Action** | `src/actions/*.ts` | Use `"use server"`, validate session |
| **Add Context** | `src/lib/contexts/` | Provider pattern, test with mocks |
| **Modify Database** | `prisma/schema.prisma` | Add field, run `npx prisma migrate dev --name desc` |
| **Add Utility** | `src/lib/` | Keep pure, export functions, test separately |
| **Hook/Tool Definition** | `src/lib/` or `src/hooks/` | Export reusable logic, document params |

---

## Quick Reference

### Import Aliases
```typescript
@/* → src/*
```

### Key Files to Know
| File | Purpose |
|------|---------|
| `src/app/layout.tsx` | Root layout wrapper |
| `src/app/page.tsx` | Home page (auth guard, project redirect) |
| `src/app/[projectId]/page.tsx` | Project page |
| `src/app/api/chat/route.ts` | Claude streaming endpoint |
| `src/lib/prompts/generation.tsx` | Claude system prompt |
| `src/lib/file-system.ts` | Virtual file system class |
| `src/middleware.ts` | Auth middleware |
| `src/lib/auth.ts` | JWT session management |
| `prisma/schema.prisma` | Database schema |

### Debug Tips
```bash
# Check logs
tail -f logs.txt  # From npm run dev:daemon

# Test in isolation
npm run test -- src/components/chat/ChatInterface.test.tsx

# Reset database
npm run db:reset

# Check database
npx prisma studio  # Visual DB explorer (if configured)

# Inspect generated Prisma client
cat src/generated/prisma/index.d.ts
```

---

## Summary

UIGen is a well-structured **AI-powered code generator** with clear separation between:
- **UI Layer** (Chat, Editor, Preview) using Context + React hooks
- **API Layer** (Streaming Claude integration with tools)
- **Data Layer** (Prisma + SQLite, with in-memory virtual FS)

**Strengths:**
✅ Modular component architecture  
✅ Type-safe with TypeScript strict mode  
✅ Responsive UI with Tailwind + Radix  
✅ Clear convention patterns  
✅ Test infrastructure in place  

**Areas for Improvement:**
⚠️ Error handling/recovery incomplete  
⚠️ Rate limiting missing  
⚠️ JSON serialization fragile  
⚠️ E2E testing gaps  
⚠️ Anonymous sessions not persistent  

For AI agents working in this codebase: Follow the established patterns, test rigorously, handle JSON edge cases, and always verify auth before database operations.
