# Sonner Patterns Reference

## Contents

- Toast Type Selection
- Toaster Configuration
- Theme Integration
- Anti-Patterns
- Integration with IPC and Store

---

## Toast Type Selection

Use the correct toast type for semantic meaning. Each type renders a distinct FontAwesome icon configured in `src/components/ui/sonner.tsx`.

| Type              | When to Use                                      | Icon                    |
| ----------------- | ------------------------------------------------ | ----------------------- |
| `toast.success()` | Operation completed: export, save, import        | `faCircleCheck`         |
| `toast.error()`   | Operation failed: parse error, permission denied | `faCircleXmark`         |
| `toast.warning()` | Non-blocking issue: format incompatibility       | `faTriangleExclamation` |
| `toast.info()`    | Status update: autosave enabled, feature hint    | `faCircleInfo`          |
| `toast.loading()` | In-progress: file parsing, PDF generation        | `faSpinner` (animated)  |
| `toast()`         | Neutral: item duplicated, checklist renamed      | No icon                 |

```typescript
// DO — Semantic toast types
toast.success("Checklist saved");
toast.error("Failed to parse ACE file", {
  description: "Invalid XML at line 42",
});
toast.warning("3 items are incompatible with PDF export");

// DON'T — Generic toast for errors
toast("Something went wrong"); // No visual severity indicator
```

---

## Toaster Configuration

The Toaster is configured once in `src/components/ui/sonner.tsx`. NEVER modify this shadcn-managed file directly — extend via props on `<Toaster />` in `src/App.tsx` if needed.

### Current Configuration

```typescript
// src/components/ui/sonner.tsx — DO NOT EDIT (shadcn-managed)
<Sonner
  theme={theme as ToasterProps["theme"]}  // synced with next-themes
  className="toaster group"
  icons={{
    success: <FontAwesomeIcon icon={faCircleCheck} className="size-4" />,
    info: <FontAwesomeIcon icon={faCircleInfo} className="size-4" />,
    warning: <FontAwesomeIcon icon={faTriangleExclamation} className="size-4" />,
    error: <FontAwesomeIcon icon={faCircleXmark} className="size-4" />,
    loading: <FontAwesomeIcon icon={faSpinner} className="size-4 animate-spin" />,
  }}
  style={{
    "--normal-bg": "var(--popover)",
    "--normal-text": "var(--popover-foreground)",
    "--normal-border": "var(--border)",
    "--border-radius": "var(--radius)",
  } as React.CSSProperties}
/>
```

### Overriding Defaults Per-Toast

```typescript
// Override duration for a specific toast
toast.success("Export complete!", { duration: 6000 });

// Override position for a critical error
toast.error("File corrupted", { position: "top-center" });

// Add a close button for persistent messages
toast.info("New version available", {
  closeButton: true,
  duration: Infinity,
});
```

---

## Theme Integration

Sonner reads the theme from `next-themes` and maps to the app's OKLCH CSS variables. This happens automatically — no manual theme handling needed.

```typescript
// DO — Just call toast(). Theme is handled by the wrapper.
toast.success("Saved");

// DON'T — Never pass theme-specific styles to individual toasts
toast.success("Saved", {
  style: { backgroundColor: "#0d1117", color: "#e6edf3" }, // WRONG
});
```

The wrapper maps these CSS variables for dark mode compatibility:

- `--normal-bg` → `var(--popover)` (OKLCH dark surface)
- `--normal-text` → `var(--popover-foreground)` (OKLCH light text)
- `--normal-border` → `var(--border)` (OKLCH border)

See the **tailwind** skill for the full OKLCH color token system.

---

## Anti-Patterns

### WARNING: Multiple Toaster Instances

**The Problem:**

```typescript
// BAD — Mounting Toaster in a layout AND App.tsx
// src/layouts/editor-layout.tsx
export default function EditorLayout() {
  return (
    <div>
      <Toolbar />
      <EditorPanels />
      <Toaster /> {/* DUPLICATE — already in App.tsx */}
    </div>
  );
}
```

**Why This Breaks:**

1. Duplicate toasts appear (one from each Toaster instance)
2. Inconsistent positioning — each Toaster manages its own stack
3. Memory leak from duplicate event listeners

**The Fix:** Toaster is mounted ONCE in `src/App.tsx`. Never mount it elsewhere.

### WARNING: Swallowing Errors with Generic Toasts

**The Problem:**

```typescript
// BAD — No context for the user
try {
  await importFile(path);
} catch {
  toast.error("Error");
}
```

**Why This Breaks:**

1. User has no idea what failed or how to fix it
2. Debugging requires checking console — users won't do this

**The Fix:**

```typescript
// GOOD — Specific error with context
try {
  await importFile(path);
  toast.success("File imported", { description: `Loaded ${fileName}` });
} catch (err) {
  toast.error(`Failed to import ${fileName}`, {
    description: err instanceof Error ? err.message : "Unknown error",
  });
}
```

### WARNING: Toast Spam from Rapid Operations

**The Problem:**

```typescript
// BAD — Autosave triggers toast every 2 seconds
useEffect(() => {
  const interval = setInterval(() => {
    saveFile(activeFile);
    toast.success("Saved"); // Toast every 2 seconds!
  }, 2000);
  return () => clearInterval(interval);
}, [activeFile]);
```

**Why This Breaks:** Toast stack overflows, user can't see important messages, notification fatigue.

**The Fix:** Use `toast.dismiss()` with a stable ID for repeated updates:

```typescript
// GOOD — Reuse toast ID for autosave
const AUTOSAVE_TOAST_ID = "autosave";

function onAutosave() {
  toast.success("Changes saved", {
    id: AUTOSAVE_TOAST_ID,
    duration: 2000,
  });
}
```

---

## Integration with IPC and Store

### IPC Action Feedback Pattern

Wrap oRPC calls with toast feedback. See the **orpc** skill for the IPC pattern.

```typescript
// src/actions/checklist.ts — renderer-side action wrapper
import { toast } from "sonner";
import { ipc } from "@/ipc/manager";

export async function importChecklistFile() {
  const result = await ipc.client.checklist.importFile();
  if (!result) return null; // user cancelled dialog

  toast.success(`Imported ${result.name}`, {
    description: `${result.format} format, ${result.groups.length} groups`,
  });
  return result;
}
```

### Store Mutation Feedback

Toast after Zustand store actions for destructive operations. See the **zustand** skill.

```typescript
// In a component handling delete
function handleDeleteChecklist(id: string, name: string) {
  const previousState = store.getState();
  store.getState().removeChecklist(id);

  toast("Checklist deleted", {
    description: name,
    action: {
      label: "Undo",
      onClick: () => store.setState(previousState),
    },
  });
}
```
