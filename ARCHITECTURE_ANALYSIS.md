# Vibe Kanban - Architecture & Extensibility Analysis

**Generated:** 2025-11-06
**Branch:** `claude/analyze-architecture-codebase-011CUrBZZp9prZZJkcrpZU7K`

---

## Table of Contents

1. [Architecture & Extensibility](#1-architecture--extensibility)
   - [1.1 Overall Architecture](#11-overall-architecture)
   - [1.2 Code Quality & Maintainability](#12-code-quality--maintainability)
2. [Data Model & State Management](#2-data-model--state-management)
   - [2.1 Data Structure](#21-data-structure)
   - [2.2 State Management](#22-state-management)

---

## 1. Architecture & Extensibility

### 1.1 Overall Architecture

#### What is the high-level architecture?

**Stack:** React 18 + TypeScript + Vite + Context API + React Query + Zustand

**Pattern:** Multi-layered state management with real-time WebSocket updates:

```
┌──────────────────────────────────────────────────────────┐
│ Presentation Layer (React Components)                    │
│  - TaskKanbanBoard, TaskCard, TaskPanel                  │
│  - Dialogs, Settings, Layout components                  │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│ State Management Layer                                    │
│  - Context API (App-level config, current project)       │
│  - React Query (Server state + caching)                  │
│  - Zustand stores (Local UI state)                       │
│  - WebSocket (Real-time task updates via JSON Patch)     │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│ API Client Layer                                          │
│  - Typed API namespaces (projectsApi, tasksApi, etc.)    │
│  - Clerk authentication headers                          │
│  - Error handling with ApiError class                    │
└──────────────────────────────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│ Backend (Rust + Axum + SQLite)                           │
│  - REST API + WebSocket endpoints                        │
│  - Server-Sent Events for logs                           │
│  - Git worktree management                               │
└──────────────────────────────────────────────────────────┘
```

**Provider Hierarchy:**

```typescript
<React.StrictMode>
  <QueryClientProvider>          // React Query for server state
    <PostHogProvider>            // Analytics
      <ClerkProvider>            // Authentication
        <Sentry.ErrorBoundary>   // Error tracking
          <BrowserRouter>        // Routing
            <UserSystemProvider> // App config, executors, capabilities
              <ClickedElementsProvider>
                <ProjectProvider>    // Current project context
                  <HotkeysProvider>  // Keyboard shortcuts
                    <NiceModal.Provider>
                      <AppContent /> // Routes
```

**File Structure:** Component-based with some feature grouping

```
frontend/src/
├── components/
│   ├── tasks/              # Task-related components
│   │   ├── TaskKanbanBoard.tsx
│   │   ├── TaskCard.tsx
│   │   └── SharedTaskCard.tsx
│   ├── panels/             # Detail panel components
│   │   ├── TaskPanel.tsx
│   │   ├── TaskAttemptPanel.tsx
│   │   └── DiffsPanel.tsx
│   ├── dialogs/            # Modal dialogs
│   ├── layout/             # Layout components
│   └── ui/                 # Shadcn/ui component library
├── pages/                  # Route pages (feature-based)
│   ├── projects.tsx
│   ├── project-tasks.tsx   # Main kanban page
│   └── settings/
├── hooks/                  # Custom React hooks (feature-agnostic)
├── contexts/               # Context providers
├── stores/                 # Zustand stores
└── lib/                    # Utilities & API client
    ├── api.ts
    ├── paths.ts
    └── utils.ts
```

#### State Management Implementation

**Location:** Distributed across:
- `frontend/src/contexts/` - Context providers
- `frontend/src/stores/` - Zustand stores
- `frontend/src/components/config-provider.tsx` - UserSystemProvider
- `frontend/src/hooks/` - Custom hooks with React Query

**State Management Libraries:**
1. **Context API** - App-level configuration and current project
2. **React Query** (@tanstack/react-query) - Server state with caching
3. **Zustand** - Local UI state (non-persisted by default)
4. **WebSocket + JSON Patch** - Real-time task updates (RFC6902)

**Key Context Providers:**

| Provider | Purpose | Hook |
|----------|---------|------|
| `UserSystemProvider` | App config, executor profiles, capabilities | `useUserSystem()` |
| `ProjectProvider` | Current project (from URL params) | `useProject()` |
| `SearchProvider` | Global search query state | `useSearch()` |
| `ExecutionProcessesProvider` | Running process updates | `useExecutionProcesses()` |
| `ReviewProvider` | Code review state | `useReview()` |
| `HotkeysProvider` | Keyboard shortcuts | - |

#### Is there a plugin/extension API?

**No formal plugin API**, but clear extension points:

1. **Executor Plugins** (backend):
   - Location: `crates/executors/`
   - Can add new AI agent integrations (Claude, Gemini, etc.)
   - Each implements common interface

2. **MCP Server Integration** (backend):
   - Vibe Kanban acts as MCP server
   - Tools: `list_projects`, `list_tasks`, `create_task`, etc.
   - External agents can manage tasks via MCP protocol

3. **Component Extension**:
   - Dialog system via `nice-modal-react` - register new dialogs
   - Layout panels are swappable (preview, diffs, custom)
   - Context providers can be added

**Example: Adding a New Panel Type**

```typescript
// project-tasks.tsx already supports dynamic modes
const [mode, setMode] = useState<'preview' | 'diffs' | null>(null);

// Easy to extend:
type PanelMode = 'preview' | 'diffs' | 'commits' | 'reviews' | null;

// Add new panel component:
{mode === 'commits' && <CommitsPanel taskId={taskId} />}
```

**Example: Custom Data Source (Theoretical)**

Currently **not supported** without modification, but could be implemented via:

```typescript
// THEORETICAL - Not currently implemented
interface DataSourcePlugin {
  loadTasks: (projectId: string) => Promise<Task[]>;
  saveTask: (task: Task) => Promise<void>;
  onTaskUpdate: (callback: (task: Task) => void) => Unsubscribe;
}

// Could be passed via Context:
<DataSourceProvider source={myCustomDataSource}>
  <App />
</DataSourceProvider>
```

#### How modular is the codebase?

**Modularity Score: 7.5/10**

**Highly Modular:**
- ✅ Components are small and focused
- ✅ Custom hooks separate data fetching from UI
- ✅ API client is centralized in `lib/api.ts`
- ✅ Layout components accept children (composable)
- ✅ TaskCard, TaskPanel can be used independently
- ✅ Dialog system allows triggering modals from anywhere

**Moderate Coupling:**
- ⚠️ TaskKanbanBoard expects specific data shape (TaskWithAttemptStatus)
- ⚠️ Panels assume certain data structures
- ⚠️ Some business logic mixed in component files (medium-sized components)

**Can components be used standalone?**

Yes, with caveats:

```typescript
// TaskCard can be used outside kanban:
<TaskCard
  task={task}
  onClick={handleClick}
  isHighlighted={false}
/>

// But requires UserSystemProvider context:
<UserSystemProvider>
  <TaskCard task={task} />
</UserSystemProvider>
```

**Business Logic Separation:**

| Concern | Location | Separation Quality |
|---------|----------|-------------------|
| API calls | `lib/api.ts` | ✅ Excellent |
| Data fetching | `hooks/` | ✅ Excellent |
| State management | `contexts/`, `stores/` | ✅ Excellent |
| UI rendering | `components/` | ⚠️ Good (some logic mixed) |
| Routing | `App.tsx`, `pages/` | ✅ Excellent |

**Module Boundaries:**

Clear boundaries between:
- API layer (`lib/api.ts`)
- State layer (`contexts/`, `stores/`, `hooks/`)
- UI layer (`components/`, `pages/`)

#### What's the component hierarchy?

**Top-Level Structure:**

```
App.tsx (BrowserRouter + Routes)
  ├─ Route: "/" → Projects (list page)
  ├─ Route: "/projects/:projectId" → Projects (detail page)
  └─ Route: "/projects/:projectId/tasks" → ProjectTasks
      └─ Route: "/projects/:projectId/tasks/:taskId" → ProjectTasks (with panel)
          └─ Route: "/projects/:projectId/tasks/:taskId/attempts/:attemptId" → ProjectTasks
```

**ProjectTasks Page Component Tree:**

```
ProjectTasks
├─ TasksLayout (responsive two-pane layout)
│   ├─ Left Pane (width: 100% - rightPanelSize)
│   │   └─ TaskKanbanBoard
│   │       ├─ SearchBar (global search)
│   │       ├─ KanbanBoard (status: "todo")
│   │       │   ├─ KanbanCard (wrapper for dnd-kit)
│   │       │   │   └─ TaskCard (task item)
│   │       │   │       ├─ TaskStatusBadge
│   │       │   │       ├─ Title & description
│   │       │   │       ├─ ExecutorBadge
│   │       │   │       └─ AttemptStatusIndicators
│   │       │   └─ [More KanbanCards...]
│   │       ├─ KanbanBoard (status: "inprogress")
│   │       │   └─ [TaskCards or SharedTaskCards...]
│   │       ├─ KanbanBoard (status: "inreview")
│   │       └─ KanbanBoard (status: "done")
│   │
│   ├─ Right Pane (width: rightPanelSize, resizable)
│   │   ├─ PanelGroup (react-resizable-panels)
│   │   │   ├─ Panel: Primary (default 50%)
│   │   │   │   └─ [TaskPanel OR TaskAttemptPanel]
│   │   │   │       ├─ TaskPanel (if only taskId)
│   │   │   │       │   ├─ TaskPanelHeaderActions
│   │   │   │       │   │   ├─ Edit button → TaskFormDialog
│   │   │   │       │   │   ├─ Delete button → DeleteTaskDialog
│   │   │   │       │   │   ├─ Share button → ShareTaskDialog
│   │   │   │       │   │   └─ Run button → Creates new attempt
│   │   │   │       │   ├─ Task metadata (executor, status)
│   │   │   │       │   ├─ Task description
│   │   │   │       │   └─ Attempts list
│   │   │   │       │       └─ AttemptCard (clickable)
│   │   │   │       │
│   │   │   │       └─ TaskAttemptPanel (if taskId + attemptId)
│   │   │   │           ├─ AttemptHeaderActions
│   │   │   │           │   ├─ Status badge
│   │   │   │           │   ├─ Approve/Reject buttons
│   │   │   │           │   ├─ Stop button
│   │   │   │           │   └─ Actions menu (merge, rebase, push)
│   │   │   │           ├─ Tabs (Processes, Conversation)
│   │   │   │           │   ├─ ProcessesTab
│   │   │   │           │   │   └─ ProcessLogViewer
│   │   │   │           │   │       ├─ Process selector
│   │   │   │           │   │       └─ ANSI-rendered logs
│   │   │   │           │   └─ ConversationDisplay
│   │   │   │           │       └─ ConversationMessage[]
│   │   │   │           └─ TaskFollowUpSection
│   │   │   │               ├─ Follow-up input
│   │   │   │               └─ Submit button
│   │   │   │
│   │   │   └─ Panel: Auxiliary (default 50%, conditional)
│   │   │       ├─ DiffsPanel (if mode === 'diffs')
│   │   │       │   ├─ FileDiffView[]
│   │   │       │   └─ FileDiffRenderer (syntax-highlighted)
│   │   │       │
│   │   │       └─ PreviewPanel (if mode === 'preview')
│   │   │           ├─ Dev server status
│   │   │           └─ Iframe with preview URL
```

**Key Component Relationships:**

| Parent | Child | Coupling |
|--------|-------|----------|
| `TaskKanbanBoard` | `KanbanBoard` | Loose - passes columns |
| `KanbanBoard` | `KanbanCard` | Loose - dnd-kit wrapper |
| `KanbanCard` | `TaskCard` | Loose - passes task |
| `TaskPanel` | `TaskPanelHeaderActions` | Moderate - task context |
| `TaskAttemptPanel` | `ProcessLogViewer` | Moderate - attempt context |
| `ProjectTasks` | `TasksLayout` | Tight - layout structure |

**Tightly Coupled:** TasksLayout ↔ ProjectTasks (layout structure)
**Loosely Coupled:** TaskCard, TaskPanel (can be used independently)

---

### 1.2 Code Quality & Maintainability

#### TypeScript Coverage

**100% TypeScript** - No JavaScript files in frontend

**Configuration:** `frontend/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "strict": true,                    // ✅ Full strict mode
    "noUnusedLocals": true,            // ✅ Catch unused variables
    "noUnusedParameters": true,        // ✅ Catch unused params
    "noFallthrowCasesInSwitch": true,  // ✅ Exhaustive switch
    "jsx": "react-jsx"
  }
}
```

**Type Generation:** Types are auto-generated from Rust using `ts-rs`

**Type Safety Analysis:**

```bash
# Ran: grep -r "any" frontend/src/ --include="*.ts" --include="*.tsx"
# Result: ~15 instances of `any` type (manual review)
```

**`any` Type Usage:**

| File | Context | Reason |
|------|---------|--------|
| `api.ts` | Error handling | Third-party API response types |
| Various component files | Event handlers | React event types (could be improved) |
| Utility functions | Generic helpers | Intentional escape hatch |

**Overall Type Safety: 9/10** - Strong types, minimal `any` usage

#### Is the code documented?

**Documentation Coverage:**

1. **JSDoc/TSDoc:** Minimal - mostly in complex utility functions
2. **README:** Yes - Comprehensive setup guide in root `README.md`
3. **CLAUDE.md:** Yes - Architecture overview for AI assistants
4. **Inline comments:** Moderate - complex logic is commented

**Example Documentation:**

```typescript
// Good: lib/api.ts has detailed error handling
/**
 * Makes an authenticated API request using Clerk auth headers
 */
const makeRequest = async (url: string, options?: RequestInit) => {
  // ... implementation
}

// Moderate: Some components have brief descriptions
// frontend/src/components/tasks/TaskKanbanBoard.tsx
/**
 * Main kanban board component for displaying tasks by status
 */
export const TaskKanbanBoard = () => { ... }

// Poor: Many components lack JSDoc
// frontend/src/components/panels/TaskPanel.tsx (no JSDoc)
```

**Architecture Documentation:**

- ✅ `CLAUDE.md` - Excellent architectural overview
- ✅ `README.md` - Setup and development guide
- ⚠️ No formal API documentation (could use TypeDoc)
- ⚠️ No component library documentation (could use Storybook)

**Getting Started Guide:**

Location: `/home/user/vibe-kanban/README.md`

Includes:
- Installation steps
- Development server commands
- Build instructions
- Tech stack overview

---

## 2. Data Model & State Management

### 2.1 Data Structure

#### Kanban Data Model

**Core Types:** Located in `shared/types.ts` (auto-generated from Rust)

```typescript
// Task Entity
export type Task = {
  id: string;                         // UUID format
  project_id: string;
  title: string;
  description: string | null;
  status: TaskStatus;                 // "todo" | "inprogress" | "inreview" | "done" | "cancelled"
  parent_task_attempt: string | null; // Optional parent relationship
  shared_task_id: string | null;      // Link to shared task (for team collaboration)
  created_at: string;                 // ISO 8601 timestamp
  updated_at: string;
};

// Task Status Enum
export type TaskStatus =
  | "todo"
  | "inprogress"
  | "inreview"
  | "done"
  | "cancelled";

// Enhanced Task with Attempt Status
export type TaskWithAttemptStatus = Task & {
  has_in_progress_attempt: boolean;
  has_merged_attempt: boolean;
  last_attempt_failed: boolean;
  executor: string;                   // e.g., "claude-sonnet-3-5"
};

// Shared Task (for team collaboration)
export type SharedTask = {
  id: string;
  organization_id: string;
  project_id: string | null;
  title: string;
  status: TaskStatus;
  assignee_user_id: string | null;
  assignee_first_name: string | null;
  assignee_last_name: string | null;
  assignee_username: string | null;
  version: bigint;                    // Optimistic locking
  last_event_seq: bigint | null;
  created_at: Date;
  updated_at: Date;
};

// Project Entity
export type Project = {
  id: string;
  name: string;
  git_repo_path: string;              // Absolute path to repo
  setup_script: string | null;        // Pre-execution script
  dev_script: string | null;          // Dev server command
  cleanup_script: string | null;      // Post-execution cleanup
  copy_files: string | null;          // Files to copy to worktree
  has_remote: boolean;
  github_repo_owner: string | null;
  github_repo_name: string | null;
  created_at: Date;
  updated_at: Date;
};

// Task Attempt (execution)
export type TaskAttempt = {
  id: string;
  task_id: string;
  status: TaskAttemptStatus;          // "pending" | "running" | "completed" | "failed" | ...
  branch_name: string;                // Git branch for this attempt
  worktree_path: string | null;       // Isolated worktree path
  executor_profile_id: string;
  action: string;                     // "coding_agent_initial" | "coding_agent_follow_up" | ...
  error_message: string | null;
  created_at: Date;
  updated_at: Date;
};

export type TaskAttemptStatus =
  | "pending"
  | "running"
  | "completed"
  | "failed"
  | "cancelled"
  | "awaiting_approval"
  | "approved"
  | "rejected"
  | "merged";
```

**Kanban Column Structure:**

```typescript
// File: frontend/src/components/tasks/TaskKanbanBoard.tsx

type KanbanColumnItem =
  | { type: 'task'; task: TaskWithAttemptStatus; sharedTask?: SharedTaskRecord }
  | { type: 'shared'; task: SharedTaskRecord };

type KanbanColumns = Record<TaskStatus, KanbanColumnItem[]>;

// Example data structure:
const columns: KanbanColumns = {
  todo: [
    { type: 'task', task: { id: '1', title: 'Fix bug', status: 'todo', ... } },
    { type: 'shared', task: { id: 's1', title: 'Shared task', ... } }
  ],
  inprogress: [ ... ],
  inreview: [ ... ],
  done: [ ... ],
  cancelled: [ ... ]
};
```

#### Example JSON Structure

**Task:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "project_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "title": "Implement user authentication",
  "description": "Add JWT-based authentication with refresh tokens",
  "status": "inprogress",
  "parent_task_attempt": null,
  "shared_task_id": null,
  "created_at": "2025-11-06T10:30:00Z",
  "updated_at": "2025-11-06T12:45:00Z"
}
```

**TaskWithAttemptStatus:**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "project_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "title": "Implement user authentication",
  "description": "Add JWT-based authentication with refresh tokens",
  "status": "inprogress",
  "parent_task_attempt": null,
  "shared_task_id": null,
  "created_at": "2025-11-06T10:30:00Z",
  "updated_at": "2025-11-06T12:45:00Z",
  "has_in_progress_attempt": true,
  "has_merged_attempt": false,
  "last_attempt_failed": false,
  "executor": "claude-sonnet-4-5"
}
```

**Project:**

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "name": "My Web App",
  "git_repo_path": "/home/user/projects/my-web-app",
  "setup_script": "npm install",
  "dev_script": "npm run dev",
  "cleanup_script": null,
  "copy_files": ".env.example",
  "has_remote": true,
  "github_repo_owner": "myorg",
  "github_repo_name": "my-web-app",
  "created_at": "2025-10-15T08:00:00Z",
  "updated_at": "2025-11-06T09:20:00Z"
}
```

#### ID Format

**All IDs are UUIDs (v4):**
- Format: `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`
- Generated by backend (Rust's `uuid` crate)
- Immutable after creation

**Example:**
```typescript
const taskId = "550e8400-e29b-41d4-a716-446655440000"; // UUID v4
```

#### Where is data stored?

**Primary Storage: SQLite Database**

- Location: `~/.config/vibe-kanban/db.sqlite` (production)
- Development: `dev_assets_seed/` (auto-copied on dev server start)
- Managed by: SQLx ORM (Rust backend)
- Migrations: `crates/db/migrations/`

**No Frontend Storage:**
- ❌ No localStorage
- ❌ No IndexedDB
- ❌ No sessionStorage
- ✅ All state is ephemeral or fetched from backend

**Caching:**
- React Query caches server state (5 minute default stale time)
- Browser HTTP cache (standard cache headers)

**Can we override the storage layer?**

**Backend:** Yes - Database abstraction via SQLx

```rust
// crates/db/src/models/task.rs
impl Task {
    pub async fn create(pool: &SqlitePool, data: CreateTask) -> Result<Task>
    pub async fn get_by_id(pool: &SqlitePool, id: &str) -> Result<Option<Task>>
    pub async fn update(pool: &SqlitePool, id: &str, updates: UpdateTask) -> Result<Task>
}

// Could swap SQLite for PostgreSQL:
// 1. Change SQLx features in Cargo.toml
// 2. Update connection string
// 3. Rewrite migrations for Postgres
```

**Frontend:** Not directly - API client is hard-coded

```typescript
// lib/api.ts
const BASE_URL = '/api'; // Hardcoded

// To support custom backend:
// 1. Make BASE_URL configurable via env var
// 2. Add authentication strategy abstraction
// 3. Allow custom headers/interceptors
```

---

### 2.2 State Management

#### How is state initialized?

**Initialization Flow:**

```
1. User navigates to app
   ↓
2. main.tsx renders providers (QueryClient, Clerk, etc.)
   ↓
3. App.tsx renders UserSystemProvider
   ├─ Fetches /api/config (app config)
   ├─ Fetches /api/executors/profiles (executor configs)
   └─ Fetches /api/environment (backend environment info)
   ↓
4. User navigates to /projects/:projectId/tasks
   ↓
5. ProjectProvider extracts projectId from URL params
   ├─ Calls projectsApi.getById(projectId)
   └─ Stores in React Query cache + context
   ↓
6. ProjectTasks page calls useProjectTasks(projectId)
   ├─ Opens WebSocket: /api/tasks/stream/ws?project_id=X
   ├─ Receives initial JSON Patch with full task list
   ├─ Applies patches using rfc6902 library
   └─ State stored in component (useMemo hook)
   ↓
7. Tasks grouped by status into KanbanColumns
   ↓
8. TaskKanbanBoard renders columns and cards
```

**Initial Data Sources:**

| State | Source | Hook/Context |
|-------|--------|--------------|
| App config | `GET /api/config` | `useUserSystem()` |
| Executor profiles | `GET /api/executors/profiles` | `useUserSystem()` |
| Projects list | `GET /api/projects` | `useProjects()` (React Query) |
| Current project | `GET /api/projects/:id` | `useProject()` |
| Tasks | `WS /api/tasks/stream/ws` | `useProjectTasks()` |
| Task attempts | `GET /api/task-attempts/:id` | `useTaskAttempt()` (React Query) |

**Can we pass initial state as props?**

**Limited:**

```typescript
// UserSystemProvider accepts no props - fetches internally
<UserSystemProvider>
  <App />
</UserSystemProvider>

// ProjectProvider reads from URL params - no prop override
const { projectId } = useParams<{ projectId: string }>();

// CANNOT do:
<ProjectProvider projectId="custom-id"> ❌
```

**To support initial state:**

```typescript
// THEORETICAL - Not currently implemented
interface UserSystemProviderProps {
  initialConfig?: Config;
  initialProfiles?: ExecutorConfig[];
}

export const UserSystemProvider = ({
  initialConfig,
  initialProfiles,
  children
}: UserSystemProviderProps) => {
  // Use initialConfig if provided, otherwise fetch
  const [config, setConfig] = useState(initialConfig);

  useEffect(() => {
    if (!initialConfig) {
      fetchConfig().then(setConfig);
    }
  }, []);

  // ...
}
```

#### How is state updated?

**Update Mechanisms:**

1. **API Mutations (via React Query or direct fetch):**

```typescript
// Example: Update task status
const updateTaskStatus = async (taskId: string, status: TaskStatus) => {
  const result = await tasksApi.update(taskId, { status });
  // React Query will invalidate cache and refetch
  return result;
};

// Usage in component:
const handleDragEnd = async (event: DragEndEvent) => {
  const newStatus = event.over?.id as TaskStatus;
  await updateTaskStatus(taskId, newStatus);
  // WebSocket will push update, triggering re-render
};
```

2. **WebSocket JSON Patch Updates:**

```typescript
// Hook: useJsonPatchWsStream.ts
const useProjectTasks = (projectId: string) => {
  const [tasks, setTasks] = useState<Task[]>([]);

  useJsonPatchWsStream({
    url: `/api/tasks/stream/ws?project_id=${projectId}`,
    onMessage: (patches: JsonPatchOperation[]) => {
      setTasks(prevTasks => {
        const newTasks = applyPatch(prevTasks, patches).newDocument;
        return newTasks;
      });
    }
  });

  return tasks;
};

// Patches are RFC6902 JSON Patch format:
[
  { "op": "replace", "path": "/0/status", "value": "done" },
  { "op": "add", "path": "/-", "value": { "id": "new", ... } }
]
```

3. **Context Setters:**

```typescript
// UserSystemProvider exposes update function:
const { updateAndSaveConfig } = useUserSystem();

// Usage:
await updateAndSaveConfig({
  theme: 'dark',
  language: 'en'
});
// Triggers re-render of all consumers
```

4. **Zustand Store Updates:**

```typescript
// stores/useTaskDetailsUiStore.ts
export const useTaskDetailsUiStore = create<TaskDetailsUiStore>((set) => ({
  ui: {},
  setTaskUi: (taskId, updates) => set((state) => ({
    ui: {
      ...state.ui,
      [taskId]: { ...state.ui[taskId], ...updates }
    }
  }))
}));

// Usage:
const { setTaskUi } = useTaskDetailsUiStore();
setTaskUi(taskId, { isExpanded: true });
```

**Update Triggers Re-renders:**

| State Type | Update Mechanism | Re-render Trigger |
|------------|------------------|-------------------|
| Context API | `setState` in provider | All context consumers |
| React Query | Cache invalidation | All queries with same key |
| Zustand | `set()` function | All hook consumers |
| WebSocket | JSON Patch applied | Component `useState` |
| Component state | `setState` | Component + children |

#### Can we inject a custom data source?

**Current Status: ❌ Not supported without code changes**

**What would need to change:**

1. **Abstract the API client:**

```typescript
// THEORETICAL IMPLEMENTATION

// 1. Define data source interface
interface TaskDataSource {
  getTasks(projectId: string): Promise<Task[]>;
  createTask(data: CreateTask): Promise<Task>;
  updateTask(id: string, updates: UpdateTask): Promise<Task>;
  deleteTask(id: string): Promise<void>;
  subscribeToUpdates(
    projectId: string,
    onUpdate: (tasks: Task[]) => void
  ): Unsubscribe;
}

// 2. Create context for data source
const DataSourceContext = createContext<TaskDataSource | null>(null);

export const DataSourceProvider = ({
  source,
  children
}: {
  source: TaskDataSource;
  children: ReactNode
}) => {
  return (
    <DataSourceContext.Provider value={source}>
      {children}
    </DataSourceContext.Provider>
  );
};

// 3. Update hooks to use injected source
const useProjectTasks = (projectId: string) => {
  const dataSource = useContext(DataSourceContext);
  const [tasks, setTasks] = useState<Task[]>([]);

  useEffect(() => {
    // Use custom data source instead of hardcoded WebSocket
    const unsubscribe = dataSource.subscribeToUpdates(projectId, setTasks);
    return unsubscribe;
  }, [projectId, dataSource]);

  return tasks;
};

// 4. Usage
const myCustomDataSource: TaskDataSource = {
  getTasks: async (projectId) => {
    return fetch(`https://mybackend.com/projects/${projectId}/tasks`)
      .then(r => r.json());
  },
  subscribeToUpdates: (projectId, onUpdate) => {
    const eventSource = new EventSource(`https://mybackend.com/sse/tasks/${projectId}`);
    eventSource.onmessage = (e) => onUpdate(JSON.parse(e.data));
    return () => eventSource.close();
  },
  // ... implement other methods
};

<DataSourceProvider source={myCustomDataSource}>
  <App />
</DataSourceProvider>
```

2. **Make BASE_URL configurable:**

```typescript
// lib/api.ts
const BASE_URL = import.meta.env.VITE_API_BASE_URL || '/api';

// .env
VITE_API_BASE_URL=https://mybackend.com/api
```

3. **Abstract authentication:**

```typescript
// THEORETICAL
interface AuthStrategy {
  getHeaders(): Promise<HeadersInit>;
}

class ClerkAuthStrategy implements AuthStrategy {
  async getHeaders() {
    return buildClerkAuthHeaders();
  }
}

class CustomAuthStrategy implements AuthStrategy {
  async getHeaders() {
    return { 'Authorization': `Bearer ${myToken}` };
  }
}

// Inject via context
<AuthProvider strategy={new CustomAuthStrategy()}>
  <App />
</AuthProvider>
```

**Example Pattern (if implemented):**

```typescript
// This would work if the above abstractions were implemented:

<DataSourceProvider source={myBackendDataSource}>
  <AuthProvider strategy={myAuthStrategy}>
    <App />
  </AuthProvider>
</DataSourceProvider>

// Or config-based:
const config = {
  dataSource: {
    type: 'rest',
    baseUrl: 'https://mybackend.com/api',
    auth: { type: 'bearer', token: 'xxx' }
  }
};

<KanbanApp config={config} />
```

---

## Summary

### Architecture Strengths

1. ✅ **Real-time updates** via WebSocket with automatic reconnection
2. ✅ **Type safety** with auto-generated types from Rust
3. ✅ **Modular components** with clear separation of concerns
4. ✅ **Multi-layer state management** (Context, React Query, Zustand, WebSocket)
5. ✅ **Responsive layout** with resizable panels
6. ✅ **Centralized API client** with error handling
7. ✅ **Extensible dialog system** via nice-modal-react

### Extensibility Limitations

1. ❌ **No plugin API** - requires code changes to extend
2. ❌ **Hard-coded data source** - cannot inject custom backend without modification
3. ❌ **Tight coupling** to Vibe Kanban backend (Rust + SQLite)
4. ⚠️ **Limited component reusability** - some components expect specific context providers

### Recommended Extension Points

If you want to extend Vibe Kanban without forking:

1. **MCP Server Integration** - Use MCP tools to manage tasks externally
2. **Executor Plugins** - Add new AI agent integrations (backend)
3. **Dialog System** - Register custom modal dialogs
4. **Layout Panels** - Add new panel modes (commits, reviews, etc.)
5. **Context Providers** - Add domain-specific state providers

### Key Files Reference

| Concept | File Path |
|---------|-----------|
| Main entry | `/home/user/vibe-kanban/frontend/src/main.tsx` |
| App component | `/home/user/vibe-kanban/frontend/src/App.tsx` |
| Kanban board | `/home/user/vibe-kanban/frontend/src/components/tasks/TaskKanbanBoard.tsx` |
| Task card | `/home/user/vibe-kanban/frontend/src/components/tasks/TaskCard.tsx` |
| Task panel | `/home/user/vibe-kanban/frontend/src/components/panels/TaskPanel.tsx` |
| API client | `/home/user/vibe-kanban/frontend/src/lib/api.ts` |
| Type definitions | `/home/user/vibe-kanban/shared/types.ts` |
| Config provider | `/home/user/vibe-kanban/frontend/src/components/config-provider.tsx` |
| Project provider | `/home/user/vibe-kanban/frontend/src/contexts/ProjectProvider.tsx` |
| WebSocket hook | `/home/user/vibe-kanban/frontend/src/hooks/useJsonPatchWsStream.ts` |
| Task details store | `/home/user/vibe-kanban/frontend/src/stores/useTaskDetailsUiStore.ts` |

---

**End of Architecture Analysis**
