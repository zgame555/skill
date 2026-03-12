# Zustand Workflows Reference

## Contents

- Adding a New Store
- Adding Actions to an Existing Store
- Wiring Undo/Redo to UI
- Connecting Stores to IPC Actions
- Store Testing Patterns

---

## Adding a New Store

Copy this checklist and track progress:

- [ ] Step 1: Define state interface in the store file
- [ ] Step 2: Create the store with appropriate middleware
- [ ] Step 3: Export from `src/stores/index.ts` barrel
- [ ] Step 4: Wire into components with selectors

### Example: Creating the UI store

```typescript
// src/stores/ui-store.ts
import { create } from "zustand";

interface UiState {
  propertiesPanelVisible: boolean;
  sidebarVisible: boolean;
  treePanelVisible: boolean;
  togglePropertiesPanel: () => void;
  toggleSidebar: () => void;
  toggleTreePanel: () => void;
}

export const useUiStore = create<UiState>()((set) => ({
  propertiesPanelVisible: true,
  sidebarVisible: true,
  treePanelVisible: true,
  togglePropertiesPanel: () =>
    set((s) => ({ propertiesPanelVisible: !s.propertiesPanelVisible })),
  toggleSidebar: () => set((s) => ({ sidebarVisible: !s.sidebarVisible })),
  toggleTreePanel: () =>
    set((s) => ({ treePanelVisible: !s.treePanelVisible })),
}));
```

**Decision: When to use immer middleware:**

- Simple flat state (booleans, strings, single IDs) → no immer needed
- Nested objects/arrays (checklist items, file groups) → use immer

**Decision: When to add temporal (undo/redo):**

- Data the user edits and expects to undo → add temporal
- UI state (panel toggles, selection) → no temporal

---

## Adding Actions to an Existing Store

Copy this checklist and track progress:

- [ ] Step 1: Add action type to the state interface
- [ ] Step 2: Implement the action in the store creator
- [ ] Step 3: If using temporal, verify partialize still excludes the new action
- [ ] Step 4: Use the action in components via selector or direct call

### Example: Adding `renameChecklist` to checklist store

```typescript
// 1. Add to interface
interface ChecklistState {
  // ... existing fields
  renameChecklist: (fileId: string, checklistId: string, name: string) => void;
}

// 2. Implement with immer
renameChecklist: (fileId, checklistId, name) =>
  set((state) => {
    const file = state.files[fileId];
    for (const group of file.groups) {
      const checklist = group.checklists.find((c) => c.id === checklistId);
      if (checklist) {
        checklist.name = name;
        break;
      }
    }
  }),
```

### Validation: partialize check

After adding any action, confirm `partialize` returns only data fields:

```typescript
partialize: (state) => {
  const { files, activeFileId, activeChecklistId, activeItemId } = state;
  return { files, activeFileId, activeChecklistId, activeItemId };
  // Actions are automatically excluded — no changes needed
},
```

---

## Wiring Undo/Redo to UI

Copy this checklist and track progress:

- [ ] Step 1: Install zundo (`pnpm add zundo`)
- [ ] Step 2: Wrap store with `temporal` middleware
- [ ] Step 3: Configure `partialize` to track only data fields
- [ ] Step 4: Create undo/redo toolbar buttons with reactive disabled state
- [ ] Step 5: Wire Ctrl+Z / Ctrl+Shift+Z keyboard shortcuts

### Toolbar integration

```typescript
// src/components/editor/toolbar.tsx
import { useTemporalStore } from "zundo";
import { useChecklistStore } from "@/stores";

function UndoRedoGroup() {
  const { undo, redo } = useChecklistStore.temporal.getState();
  const canUndo = useTemporalStore(useChecklistStore)(
    (s) => s.pastStates.length > 0,
  );
  const canRedo = useTemporalStore(useChecklistStore)(
    (s) => s.futureStates.length > 0,
  );

  return (
    <div className="flex items-center gap-0.5">
      <ToolbarButton
        icon={Undo2}
        tooltip="Undo (Ctrl+Z)"
        disabled={!canUndo}
        onClick={() => undo()}
      />
      <ToolbarButton
        icon={Redo2}
        tooltip="Redo (Ctrl+Shift+Z)"
        disabled={!canRedo}
        onClick={() => redo()}
      />
    </div>
  );
}
```

