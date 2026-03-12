---
name: zustand
description: |
  Manages Zustand state stores with immer middleware, zundo undo/redo, slices pattern, and computed selectors for the EFIS Checklist Editor.
  Use when: creating or modifying Zustand stores, adding store actions, wiring undo/redo,
  splitting stores into slices, optimizing selectors, or connecting stores to React components.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, mcp__context7__resolve-library-id, mcp__context7__query-docs
---

# Zustand Skill

Zustand manages all client-side state in the EFIS Checklist Editor. The project uses three stores: `checklist-store` (data + CRUD), `ui-store` (panel visibility), and undo/redo via `zundo` temporal middleware. All stores live in `src/stores/` with a barrel export. Immer middleware enables mutable update syntax for deeply nested checklist structures.

## Quick Start

### Create a typed store

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

### Store with immer + zundo (undo/redo)

```typescript
// src/stores/checklist-store.ts
import { create } from "zustand";
import { immer } from "zustand/middleware/immer";
import { temporal } from "zundo";
import type { ChecklistFile } from "@/types/checklist";

interface ChecklistState {
  files: Record<string, ChecklistFile>;
  activeFileId: string | null;
  addFile: (file: ChecklistFile) => void;
  removeFile: (id: string) => void;
}

export const useChecklistStore = create<ChecklistState>()(
  temporal(
    immer((set) => ({
      files: {},
      activeFileId: null,
      addFile: (file) =>
        set((state) => {
          state.files[file.id] = file;
          state.activeFileId = file.id;
        }),
      removeFile: (id) =>
        set((state) => {
          delete state.files[id];
          if (state.activeFileId === id) {
            state.activeFileId = null;
          }
        }),
    })),
    {
      partialize: (state) => {
        const { files, activeFileId } = state;
        return { files, activeFileId };
      },
    },
  ),
);
```

### Use undo/redo in components

```typescript
const { undo, redo, pastStates, futureStates } =
  useChecklistStore.temporal.getState();
const canUndo = pastStates.length > 0;
const canRedo = futureStates.length > 0;
```

## Key Concepts

| Concept          | Usage                             | Example                                 |
| ---------------- | --------------------------------- | --------------------------------------- |
| Immer middleware | Mutable syntax for nested updates | `set((s) => { s.files[id].name = n })`  |
| Zundo temporal   | Undo/redo on data stores          | `temporal(immer(...), { partialize })`  |
| Partialize       | Exclude actions from undo history | Return only state fields, not functions |
| Selectors        | Prevent unnecessary re-renders    | `useStore((s) => s.activeFileId)`       |
| Separate stores  | UI state vs data state            | `useUiStore` vs `useChecklistStore`     |

## Common Patterns

### Selector with shallow equality

**When:** Selecting multiple fields from a store.

```typescript
import { useShallow } from "zustand/shallow";

const { activeFileId, files } = useChecklistStore(
  useShallow((s) => ({ activeFileId: s.activeFileId, files: s.files })),
);
```

### Derived/computed values

**When:** Deriving active file/checklist/item from IDs in the store.

```typescript
// Compute during render — do NOT store derived state
function EditorHeader() {
  const activeFile = useChecklistStore((s) =>
    s.activeFileId ? s.files[s.activeFileId] : null,
  );
  // ...
}
```

## Dependencies

```bash
pnpm add zustand immer zundo
```

## See Also

- [patterns](references/patterns.md) — Store structure, slices, anti-patterns
- [workflows](references/workflows.md) — Adding stores, wiring undo/redo, testing

## Related Skills

- See the **react** skill for component integration and hooks
- See the **typescript** skill for type definitions used in store state
- See the **zod** skill for validating data flowing into stores from IPC

## Documentation Resources

> Fetch latest Zustand documentation with Context7.

**How to use Context7:**

1. Use `mcp__context7__resolve-library-id` to search for "zustand"
2. **Prefer website documentation** (IDs starting with `/websites/`) over source code repositories
3. Query with `mcp__context7__query-docs` using the resolved library ID

**Library IDs:**

- Zustand docs: `/websites/zustand_pmnd_rs`
- Zustand source: `/pmndrs/zustand`
- Zundo (undo/redo): `/charkour/zundo`

**Recommended Queries:**

- "zustand store creation TypeScript middleware"
- "zustand immer middleware nested state updates"
- "zundo temporal middleware undo redo partialize"
- "zustand slices pattern compose stores"
