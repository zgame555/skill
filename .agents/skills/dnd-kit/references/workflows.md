# dnd-kit Workflows

## Contents

- Installation and Setup
- Adding Sortable to a New List
- Connecting to Zustand Store
- Testing Drag and Drop
- Debugging Common Issues

## Installation and Setup

### Install Dependencies

```bash
pnpm add @dnd-kit/core @dnd-kit/sortable @dnd-kit/utilities
```

No `@types/` packages needed — dnd-kit ships TypeScript types.

### Setup Checklist

Copy this checklist and track progress:

- [ ] Install `@dnd-kit/core`, `@dnd-kit/sortable`, `@dnd-kit/utilities`
- [ ] Create sensors configuration (shared across components)
- [ ] Add `DndContext` wrapper to editor panel
- [ ] Add `SortableContext` with item IDs
- [ ] Implement `useSortable` in item row component
- [ ] Add drag handle with `setActivatorNodeRef`
- [ ] Wire `onDragEnd` to Zustand store action
- [ ] Add `DragOverlay` for visual feedback
- [ ] Test keyboard reordering (Space to pick up, arrows to move, Space to drop)
- [ ] Repeat for tree panel (checklist reordering)

## Adding Sortable to a New List

### Step 1: Create a Shared Sensors Hook

Create a reusable sensors configuration to avoid duplication between editor and tree panels.

```tsx
// src/hooks/use-sortable-sensors.ts
import {
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
} from "@dnd-kit/core";
import { sortableKeyboardCoordinates } from "@dnd-kit/sortable";

export function useSortableSensors() {
  return useSensors(
    useSensor(PointerSensor, {
      activationConstraint: { distance: 5 },
    }),
    useSensor(KeyboardSensor, {
      coordinateGetter: sortableKeyboardCoordinates,
    }),
  );
}
```

### Step 2: Wrap the List

```tsx
import { DndContext, closestCenter } from "@dnd-kit/core";
import type { DragEndEvent } from "@dnd-kit/core";
import {
  SortableContext,
  verticalListSortingStrategy,
} from "@dnd-kit/sortable";
import { useSortableSensors } from "@/hooks/use-sortable-sensors";

function MyList({ items, onReorder }: Props) {
  const sensors = useSortableSensors();

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event;
    if (!over || active.id === over.id) return;
    // call store action
  }

  return (
    <DndContext
      sensors={sensors}
      collisionDetection={closestCenter}
      onDragEnd={handleDragEnd}
    >
      <SortableContext
        items={items.map((i) => i.id)}
        strategy={verticalListSortingStrategy}
      >
        {items.map((item) => (
          <SortableRow key={item.id} id={item.id}>
            {/* content */}
          </SortableRow>
        ))}
      </SortableContext>
    </DndContext>
  );
}
```

### Step 3: Make Each Row Sortable

```tsx
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { GripVertical } from "lucide-react";
import { cn } from "@/utils/cn";

function SortableRow({
  id,
  children,
}: {
  id: string;
  children: React.ReactNode;
}) {
  const {
    attributes,
    listeners,
    setNodeRef,
    setActivatorNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id });

  return (
    <div
      ref={setNodeRef}
      className={cn(
        "group flex items-center gap-2",
        isDragging && "opacity-70 shadow-lg",
      )}
      style={{ transform: CSS.Transform.toString(transform), transition }}
    >
      <div
        ref={setActivatorNodeRef}
        className="flex w-6 cursor-grab items-center justify-center opacity-0 transition-opacity duration-150 group-hover:opacity-100 active:cursor-grabbing"
        {...attributes}
        {...listeners}
      >
        <GripVertical className="text-muted-foreground h-4 w-4" />
      </div>
      {children}
    </div>
  );
}
```

## Connecting to Zustand Store

The Zustand store action for reordering should use `arrayMove` and integrate with the undo/redo middleware. See the **zustand** skill for temporal middleware.

