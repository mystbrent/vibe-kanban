# Drag & Drop Investigation - Vibe Kanban

## Executive Summary

This document provides a comprehensive investigation of the drag-and-drop implementation in the Vibe Kanban codebase, covering the library used, event handling, data structures, and extensibility options.

---

## 1. DnD Library

### Library: `@dnd-kit`

**Packages Used:**
- `@dnd-kit/core`: v6.3.1
- `@dnd-kit/modifiers`: v9.0.0
- `@dnd-kit/utilities`: v3.2.2

**Status:** ✅ Well-maintained, modern React drag-and-drop library
- Active development and community support
- Built specifically for React with hooks-first approach
- Better performance than older libraries like react-beautiful-dnd
- Fully accessible and keyboard-friendly

**Location:** `frontend/package.json:23-25`

---

## 2. Event Handling Architecture

### 2.1 Component Hierarchy

```
DndContext (KanbanProvider)
  └─ KanbanBoard (droppable columns)
      └─ KanbanCards
          └─ KanbanCard (draggable items)
```

### 2.2 Event Handler Location

**Primary Handler:** `handleDragEnd` in `/home/user/vibe-kanban/frontend/src/pages/project-tasks.tsx:730`

```typescript
const handleDragEnd = useCallback(
  async (event: DragEndEvent) => {
    const { active, over } = event;
    if (!over || !active.data.current) return;

    const draggedTaskId = active.id as string;
    const newStatus = over.id as Task['status'];
    const task = tasksById[draggedTaskId];
    if (!task || task.status === newStatus) return;

    try {
      await tasksApi.update(draggedTaskId, {
        title: task.title,
        description: task.description,
        status: newStatus,
        parent_task_attempt: task.parent_task_attempt,
        image_ids: null,
      });
    } catch (err) {
      console.error('Failed to update task status:', err);
    }
  },
  [tasksById]
);
```

**Data Flow:**
1. `project-tasks.tsx:730` - Defines `handleDragEnd` callback
2. `project-tasks.tsx:836` - Passes to `<TaskKanbanBoard onDragEnd={handleDragEnd} />`
3. `TaskKanbanBoard.tsx:53` - Passes to `<KanbanProvider onDragEnd={onDragEnd} />`
4. `kanban/index.tsx:282` - Connects to `<DndContext onDragEnd={onDragEnd} />`

---

## 3. Drag Event Data Structures

### 3.1 DragEndEvent Interface

```typescript
interface DragEndEvent {
  active: {
    id: string | number;           // Draggable item ID
    data: {
      current?: {                  // Custom data from useDraggable
        index: number;
        parent: string;
      }
    };
    rect: ClientRect;              // Position/dimensions of dragged element
  };
  over: {                          // null if not dropped over a droppable
    id: string | number;           // Droppable container ID
    data: {
      current?: any;               // Custom data from useDroppable
    };
    rect: ClientRect;
    disabled: boolean;
  } | null;
  activatorEvent?: Event;          // Original DOM event that triggered drag
  collisions?: Collision[];        // Collision detection results
  delta: {                         // Movement delta
    x: number;
    y: number;
  };
}
```

### 3.2 Custom Data Attached to Draggables

**Location:** `kanban/index.tsx:97`

```typescript
const { attributes, listeners, setNodeRef, transform, isDragging } = useDraggable({
  id,
  data: { index, parent },  // Custom data attached to drag events
  disabled: dragDisabled,
});
```

**Usage:** The `index` and `parent` are available in drag events via `event.active.data.current.index` and `event.active.data.current.parent`.

---

## 4. Event Callbacks & Extensibility

### 4.1 Currently Implemented Callbacks

| Callback | Implemented | Location |
|----------|-------------|----------|
| `onDragEnd` | ✅ Yes | `project-tasks.tsx:730` |
| `onDragStart` | ❌ No | N/A |
| `onDragMove` | ❌ No | N/A |
| `onDragOver` | ❌ No | N/A |
| `onDragCancel` | ❌ No | N/A |

### 4.2 Available @dnd-kit Event Callbacks

The `DndContext` component supports these event callbacks:

```typescript
<DndContext
  onDragStart={(event: DragStartEvent) => void}    // When drag starts
  onDragMove={(event: DragMoveEvent) => void}      // During drag movement
  onDragOver={(event: DragOverEvent) => void}      // When over droppable
  onDragEnd={(event: DragEndEvent) => void}        // When drag ends
  onDragCancel={() => void}                        // When drag is cancelled
>
```

### 4.3 How to Add New Callbacks

**Step 1:** Add callback prop to `KanbanProviderProps` in `kanban/index.tsx:262`

```typescript
export type KanbanProviderProps = {
  children: ReactNode;
  onDragEnd: (event: DragEndEvent) => void;
  onDragStart?: (event: DragStartEvent) => void;  // Add this
  className?: string;
};
```

**Step 2:** Pass to DndContext in `kanban/index.tsx:280`

