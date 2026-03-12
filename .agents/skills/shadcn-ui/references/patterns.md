# shadcn/ui Patterns Reference

## Contents

- Component Composition
- Variant Override Pattern
- Conditional Styling with cn()
- Custom Wrapper Components
- Data-Slot Targeting
- Anti-Patterns

---

## Component Composition

shadcn components use a **compound component** pattern. Never render flat HTML — compose from sub-components.

```tsx
// GOOD — compound composition
import {
  Card,
  CardHeader,
  CardTitle,
  CardDescription,
  CardContent,
} from "@/components/ui/card";

function ChecklistCard({ name, count }: { name: string; count: number }) {
  return (
    <Card size="sm">
      <CardHeader>
        <CardTitle className="text-[13px]">{name}</CardTitle>
        <CardDescription>{count} items</CardDescription>
      </CardHeader>
      <CardContent>{/* content */}</CardContent>
    </Card>
  );
}
```

```tsx
// BAD — bypassing compound pattern
<div className="bg-card rounded-xl border p-4">
  <h3>Title</h3>
  <p>Description</p>
</div>
```

**Why compound matters:** Card internally sets `data-slot="card"`, handles size variants via `data-[size=sm]`, and manages consistent spacing. Raw divs lose all of this.

---

## Variant Override Pattern

Use variant props for standard states. Use `className` only for project-specific overrides that don't exist as variants.

```tsx
// GOOD — using variant props
<Button variant="ghost" size="icon-sm">
  <FontAwesomeIcon icon={faUndo} />
</Button>

// GOOD — className for EFIS-specific toolbar button style
<Button
  variant="ghost"
  size="sm"
  className="text-muted-foreground hover:bg-hover hover:text-foreground"
>
  Import
</Button>

// GOOD — primary accent style for Quick Export
<Button className="bg-accent-dim text-accent border border-accent/25 hover:bg-accent/20">
  Quick Export
</Button>
```

### WARNING: Conflicting Class Overrides

**The Problem:**

```tsx
// BAD — className conflicts with variant base classes
<Button variant="default" className="bg-red-500">
  Delete
</Button>
```

**Why This Breaks:**

1. `variant="default"` applies `bg-primary` — `twMerge` resolves conflicts but intent is unclear
2. If you want destructive styling, use `variant="destructive"` instead
3. Mixing variant semantics with manual overrides creates maintenance confusion

**The Fix:**

```tsx
// GOOD — use the right variant
<Button variant="destructive" size="sm">
  Delete
</Button>;

// GOOD — or create a custom wrapper if no variant fits
function ToolbarButton({
  className,
  ...props
}: React.ComponentProps<typeof Button>) {
  return (
    <Button
      variant="ghost"
      size="sm"
      className={cn("text-muted-foreground", className)}
      {...props}
    />
  );
}
```

---

## Conditional Styling with cn()

The `cn()` utility from `@/utils/tailwind` combines `clsx` (conditional classes) with `twMerge` (conflict resolution). Use it everywhere.

```tsx
import { cn } from "@/utils/tailwind";

// Boolean conditions
<div className={cn(
  "flex items-center gap-2 px-3.5 py-1.5 text-xs",
  isActive && "bg-active text-foreground",
  isHovered && "bg-hover",
  !isActive && !isHovered && "text-muted-foreground"
)} />

// Object syntax
<div className={cn("base-class", {
  "bg-accent-dim text-accent": isSelected,
  "opacity-40 cursor-not-allowed": isDisabled,
})} />

// Forwarding className prop (always last to allow consumer overrides)
function FileItem({ className, ...props }: FileItemProps) {
  return (
    <div className={cn("flex items-center gap-2 px-3.5", className)} {...props} />
  );
}
```

### WARNING: Forgetting to Forward className

**The Problem:**

```tsx
// BAD — ignores consumer className, can't be styled externally
function StatusBadge({ label }: { label: string }) {
  return (
    <span className="bg-overlay rounded-full px-1.5 text-xs">{label}</span>
  );
}
```

**Why This Breaks:**

1. Consumer cannot override styles: `<StatusBadge className="bg-green-dim" />` is silently ignored
2. Every shadcn component accepts and merges className — your custom components should too

**The Fix:**

