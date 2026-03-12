# dnd-kit Patterns

## Contents

- Drag Handle Pattern
- Zustand Store Integration
- Multiple Sortable Contexts
- Drag Overlay for Visual Feedback
- Activation Constraints
- Anti-Patterns

## Drag Handle Pattern

The editor uses a dedicated drag handle (6-dot grip icon) instead of making entire rows draggable. This prevents accidental drags when clicking to select or edit items.

```tsx
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { GripVertical } from "lucide-react";
import { cn } from "@/utils/cn";

interface SortableRowProps {
  id: string;
  children: React.ReactNode;
}

function SortableRow({ id, children }: SortableRowProps) {
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
        "group flex items-center gap-2 transition-colors duration-150",
        isDragging && "z-50 opacity-70 shadow-lg",
      )}
      style={{
        transform: CSS.Transform.toString(transform),
        transition,
      }}
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

Key details:

- `setNodeRef` goes on the entire row (the sortable container)
- `setActivatorNodeRef` goes on the drag handle element
- `listeners` and `attributes` go on the handle, NOT the container
- Use `opacity-0 group-hover:opacity-100` for the handle visibility per UI spec

## Zustand Store Integration

`onDragEnd` should dispatch to the Zustand store, not manage local state. The store's `reorderItems` action uses `arrayMove` and records the change for undo/redo. See the **zustand** skill for store patterns.

```tsx
import type { DragEndEvent } from "@dnd-kit/core";
import { arrayMove } from "@dnd-kit/sortable";
import { useChecklistStore } from "@/stores/checklist-store";

function handleDragEnd(event: DragEndEvent) {
  const { active, over } = event;
  if (!over || active.id === over.id) return;

  const items = useChecklistStore.getState().activeChecklist?.items;
  if (!items) return;

  const oldIndex = items.findIndex((i) => i.id === active.id);
  const newIndex = items.findIndex((i) => i.id === over.id);

  if (oldIndex === -1 || newIndex === -1) return;

  useChecklistStore
    .getState()
    .reorderItems(arrayMove(items, oldIndex, newIndex));
}
```

NEVER call `setItems` with local `useState` for data that belongs in the store. The store is the single source of truth.

## Multiple Sortable Contexts

The app has two independent sortable surfaces:

1. **Editor panel** - checklist items (vertical list)
2. **Tree panel** - checklists within groups (vertical list)

Each gets its own `DndContext`. NEVER nest `DndContext` providers.

```tsx
// checklist-editor.tsx - items are sortable
function ChecklistEditor() {
  return (
    <DndContext sensors={sensors} onDragEnd={handleItemDragEnd}>
      <SortableContext items={itemIds} strategy={verticalListSortingStrategy}>
        {/* item rows */}
      </SortableContext>
    </DndContext>
  );
}

// checklist-tree.tsx - checklists are sortable within groups
function ChecklistTree() {
  return (
    <DndContext sensors={sensors} onDragEnd={handleChecklistDragEnd}>
      {groups.map((group) => (
        <SortableContext
          key={group.id}
          items={group.checklists.map((c) => c.id)}
          strategy={verticalListSortingStrategy}
        >
          {/* checklist rows */}
        </SortableContext>
      ))}
    </DndContext>
  );
}
```

Multiple `SortableContext` components under one `DndContext` is fine for groups within the tree panel.

## Drag Overlay for Visual Feedback

Use `DragOverlay` to render a floating copy of the dragged item. Without it, the original item just becomes semi-transparent.

```tsx
import { DndContext, DragOverlay } from "@dnd-kit/core";
import { useState } from "react";
import type { DragStartEvent } from "@dnd-kit/core";

function SortableList() {
  const [activeId, setActiveId] = useState<string | null>(null);
  const activeItem = items.find((i) => i.id === activeId);

  return (
    <DndContext
      onDragStart={(event: DragStartEvent) =>
        setActiveId(String(event.active.id))
      }
      onDragEnd={(event) => {
        setActiveId(null);
        handleDragEnd(event);
      }}
      onDragCancel={() => setActiveId(null)}
    >
      <SortableContext items={itemIds} strategy={verticalListSortingStrategy}>
        {items.map((item) => (
          <SortableItemRow key={item.id} item={item} />
        ))}
      </SortableContext>
      <DragOverlay>
        {activeItem ? (
          <div className="bg-elevated border-border rounded border opacity-90 shadow-lg">
            <ItemRowContent item={activeItem} />
          </div>
        ) : null}
      </DragOverlay>
    </DndContext>
  );
}
```

## Activation Constraints

Always set an activation constraint on `PointerSensor` to prevent accidental drags when clicking.

```tsx
const sensors = useSensors(
  useSensor(PointerSensor, {
    activationConstraint: { distance: 5 }, // 5px movement required to start drag
  }),
  useSensor(KeyboardSensor, {
    coordinateGetter: sortableKeyboardCoordinates,
  }),
);
```

Without the `distance: 5` constraint, any click on the drag handle triggers a drag operation, making it impossible to click items normally.

## Anti-Patterns

### WARNING: Inline Styles for Visual State

**The Problem:**

```tsx
// BAD - inline styles for visual feedback
<div style={{ opacity: isDragging ? 0.5 : 1, backgroundColor: isDragging ? "#1c2333" : "transparent" }}>
```

**Why This Breaks:** Violates the project's strict "no inline styles" rule. Hardcoded hex values bypass theme tokens and won't adapt if the theme changes.

**The Fix:**

```tsx
// GOOD - Tailwind classes with cn()
<div className={cn("transition-opacity", isDragging && "opacity-50 bg-elevated shadow-lg")}>
```

The ONLY acceptable inline `style` prop is `transform` and `transition` from `useSortable`, which dnd-kit requires for smooth animations.

### WARNING: Listeners on the Container

**The Problem:**

```tsx
// BAD - entire row is draggable
<div ref={setNodeRef} {...attributes} {...listeners}>
```

**Why This Breaks:** Users can't click items to select them or double-click to edit. Every mouse interaction starts a drag. The UI spec requires a dedicated drag handle.

**The Fix:**

```tsx
// GOOD - only handle triggers drag
<div ref={setNodeRef}>
  <button ref={setActivatorNodeRef} {...attributes} {...listeners}>
    <GripVertical />
  </button>
  {/* clickable content */}
</div>
```

### WARNING: Missing Keyboard Sensor

**The Problem:**

```tsx
// BAD - pointer only
const sensors = useSensors(useSensor(PointerSensor));
```

**Why This Breaks:** Keyboard users cannot reorder items. dnd-kit is built for accessibility; omitting the keyboard sensor defeats its purpose.

**The Fix:**

```tsx
// GOOD - always include both sensors
const sensors = useSensors(
  useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
  useSensor(KeyboardSensor, { coordinateGetter: sortableKeyboardCoordinates }),
);
```