```tsx
// In the store definition
import { arrayMove } from "@dnd-kit/sortable";

// Store action
reorderItems: (checklistId: string, oldIndex: number, newIndex: number) => {
  set((state) => {
    const checklist = findChecklist(state, checklistId);
    if (!checklist) return state;
    checklist.items = arrayMove(checklist.items, oldIndex, newIndex);
    return { ...state };
  });
},
```

```tsx
// In the component's onDragEnd handler
function handleDragEnd(event: DragEndEvent) {
  const { active, over } = event;
  if (!over || active.id === over.id) return;

  const items = useChecklistStore.getState().activeChecklist?.items ?? [];
  const oldIndex = items.findIndex((i) => i.id === active.id);
  const newIndex = items.findIndex((i) => i.id === over.id);
  if (oldIndex === -1 || newIndex === -1) return;

  const checklistId = useChecklistStore.getState().activeChecklistId;
  if (checklistId) {
    useChecklistStore.getState().reorderItems(checklistId, oldIndex, newIndex);
  }
}
```

### Validation Loop

After implementing reorder:

1. Drag an item to a new position
2. Verify the Zustand store updated correctly
3. Undo (Ctrl+Z) and verify the item returns to original position
4. If undo doesn't work, check that the store action is tracked by temporal middleware
5. Repeat until undo/redo works correctly for all reorder operations

## Testing Drag and Drop

### Manual Test Checklist

Copy and use for QA:

- [ ] **Pointer drag**: Click and drag handle, item follows cursor
- [ ] **Drop indicator**: Visual line appears between items showing drop position
- [ ] **Reorder persists**: After dropping, item stays in new position
- [ ] **Keyboard drag**: Focus handle, press Space, use arrows, press Space to drop
- [ ] **Cancel drag**: Press Escape during drag, item returns to original position
- [ ] **Undo after drag**: Ctrl+Z restores previous order
- [ ] **Click still works**: Clicking an item (not handle) selects it, doesn't drag
- [ ] **Double-click still works**: Double-clicking enters edit mode
- [ ] **Scroll during drag**: Dragging near container edges scrolls the list
- [ ] **Multiple items**: Reorder works with 1, 2, 5, 50+ items

## Debugging Common Issues

### Items Don't Move

**Check:** Are the `items` prop of `SortableContext` matching the actual item IDs?

```tsx
// BAD - passing full objects
<SortableContext items={items}>

// GOOD - passing IDs only
<SortableContext items={items.map((i) => i.id)}>
```

The `items` array must contain the same IDs used in `useSortable({ id })`.

### Drag Starts but Item Snaps Back

**Check:** Is `onDragEnd` actually updating the state?

```tsx
// Debug: log the event
function handleDragEnd(event: DragEndEvent) {
  console.log("dragEnd", event.active.id, "→", event.over?.id);
  // ... rest of handler
}
```

Common causes:

- `over` is `null` (dropped outside any sortable)
- `findIndex` returns -1 (ID mismatch between store and sortable)
- Store action doesn't trigger re-render (missing `set()` call in Zustand)

### Transform Looks Wrong

**Check:** Are you using `CSS.Transform.toString()` from `@dnd-kit/utilities`?

```tsx
// BAD - manual transform
style={{ transform: `translate(${transform?.x}px, ${transform?.y}px)` }}

// GOOD - utility handles null safety and format
style={{ transform: CSS.Transform.toString(transform), transition }}
```

### Drag Handle Not Working

**Check:** Are `listeners` attached to the handle element, not the container?

```tsx
// BAD - listeners on container, handle is decorative
<div ref={setNodeRef} {...listeners}>
  <GripVertical />  {/* This does nothing */}
</div>

// GOOD - listeners on handle
<div ref={setNodeRef}>
  <div ref={setActivatorNodeRef} {...listeners}>
    <GripVertical />
  </div>
</div>
```

### Flickering During Drag

**Check:** Is the component re-rendering excessively? Memoize the sortable item.

```tsx
import { memo } from "react";

const SortableItemRow = memo(function SortableItemRow({ item }: Props) {
  const { setNodeRef, transform, transition, isDragging } = useSortable({
    id: item.id,
  });
  // ...
});
```

Use React DevTools Profiler to confirm unnecessary re-renders. See the **react** skill for memoization patterns.
