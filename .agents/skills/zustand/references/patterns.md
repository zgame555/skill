# Zustand Patterns Reference

## Contents

- Store Architecture
- Immer for Nested Checklist Data
- Undo/Redo with Zundo
- Selectors and Re-render Optimization
- Anti-Patterns

---

## Store Architecture

This project uses **two separate stores** to avoid unnecessary re-renders:

| Store               | Purpose                                       | Middleware           | File                            |
| ------------------- | --------------------------------------------- | -------------------- | ------------------------------- |
| `useChecklistStore` | Checklist files, groups, items, selection IDs | `temporal` + `immer` | `src/stores/checklist-store.ts` |
| `useUiStore`        | Panel visibility, sidebar collapsed           | None                 | `src/stores/ui-store.ts`        |

**Why two stores:** UI toggles (properties panel, sidebar) change frequently and shouldn't trigger re-renders in the editor item list. Checklist data changes need undo/redo tracking; UI state does not.

```typescript
// src/stores/index.ts — barrel export
export { useChecklistStore } from "./checklist-store";
export { useUiStore } from "./ui-store";
```

---

## Immer for Nested Checklist Data

Checklist data is deeply nested: `file → groups → checklists → items`. Without immer, every update requires spreading at every level.

```typescript
// GOOD — immer lets you mutate the draft directly
updateItem: (fileId, checklistId, itemId, updates) =>
  set((state) => {
    const file = state.files[fileId];
    const group = file.groups.find((g) =>
      g.checklists.some((c) => c.id === checklistId),
    );
    const checklist = group?.checklists.find((c) => c.id === checklistId);
    const item = checklist?.items.find((i) => i.id === itemId);
    if (item) {
      Object.assign(item, updates);
    }
  }),
```

```typescript
// BAD — manual spreading is error-prone and unreadable at this depth
updateItem: (fileId, checklistId, itemId, updates) =>
  set((state) => ({
    files: {
      ...state.files,
      [fileId]: {
        ...state.files[fileId],
        groups: state.files[fileId].groups.map((g) => ({
          ...g,
          checklists: g.checklists.map((c) =>
            c.id === checklistId
              ? { ...c, items: c.items.map((i) =>
                  i.id === itemId ? { ...i, ...updates } : i,
                )}
              : c,
          ),
        })),
      },
    },
  })),
```

### Reorder items with immer

```typescript
reorderItems: (checklistId, fromIndex, toIndex) =>
  set((state) => {
    const checklist = findChecklist(state, checklistId);
    if (!checklist) return;
    const [moved] = checklist.items.splice(fromIndex, 1);
    checklist.items.splice(toIndex, 0, moved);
  }),
```

---

## Undo/Redo with Zundo

### Middleware stacking order

The `temporal` middleware wraps `immer`. Order matters — `temporal` must be outermost to capture state snapshots after immer produces the next immutable state.

```typescript
create<ChecklistState>()(
  temporal(
    // outermost — captures snapshots
    immer((set) => ({
      // innermost — enables mutable syntax
      // ...
    })),
    { partialize: (s) => ({ files: s.files, activeFileId: s.activeFileId }) },
  ),
);
```

### Partialize — exclude actions from history

Zundo snapshots the **return value of `partialize`**. Always exclude action functions — they're static and waste memory.

```typescript
// GOOD — only track data fields
partialize: (state) => {
  const { files, activeFileId, activeChecklistId, activeItemId } = state;
  return { files, activeFileId, activeChecklistId, activeItemId };
},
```

```typescript
// BAD — omit approach leaks new actions into history when store grows
partialize: (state) => {
  const { addFile, removeFile, ...rest } = state;
  return rest; // breaks when you add new actions and forget to exclude them
},
```

### Accessing undo/redo imperatively

```typescript
// In a toolbar component or keyboard shortcut handler
const handleUndo = () => useChecklistStore.temporal.getState().undo();
const handleRedo = () => useChecklistStore.temporal.getState().redo();
```

### Reactive undo/redo state for button disabled

`pastStates` and `futureStates` from `temporal.getState()` are NOT reactive. To get reactive canUndo/canRedo, subscribe to the temporal store:

```typescript
import { useTemporalStore } from "zundo";

function UndoRedoButtons() {
  const canUndo = useTemporalStore(useChecklistStore)((s) => s.pastStates.length > 0);
  const canRedo = useTemporalStore(useChecklistStore)((s) => s.futureStates.length > 0);
  const { undo, redo } = useChecklistStore.temporal.getState();

  return (
    <>
      <button disabled={!canUndo} onClick={() => undo()}>Undo</button>
      <button disabled={!canRedo} onClick={() => redo()}>Redo</button>
    </>
  );
}
```

---

## Selectors and Re-render Optimization

### Single field selector (most common)

```typescript
// Component only re-renders when activeFileId changes
const activeFileId = useChecklistStore((s) => s.activeFileId);
```

### Multi-field selector with useShallow

```typescript
import { useShallow } from "zustand/shallow";

// Re-renders only when activeFileId OR activeChecklistId changes
const { activeFileId, activeChecklistId } = useChecklistStore(
  useShallow((s) => ({
    activeFileId: s.activeFileId,
    activeChecklistId: s.activeChecklistId,
  })),
);
```

### Derived selector — compute the active file

```typescript
// GOOD — derive at selection time, no extra state
const activeFile = useChecklistStore((s) =>
  s.activeFileId ? s.files[s.activeFileId] : null,
);
```

---

## Anti-Patterns

### WARNING: Storing derived state

**The Problem:**

```typescript
// BAD — activeFile is derivable from activeFileId + files
interface State {
  files: Record<string, ChecklistFile>;
  activeFileId: string | null;
  activeFile: ChecklistFile | null; // REDUNDANT
}
```

**Why This Breaks:**

1. Two sources of truth — `activeFile` can desync from `files[activeFileId]`
2. Every mutation to a file must also update `activeFile`
3. Undo/redo restores `activeFile` snapshot that may not match restored `files`

**The Fix:** Derive in selectors or during render.

### WARNING: Selecting the entire store

**The Problem:**

```typescript
// BAD — re-renders on ANY state change
const state = useChecklistStore();
```

**Why This Breaks:** Every `set()` call triggers a re-render in this component, even for unrelated fields. In a checklist editor with frequent item edits, this causes severe performance issues.

**The Fix:** Always use a selector:

```typescript
const activeFileId = useChecklistStore((s) => s.activeFileId);
```

### WARNING: Calling set() in a loop

**The Problem:**

```typescript
// BAD — N separate state updates, N undo history entries
items.forEach((item) => {
  set((state) => {
    state.files[fid].items.push(item);
  });
});
```

**Why This Breaks:** Each `set()` is a separate state snapshot. Undo would require N undos to reverse a single user action.

**The Fix:** Batch in a single `set()`:

```typescript
set((state) => {
  items.forEach((item) => {
    state.files[fid].items.push(item);
  });
});
```
