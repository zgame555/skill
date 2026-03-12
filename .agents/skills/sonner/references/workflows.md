# Sonner Workflows Reference

## Contents

- Adding Toast Feedback to a New Feature
- File Operation Workflow
- Promise Toast for Async Operations
- Undo with Toast Actions
- Toast Testing Workflow

---

## Adding Toast Feedback to a New Feature

Copy this checklist and track progress:

- [ ] Step 1: Import `toast` from `"sonner"` in the component or action file
- [ ] Step 2: Identify success and error paths
- [ ] Step 3: Add `toast.success()` on success with description
- [ ] Step 4: Add `toast.error()` in catch with error message
- [ ] Step 5: Verify toast appears correctly in dev mode (`pnpm run dev`)

```typescript
// Step 1: Import
import { toast } from "sonner";

// Steps 2-4: Add feedback to both paths
async function handleSomeAction() {
  try {
    const result = await someAsyncOperation();
    toast.success("Action completed", {
      description: `Processed ${result.count} items`,
    });
  } catch (err) {
    toast.error("Action failed", {
      description: err instanceof Error ? err.message : "Unknown error",
    });
  }
}
```

**Validation loop:**

1. Trigger the action in the app
2. Verify toast appears with correct type (success/error)
3. Verify description text is helpful and specific
4. If toast doesn't appear, check that `<Toaster />` is mounted in `src/App.tsx`
5. Repeat for both success and error paths

---

## File Operation Workflow

All file operations go through IPC (see the **orpc** skill). Each IPC call should show toast feedback.

### Import File Flow

```typescript
import { toast } from "sonner";
import { ipc } from "@/ipc/manager";

async function handleImport() {
  try {
    // IPC opens native file dialog + parses file
    const file = await ipc.client.checklist.importFile();

    if (!file) return; // User cancelled — no toast needed

    toast.success(`Imported "${file.name}"`, {
      description: `${file.groups.length} groups, ${file.format} format`,
    });

    return file;
  } catch (err) {
    toast.error("Import failed", {
      description:
        err instanceof Error ? err.message : "Unsupported file format",
    });
    return null;
  }
}
```

### Export File Flow

```typescript
async function handleExport(format: ChecklistFormat) {
  const path = await ipc.client.dialog.saveFileDialog({
    defaultName: activeFile.name,
    format,
  });

  if (!path) return; // User cancelled

  toast.promise(
    ipc.client.checklist.exportFile({ file: activeFile, format, path }),
    {
      loading: `Exporting as ${format}...`,
      success: `Saved to ${path}`,
      error: (err) => `Export failed: ${err.message}`,
    },
  );
}
```

### Quick Export Flow

Quick export reuses last format/path — use a simpler toast:

```typescript
async function handleQuickExport() {
  if (!lastExportConfig) {
    toast.info("No previous export", {
      description: "Use Export to choose a format first.",
    });
    return;
  }

  toast.promise(
    ipc.client.checklist.exportFile({
      file: activeFile,
      format: lastExportConfig.format,
      path: lastExportConfig.path,
    }),
    {
      loading: "Exporting...",
      success: "Quick export complete",
      error: (err) => `Export failed: ${err.message}`,
    },
  );
}
```

---

## Promise Toast for Async Operations

Use `toast.promise()` for operations with clear loading → success/error states. This is the preferred pattern for IPC calls.

### Basic Pattern

```typescript
// The promise resolves/rejects — sonner handles the state transitions
toast.promise(asyncOperation(), {
  loading: "Working...",
  success: "Done!",
  error: "Failed",
});
```

### Dynamic Messages from Result

```typescript
// Success and error can be functions that receive the result/error
toast.promise(ipc.client.checklist.importFile(), {
  loading: "Parsing file...",
  success: (file) => `Loaded "${file.name}" with ${file.groups.length} groups`,
  error: (err) => `Parse error: ${err.message}`,
});
```

### WARNING: Don't Wrap Non-Rejecting Promises

```typescript
// BAD — This promise never rejects, error toast never shows
toast.promise(
  fetch("/api/data").then((r) => r.json()), // fetch doesn't reject on 404
  { loading: "...", success: "OK", error: "Failed" },
);

// GOOD — Ensure the promise actually rejects on failure
toast.promise(
  (async () => {
    const res = await fetch("/api/data");
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return res.json();
  })(),
  { loading: "...", success: "OK", error: (err) => err.message },
);
```

---

## Undo with Toast Actions

For destructive operations (delete item, delete checklist), show an undo action in the toast. See the **zustand** skill for state snapshots.

### Pattern: Optimistic Delete with Undo

```typescript
function handleDeleteItem(itemId: string) {
  // Capture state before deletion for undo
  const snapshot = useChecklistStore.getState();
  const item = snapshot.items.find((i) => i.id === itemId);

  // Perform deletion immediately (optimistic)
  useChecklistStore.getState().removeItem(itemId);

  toast(`Deleted "${item?.challengeText ?? "item"}"`, {
    action: {
      label: "Undo",
      onClick: () => {
        // Restore previous state
        useChecklistStore.setState(snapshot);
        toast.success("Restored");
      },
    },
    duration: 5000, // Give user time to undo
  });
}
```

### Pattern: Batch Delete with Undo

```typescript
function handleDeleteChecklist(checklistId: string, name: string) {
  const snapshot = useChecklistStore.getState();
  useChecklistStore.getState().removeChecklist(checklistId);

  toast(`Deleted checklist "${name}"`, {
    description: `${snapshot.activeChecklist?.items.length ?? 0} items removed`,
    action: {
      label: "Undo",
      onClick: () => {
        useChecklistStore.setState(snapshot);
        toast.success(`Restored "${name}"`);
      },
    },
    duration: 8000, // Longer for bigger deletions
  });
}
```

---

## Toast Testing Workflow

The codebase already has a test toast in `src/routes/index.tsx`. Use this pattern when developing new toast types.

### Manual Testing Steps

1. Run `pnpm run dev`
2. Trigger the operation that should show a toast
3. Verify:
   - Correct toast type (success/error/warning/info)
   - Icon matches type (FontAwesome icons configured in wrapper)
   - Description provides useful context
   - Toast auto-dismisses after expected duration
   - Toast respects dark theme colors
4. Test error path by simulating failure (disconnect network, use invalid file)

### Quick Toast Verification

Drop this temporarily in any component to verify toasts work:

```typescript
import { toast } from "sonner";

// In a useEffect or button handler:
toast.success("Success toast", { description: "With description" });
toast.error("Error toast", { description: "Something failed" });
toast.warning("Warning toast", { description: "Be careful" });
toast.info("Info toast", { description: "FYI" });
toast.loading("Loading toast");
```

### Debugging Missing Toasts

If toasts don't appear:

1. Confirm `<Toaster />` is in `src/App.tsx` (not removed accidentally)
2. Check browser console for React errors
3. Verify `import { toast } from "sonner"` — not from another package
4. Ensure the component is mounted inside the React tree (not in a portal outside `<App />`)