```typescript
<DndContext
  collisionDetection={rectIntersection}
  onDragEnd={onDragEnd}
  onDragStart={onDragStart}  // Add this
  sensors={sensors}
  modifiers={[restrictToFirstScrollableAncestorCustom]}
>
```

**Step 3:** Wire through component hierarchy:
- `TaskKanbanBoard.tsx` - Add prop and pass through
- `project-tasks.tsx` - Implement handler

### 4.4 Monitoring Events Inside Components

Use the `useDndMonitor` hook anywhere within the DndContext:

```typescript
import { useDndMonitor } from '@dnd-kit/core';

function MyComponent() {
  useDndMonitor({
    onDragStart(event) {
      console.log('Drag started:', event.active.id);
    },
    onDragEnd(event) {
      console.log('Drag ended:', event.active.id, 'over', event.over?.id);
    },
  });
}
```

---

## 5. Event Prevention Patterns

### 5.1 Current Prevention Mechanism

**Early Return Pattern** - Used to cancel drag operations without updating state:

```typescript
const handleDragEnd = useCallback(
  async (event: DragEndEvent) => {
    const { active, over } = event;

    // Prevention #1: No valid drop target
    if (!over || !active.data.current) return;

    const draggedTaskId = active.id as string;
    const newStatus = over.id as Task['status'];
    const task = tasksById[draggedTaskId];

    // Prevention #2: Invalid task or no status change
    if (!task || task.status === newStatus) return;

    // Only proceed if all conditions pass
    await tasksApi.update(...);
  },
  [tasksById]
);
```

### 5.2 Can We Prevent Default Behavior?

**Answer:** Yes, through multiple mechanisms:

#### Option 1: Early Return (Current Approach)
- Return early from `onDragEnd` to skip state updates
- ✅ Simple and effective
- ❌ Drag animation still completes

#### Option 2: Conditional API Call
```typescript
const handleDragEnd = async (event: DragEndEvent) => {
  const { active, over } = event;

  // Custom validation logic
  if (shouldPreventDrop(active, over)) {
    console.log('Drop prevented by business logic');
    return;
  }

  // Proceed with update
  await tasksApi.update(...);
};
```

#### Option 3: Disable Dragging Conditionally
```typescript
const { attributes, listeners, setNodeRef } = useDraggable({
  id,
  data: { index, parent },
  disabled: dragDisabled,  // Prevent dragging entirely
});
```

**Location:** `kanban/index.tsx:92-99`

Current usage in TaskCard:
```typescript
<TaskCard
  dragDisabled={!isOwnTask || someOtherCondition}
  ...
/>
```

#### Option 4: Custom Collision Detection
Implement custom logic to prevent drops on certain targets:

```typescript
<DndContext
  collisionDetection={(args) => {
    const collisions = rectIntersection(args);
    // Filter out invalid drop targets
    return collisions.filter(collision => isValidDropTarget(collision));
  }}
>
```

### 5.3 Drag Activation Constraints

**Current Configuration:** `kanban/index.tsx:274-276`

```typescript
const sensors = useSensors(
  useSensor(PointerSensor, {
    activationConstraint: { distance: 8 },  // Must drag 8px to activate
  })
);
```

**Purpose:** Prevents accidental drags during clicks/scrolls

**Options Available:**
```typescript
activationConstraint: {
  distance: 8,              // Minimum pixels to drag
  // OR
  delay: 250,              // Minimum ms to hold before drag
  tolerance: 5,            // Allowed movement during delay
}
```

---

## 6. Drag Constraints & Modifiers

### 6.1 Custom Scroll Boundary Modifier

**Location:** `kanban/index.tsx:241-260`

A custom modifier `restrictToFirstScrollableAncestorCustom` prevents infinite horizontal scrolling:

```typescript
const restrictToFirstScrollableAncestorCustom: Modifier = (args) => {
  const { draggingNodeRect, transform, scrollableAncestorRects } = args;
  const firstScrollableAncestorRect = scrollableAncestorRects[0];

  if (!draggingNodeRect || !firstScrollableAncestorRect) {
    return transform;
  }

  // Inset right edge by 16px to prevent infinite scroll
  const rightPadding = 16;
  return restrictToBoundingRectWithRightPadding(
    transform,
    draggingNodeRect,
    firstScrollableAncestorRect,
    rightPadding
  );
};
```

**Applied At:** `kanban/index.tsx:284`
```typescript
<DndContext
  modifiers={[restrictToFirstScrollableAncestorCustom]}
>
```

### 6.2 Available @dnd-kit Modifiers

Can import from `@dnd-kit/modifiers`:
- `restrictToVerticalAxis`
- `restrictToHorizontalAxis`
- `restrictToWindowEdges`
- `restrictToParentElement`
- `restrictToFirstScrollableAncestor`
- `snapCenterToCursor`

---

## 7. Collision Detection

**Current Algorithm:** `rectIntersection`

