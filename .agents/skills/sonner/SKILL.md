---
name: sonner
description: |
  Displays toast notifications for user feedback in the EFIS Checklist Editor.
  Use when: showing success/error/warning/info feedback for file operations (import, export, save),
  validation errors, format compatibility warnings, or any user-facing status messages.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, mcp__context7__resolve-library-id, mcp__context7__query-docs
---

# Sonner Skill

Sonner v2 is the toast notification system for this Electron + React app. It's wrapped by shadcn/ui in `src/components/ui/sonner.tsx` with custom FontAwesome icons and theme-aware styling via `next-themes`. The `<Toaster />` is mounted once in `src/App.tsx` — never mount it again.

## Quick Start

### Basic Toast Types

```typescript
import { toast } from "sonner";

// Success — file saved, export complete
toast.success("Checklist exported", {
  description: "Saved as N172SP_Checklists.ace",
});

// Error — import failed, parse error
toast.error("Import failed", {
  description: "File is not a valid Garmin ACE format.",
});

// Warning — format compatibility issue
toast.warning("Compatibility warning", {
  description: "Caution items are not supported in GRT format.",
});

// Info — general status
toast.info("Autosave enabled", {
  description: "Changes saved every 2 seconds.",
});

// Plain — no icon
toast("Checklist duplicated");
```

### Async Operations with Promise Toast

```typescript
toast.promise(exportFile(file, format, path), {
  loading: "Exporting checklist...",
  success: "Export complete!",
  error: (err) => `Export failed: ${err.message}`,
});
```

### Toast with Action Button

```typescript
toast("Item deleted", {
  action: {
    label: "Undo",
    onClick: () => undoLastAction(),
  },
});
```

## Key Concepts

| Concept       | Usage                              | Example                                       |
| ------------- | ---------------------------------- | --------------------------------------------- |
| Import        | Always from `"sonner"` directly    | `import { toast } from "sonner"`              |
| Toaster mount | Once in `src/App.tsx`, never again | `<Toaster />`                                 |
| Theme sync    | Automatic via `next-themes`        | Reads `useTheme()` in wrapper                 |
| Duration      | Default 4000ms, override per toast | `toast("msg", { duration: 6000 })`            |
| Dismiss       | `toast.dismiss(id)` or auto        | `const id = toast("msg"); toast.dismiss(id);` |
| Custom icons  | Set globally in wrapper component  | FontAwesome icons in `sonner.tsx`             |

## Common Patterns

### IPC Operation Feedback

**When:** Any file I/O operation via oRPC (import, export, save).

```typescript
async function handleExport(format: ChecklistFormat) {
  const path = await saveFileDialog(file.name, format);
  if (!path) return;

  toast.promise(exportFile(activeFile, format, path), {
    loading: "Exporting...",
    success: `Exported to ${path}`,
    error: (err) => `Export failed: ${err.message}`,
  });
}
```

### Dismissible Status Updates

**When:** Long-running operations or persistent status.

```typescript
const toastId = toast.loading("Parsing large checklist file...");
// ... later
toast.success("File loaded", { id: toastId });
```

## See Also

- [patterns](references/patterns.md) — DO/DON'T pairs, anti-patterns
- [workflows](references/workflows.md) — Multi-step toast workflows

## Related Skills

- See the **shadcn-ui** skill — Toaster wrapper lives in `src/components/ui/sonner.tsx`
- See the **orpc** skill — Toast feedback for IPC operations
- See the **zustand** skill — Toast after store mutations (undo, delete)
- See the **electron** skill — Toast for native dialog results

## Documentation Resources

> Fetch latest sonner documentation with Context7.

**How to use Context7:**

1. Use `mcp__context7__resolve-library-id` to search for "sonner"
2. **Prefer website documentation** (IDs starting with `/websites/`) over source code repositories
3. Query with `mcp__context7__query-docs` using the resolved library ID

**Library ID:** `/websites/sonner_emilkowal_ski`

**Recommended Queries:**

- "sonner toast types and API"
- "sonner promise toast patterns"
- "sonner custom styling and theming"