```tsx
function StatusBadge({
  label,
  className,
}: {
  label: string;
  className?: string;
}) {
  return (
    <span className={cn("bg-overlay rounded-full px-1.5 text-xs", className)}>
      {label}
    </span>
  );
}
```

---

## Custom Wrapper Components

When shadcn variants don't cover your needs, create wrappers in `src/components/editor/` or `src/components/shared/`. Never modify `src/components/ui/`.

```tsx
// src/components/editor/toolbar-button.tsx
import { Button } from "@/components/ui/button";
import {
  Tooltip,
  TooltipContent,
  TooltipTrigger,
} from "@/components/ui/tooltip";
import { cn } from "@/utils/tailwind";

interface ToolbarButtonProps extends React.ComponentProps<typeof Button> {
  tooltip?: string;
  shortcut?: string;
}

export function ToolbarButton({
  tooltip,
  shortcut,
  className,
  children,
  ...props
}: ToolbarButtonProps) {
  const button = (
    <Button
      variant="ghost"
      size="sm"
      className={cn(
        "text-muted-foreground hover:bg-hover hover:text-foreground",
        className,
      )}
      {...props}
    >
      {children}
    </Button>
  );

  if (!tooltip) return button;

  return (
    <Tooltip>
      <TooltipTrigger asChild>{button}</TooltipTrigger>
      <TooltipContent>
        {tooltip}
        {shortcut && (
          <kbd className="text-muted ml-2 text-[10px]">{shortcut}</kbd>
        )}
      </TooltipContent>
    </Tooltip>
  );
}
```

---

## Data-Slot Targeting

Every shadcn component emits `data-slot` attributes. Use these for CSS targeting instead of fragile class selectors.

```css
/* In global.css or component-level styles via Tailwind arbitrary selectors */
/* Target all buttons inside a toolbar */
[data-slot="toolbar"] [data-slot="button"] {
  /* styles */
}
```

```tsx
// Tailwind arbitrary selector for data-slot
<div className="[&_[data-slot=button]]:h-8">
  <Button>Taller in this context</Button>
</div>
```

Available data-slot values in this project: `button`, `card`, `dialog`, `dialog-content`, `dialog-overlay`, `command`, `command-input`, `command-item`, `select-trigger`, `select-content`, `scroll-area`, `separator`, `badge`, `input`, `label`, and more.

---

## Anti-Patterns

### WARNING: Editing shadcn/ui Source Files

**The Problem:**

```tsx
// BAD — modifying src/components/ui/button.tsx
// Adding a custom "toolbar" variant directly in the CVA config
```

**Why This Breaks:**

1. Running `pnpm run bump-shadcn-components` overwrites ALL files in `src/components/ui/`
2. Custom changes are silently lost with no merge conflict warning
3. The shadcn CLI treats these as generated files

**The Fix:** Create wrapper components in `src/components/editor/` or `src/components/shared/`.

### WARNING: Hardcoded Colors Instead of Design Tokens

**The Problem:**

```tsx
// BAD — hardcoded hex values
<div className="bg-[#0d1117] text-[#e6edf3] border-[#30363d]">
```

**Why This Breaks:**

1. Bypasses the theme system — won't respond to token changes
2. EFIS tokens (`bg-base`, `text-primary`, `border-border`) are already mapped in `@theme inline`
3. See the **tailwind** skill for the full EFIS token reference

**The Fix:**

```tsx
// GOOD — use design tokens
<div className="bg-base text-primary border-border">
```

### WARNING: Inline Styles

```tsx
// BAD — violates CLAUDE.md styling rules
<div style={{ backgroundColor: "#0d1117", padding: "8px" }}>

// GOOD — always Tailwind
<div className="bg-base p-2">
```

This is a **hard rule** in this project. See the **tailwind** skill for details.

### WARNING: Using Radix Primitives Directly

**The Problem:**

```tsx
// BAD — importing Radix directly instead of shadcn wrapper
import { Dialog as DialogPrimitive } from "radix-ui";

function MyDialog() {
  return <DialogPrimitive.Root>...</DialogPrimitive.Root>;
}
```

**Why This Breaks:**

1. Loses all project-specific styling (animations, colors, border radius)
2. Loses `data-slot` attributes used for global targeting
3. Loses custom props like `showCloseButton` on DialogContent

**The Fix:** Always import from `@/components/ui/`, never from `radix-ui` directly.
