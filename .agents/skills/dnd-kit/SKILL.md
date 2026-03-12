---
name: dnd-kit
description: |
  Implements drag-and-drop reordering for checklist items and checklists using @dnd-kit.
  Use when: adding sortable lists, reordering items via drag handle, building drag-and-drop
  interactions for the checklist editor or tree panel, or debugging dnd-kit integration issues.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, mcp__context7__resolve-library-id, mcp__context7__query-docs
---

# dnd-kit

@dnd-kit provides drag-and-drop reordering for two surfaces in this app: checklist items in the editor panel and checklists/groups in the tree panel. Both use the `@dnd-kit/sortable` preset with dedicated drag handles (6-dot grip icons) rather than making entire rows draggable.

## Quick Start

### Sortable List with Drag Handle

```tsx
import {
  DndContext,
  closestCenter,
  KeyboardSensor,
  PointerSensor,
  useSensor,
  useSensors,
} from "@dnd-kit/core";
import type { DragEndEvent } from "@dnd-kit/core";
import {
  arrayMove,
  SortableContext,
  sortableKeyboardCoordinates,
  verticalListSortingStrategy,
} from "@dnd-kit/sortable";
import { useChecklistStore } from "@/stores/checklist-store";

function ChecklistItemList() {
  const items = useChecklistStore((s) => s.activeChecklist?.items ?? []);
  const reorderItems = useChecklistStore((s) => s.reorderItems);

  const sensors = useSensors(
    useSensor(PointerSensor, { activationConstraint: { distance: 5 } }),
    useSensor(KeyboardSensor, {
      coordinateGetter: sortableKeyboardCoordinates,
    }),
  );

  function handleDragEnd(event: DragEndEvent) {
    const { active, over } = event;
    if (!over || active.id === over.id) return;

    const oldIndex = items.findIndex((item) => item.id === active.id);
    const newIndex = items.findIndex((item) => item.id === over.id);
    reorderItems(arrayMove(items, oldIndex, newIndex));
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
          <SortableItemRow key={item.id} item={item} />
        ))}
      </SortableContext>
    </DndContext>
  );
}
```

### Sortable Item with Drag Handle

```tsx
import { useSortable } from "@dnd-kit/sortable";
import { CSS } from "@dnd-kit/utilities";
import { GripVertical } from "lucide-react";
import { cn } from "@/utils/cn";

function SortableItemRow({ item }: { item: ChecklistItem }) {
  const {
    attributes,
    listeners,
    setNodeRef,
    setActivatorNodeRef,
    transform,
    transition,
    isDragging,
  } = useSortable({ id: item.id });

  return (
    <div
      ref={setNodeRef}
      className={cn(
        "group flex items-center gap-2 px-2 py-1",
        isDragging && "opacity-50 shadow-lg",
      )}
      style={{
        transform: CSS.Transform.toString(transform),
        transition,
      }}
    >
      <button
        ref={setActivatorNodeRef}
        className="cursor-grab opacity-0 group-hover:opacity-100 active:cursor-grabbing"
        {...attributes}
        {...listeners}
      >
        <GripVertical className="text-muted-foreground h-4 w-4" />
      </button>
      {/* item content */}
    </div>
  );
}
```

## Key Concepts

| Concept                       | Usage                      | Example                                 |
| ----------------------------- | -------------------------- | --------------------------------------- |
| `DndContext`                  | Wraps entire sortable area | One per sortable list                   |
| `SortableContext`             | Defines sortable item IDs  | `items={ids}` array of string/number    |
| `useSortable`                 | Hook on each sortable item | Returns `ref`, `listeners`, `transform` |
| `setActivatorNodeRef`         | Separate drag handle       | Attach to grip icon, not the whole row  |
| `arrayMove`                   | Reorder utility            | `arrayMove(items, oldIndex, newIndex)`  |
| Sensors                       | Input methods              | `PointerSensor` + `KeyboardSensor`      |
| `verticalListSortingStrategy` | Vertical list optimization | Always use for vertical lists           |

## Integration Points

- **Zustand store**: `onDragEnd` calls store actions like `reorderItems` or `reorderChecklists`. See the **zustand** skill.
- **Tailwind styling**: Use `isDragging` from `useSortable` to apply drag styles via `cn()`. NEVER use inline styles for colors/opacity. The `transform` and `transition` style props are the only acceptable inline styles.
- **Accessibility**: Always include `KeyboardSensor` with `sortableKeyboardCoordinates` for keyboard reordering.

## See Also

- [patterns](references/patterns.md) - Drag handle, multiple contexts, Zustand integration
- [workflows](references/workflows.md) - Setup checklist, testing, debugging

## Related Skills

- See the **react** skill for component patterns and hooks
- See the **zustand** skill for store actions that handle reorder state
- See the **tailwind** skill for drag state styling (opacity, shadow, cursor)
- See the **lucide-react** skill for the `GripVertical` drag handle icon

## Documentation Resources

> Fetch latest dnd-kit documentation with Context7.

**How to use Context7:**

1. Use `mcp__context7__resolve-library-id` to search for "dnd-kit"
2. Prefer website documentation (IDs starting with `/websites/`) over source code
3. Query with `mcp__context7__query-docs` using the resolved library ID

**Library ID:** `/websites/next_dndkit`

**Recommended Queries:**

- "sortable list useSortable hook drag handle"
- "sensors keyboard accessibility collision detection"
- "drag overlay multiple sortable contexts"