### Keyboard shortcut integration

```typescript
// src/hooks/use-keyboard-shortcuts.ts
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    const mod = e.ctrlKey || e.metaKey;
    if (mod && e.key === "z" && !e.shiftKey) {
      e.preventDefault();
      useChecklistStore.temporal.getState().undo();
    }
    if (mod && e.key === "z" && e.shiftKey) {
      e.preventDefault();
      useChecklistStore.temporal.getState().redo();
    }
  };
  window.addEventListener("keydown", handler);
  return () => window.removeEventListener("keydown", handler);
}, []);
```

---

## Connecting Stores to IPC Actions

Zustand stores hold in-memory state. File I/O goes through IPC (main process). The pattern:

1. **Import flow:** IPC returns parsed data → store action adds it
2. **Export flow:** Read from store → pass to IPC for serialization + write
3. **Autosave flow:** Subscribe to store changes → debounce → write via IPC

### Import: IPC → Store

```typescript
// src/components/editor/files-sidebar.tsx
import { importFile } from "@/actions/checklist";
import { useChecklistStore } from "@/stores";

async function handleImport() {
  const file = await importFile(); // IPC call — returns ChecklistFile | null
  if (file) {
    useChecklistStore.getState().addFile(file);
  }
}
```

### Export: Store → IPC

```typescript
import { exportFile } from "@/actions/checklist";
import { useChecklistStore } from "@/stores";

async function handleExport(format: ChecklistFormat) {
  const { files, activeFileId } = useChecklistStore.getState();
  if (!activeFileId) return;
  const file = files[activeFileId];
  await exportFile(file, format); // IPC call — serializes + saves
}
```

### Autosave: Subscribe to store changes

```typescript
// src/hooks/use-autosave.ts
import { useEffect } from "react";
import { useChecklistStore } from "@/stores";
import { writeChecklistFile } from "@/actions/checklist";

export function useAutosave() {
  useEffect(() => {
    const unsub = useChecklistStore.subscribe(
      (state) => state.files,
      (files) => {
        // Debounced write — implementation omitted for brevity
        for (const file of Object.values(files)) {
          writeChecklistFile(file, file.filePath);
        }
      },
      { equalityFn: Object.is },
    );
    return unsub;
  }, []);
}
```

### WARNING: Never call IPC inside store actions

**The Problem:**

```typescript
// BAD — mixing side effects into store
addFile: async (path) =>
  set(async (state) => {
    const data = await readChecklistFile(path); // IPC call inside store!
    state.files[data.id] = data;
  }),
```

**Why This Breaks:**

1. Zustand `set()` is synchronous — async callbacks don't work as expected
2. Undo/redo captures intermediate states mid-fetch
3. Error handling is impossible inside the store updater

**The Fix:** Call IPC in the component/hook, then call the store action with the result:

```typescript
// In component
const data = await readChecklistFile(path);
useChecklistStore.getState().addFile(data);
```

---

## Store Testing Patterns

### Reset store between tests

```typescript
import { useChecklistStore } from "@/stores";

beforeEach(() => {
  useChecklistStore.setState({
    files: {},
    activeFileId: null,
    activeChecklistId: null,
    activeItemId: null,
  });
  useChecklistStore.temporal.getState().clear();
});
```

### Test an action

```typescript
it("adds a file and sets it active", () => {
  const file = createMockChecklistFile({ id: "f1", name: "Test" });
  useChecklistStore.getState().addFile(file);

  const state = useChecklistStore.getState();
  expect(state.files["f1"]).toEqual(file);
  expect(state.activeFileId).toBe("f1");
});
```

### Test undo/redo

```typescript
it("undoes addFile", () => {
  const file = createMockChecklistFile({ id: "f1" });
  useChecklistStore.getState().addFile(file);
  expect(useChecklistStore.getState().files["f1"]).toBeDefined();

  useChecklistStore.temporal.getState().undo();
  expect(useChecklistStore.getState().files["f1"]).toBeUndefined();

  useChecklistStore.temporal.getState().redo();
  expect(useChecklistStore.getState().files["f1"]).toBeDefined();
});
```

### Iterate-until-pass validation

1. Make store changes
2. Validate: `pnpm run lint && pnpm run build`
3. If build fails with type errors, fix the store interface and repeat step 2
4. Only proceed when build passes