**Location:** `kanban/index.tsx:281`

```typescript
<DndContext
  collisionDetection={rectIntersection}
  ...
>
```

**Available Algorithms:**
- `rectIntersection` - Default, checks bounding box overlap
- `closestCenter` - Finds droppable closest to drag center
- `closestCorners` - Finds droppable with closest corner
- `pointerWithin` - Only triggers when pointer is inside droppable

---

## 8. Key Implementation Files

| File | Purpose | Key Exports |
|------|---------|-------------|
| `frontend/src/pages/project-tasks.tsx` | Main page logic | `handleDragEnd`, kanban state |
| `frontend/src/components/tasks/TaskKanbanBoard.tsx` | Board wrapper | `TaskKanbanBoard` component |
| `frontend/src/components/ui/shadcn-io/kanban/index.tsx` | Core DnD logic | `KanbanProvider`, `KanbanBoard`, `KanbanCard` |
| `frontend/src/components/tasks/TaskCard.tsx` | Task card UI | `TaskCard` component |

---

## 9. Answers to Original Questions

### ✅ What DnD library is used?
**@dnd-kit v6.3.1** - Modern, well-maintained library with excellent React support

### ✅ How are drag events handled?
Via `handleDragEnd` callback in `project-tasks.tsx:730`, which:
1. Extracts `active` (dragged item) and `over` (drop target) from event
2. Validates the drag operation
3. Calls backend API to update task status
4. State updates automatically via SSE/polling

### ✅ What data is passed in drag events?
```typescript
{
  active: { id, data: { index, parent }, rect },
  over: { id, data, rect, disabled } | null,
  delta: { x, y },
  activatorEvent?: Event,
  collisions?: Collision[]
}
```

### ✅ Are there configurable event callbacks?
**Currently:** Only `onDragEnd` is used

**Available:** `onDragStart`, `onDragMove`, `onDragOver`, `onDragCancel`

**To Add:** Follow the steps in section 4.3

### ✅ Can we prevent default behavior?
**Yes, multiple ways:**
1. **Early return** from `onDragEnd` (current approach)
2. **Conditional `disabled` prop** on draggables
3. **Custom collision detection** to filter invalid targets
4. **Activation constraints** to prevent accidental drags

**Note:** @dnd-kit doesn't have "default behavior" that auto-moves items. The consumer must implement all state updates manually, giving full control.

---

## 10. Recommendations

### For Intercepting Drag Events

If you need to hook into drag events for logging, analytics, or custom logic:

**Option A: Add callbacks to existing components**
```typescript
<TaskKanbanBoard
  onDragEnd={handleDragEnd}
  onDragStart={(event) => {
    console.log('Started dragging:', event.active.id);
    posthog.capture('task_drag_started');
  }}
  onDragOver={(event) => {
    // Preview logic, highlight drop zones, etc.
  }}
/>
```

**Option B: Use useDndMonitor inside TaskCard or other child components**
```typescript
// Inside TaskCard.tsx
import { useDndMonitor } from '@dnd-kit/core';

export function TaskCard({ task, ... }) {
  useDndMonitor({
    onDragStart(event) {
      if (event.active.id === task.id) {
        console.log('This card started dragging');
      }
    }
  });

  // ... rest of component
}
```

### For Preventing Drops

```typescript
const handleDragEnd = async (event: DragEndEvent) => {
  const { active, over } = event;
  if (!over) return;

  // Custom validation
  const task = tasksById[active.id];
  const targetStatus = over.id as TaskStatus;

  if (!canMoveToStatus(task, targetStatus)) {
    // Show toast/notification
    toast.error('Cannot move task to this status');
    return; // Prevents API call, no state update
  }

  // Proceed with update
  await tasksApi.update(...);
};
```

### For Conditional Drag Enabling

Already implemented for shared tasks:

```typescript
// In TaskKanbanBoard.tsx:65-70
const isOwnTask =
  item.type === 'task' &&
  (!item.sharedTask?.assignee_user_id ||
    !userId ||
    item.sharedTask?.assignee_user_id === userId);

// Passed to TaskCard which uses:
dragDisabled={!isOwnTask}
```

---

## 11. Related GitHub Issues

Based on git log, there was a recent fix for drag-and-drop:
- **Commit:** `b6d876a` - "Can't drag task onto kanban below page fold (vibe-kanban) (#1179)"
- This likely addressed scrolling/viewport issues with the custom modifier

---

## Conclusion

The vibe-kanban drag-and-drop implementation uses **@dnd-kit**, a modern and flexible library that gives full control over drag behavior. The current implementation:

✅ Uses only `onDragEnd` for updates
✅ Has early-return patterns for preventing invalid drops
✅ Supports disabling drag via `dragDisabled` prop
✅ Has custom scroll constraints via modifiers
✅ Can be extended with additional callbacks (`onDragStart`, `onDragOver`, etc.)

The architecture is well-structured for adding custom event logic, monitoring, or validation without major refactoring.
