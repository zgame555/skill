---
name: shadcn-ui
description: |
  Builds UI using shadcn/ui primitives with Radix UI, CVA variants, and Tailwind CSS 4.
  Use when: creating or modifying components in src/components/, composing UI from shadcn primitives,
  adding new shadcn components via CLI, styling with design tokens, building modals/dialogs/forms,
  or working with the command palette.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, mcp__context7__resolve-library-id, mcp__context7__query-docs
---

# shadcn/ui Skill

This project uses shadcn/ui v3.6.2 with the `radix-mira` style, Tailwind CSS 4 (CSS-first, no config file), and OKLCH color variables. Components live in `src/components/ui/` and are **never modified directly** — they are managed by the shadcn CLI. Custom components compose these primitives.

## Quick Start

### Composing Components

```tsx
import { Button } from "@/components/ui/button";
import { Card, CardHeader, CardTitle, CardContent } from "@/components/ui/card";
import { cn } from "@/utils/tailwind";

function MyPanel({ className }: { className?: string }) {
  return (
    <Card size="sm" className={cn("bg-surface", className)}>
      <CardHeader>
        <CardTitle>Panel Title</CardTitle>
      </CardHeader>
      <CardContent>
        <Button variant="ghost" size="sm">
          Action
        </Button>
      </CardContent>
    </Card>
  );
}
```

### Adding New Components

```bash
npx shadcn@latest add switch context-menu dropdown-menu
```

### Overriding Styles (Never Edit ui/ Files)

```tsx
// GOOD — override via className prop
<Button
  variant="ghost"
  className="bg-accent-dim text-accent border-accent/25 border"
>
  Quick Export
</Button>

// BAD — editing src/components/ui/button.tsx directly
```

## Key Concepts

| Concept         | Usage                                    | Example                                             |
| --------------- | ---------------------------------------- | --------------------------------------------------- |
| `cn()`          | Merge Tailwind classes without conflicts | `cn("px-2", conditional && "px-4")`                 |
| CVA variants    | Type-safe component variants             | `variant="ghost"`, `size="icon-sm"`                 |
| `asChild`       | Polymorphic rendering via Radix Slot     | `<Button asChild><Link to="/">Home</Link></Button>` |
| `data-slot`     | Semantic CSS targeting                   | `data-slot="button"` on every Button                |
| Design tokens   | Theme colors via CSS variables           | `bg-background`, `text-muted-foreground`            |
| `@theme inline` | Tailwind 4 CSS-first theming             | Maps CSS vars to Tailwind utilities in `global.css` |

## Installed Components (21)

**Layout**: Card, ScrollArea, Separator
**Forms**: Button, Input, Label, Select, Textarea, InputGroup, NumberInput
**Feedback**: Alert, Badge, Progress, Spinner, Sonner
**Overlays**: Dialog, Popover, Tooltip, Command
**Dates**: Calendar, DatePicker

## Button Variants Reference

| Variant       | Use For                         |
| ------------- | ------------------------------- |
| `default`     | Primary actions                 |
| `outline`     | Secondary actions with border   |
| `ghost`       | Toolbar buttons, subtle actions |
| `secondary`   | Alternative actions             |
| `destructive` | Delete, remove, danger          |
| `link`        | Text links with underline       |

Sizes: `xs`, `sm`, `default`, `lg`, `icon`, `icon-xs`, `icon-sm`, `icon-lg`

## Common Patterns

### Dialog with Custom Content

```tsx
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogFooter,
} from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";

function ExportModal({
  open,
  onOpenChange,
}: {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}) {
  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-md">
        <DialogHeader>
          <DialogTitle>Export Checklists</DialogTitle>
        </DialogHeader>
        {/* Grid of format options */}
        <DialogFooter showCloseButton>
          <Button>Export</Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

### Command Palette

```tsx
import {
  CommandDialog,
  Command,
  CommandInput,
  CommandList,
  CommandGroup,
  CommandItem,
  CommandEmpty,
} from "@/components/ui/command";

function SearchPalette({
  open,
  onOpenChange,
}: {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}) {
  return (
    <CommandDialog
      open={open}
      onOpenChange={onOpenChange}
      title="Search checklists"
    >
      <Command>
        <CommandInput placeholder="Search checklists and items..." />
        <CommandList>
          <CommandEmpty>No results found.</CommandEmpty>
          <CommandGroup heading="Checklists">
            <CommandItem>Preflight Inspection</CommandItem>
          </CommandGroup>
        </CommandList>
      </Command>
    </CommandDialog>
  );
}
```

## See Also

- [patterns](references/patterns.md) - Component composition, variant creation, theming
- [workflows](references/workflows.md) - Adding components, building editor UI, extending theme

## Related Skills

- See the **react** skill for component lifecycle and hooks patterns
- See the **tailwind** skill for utility class usage and EFIS theme tokens
- See the **lucide-react** skill for icon integration (planned migration from FontAwesome)
- See the **sonner** skill for toast notifications
- See the **zod** skill for form validation patterns

## Documentation Resources

> Fetch latest shadcn/ui documentation with Context7.

**How to use Context7:**

1. Use `mcp__context7__resolve-library-id` to search for "shadcn-ui"
2. **Prefer website documentation** (IDs starting with `/websites/`) over source code repositories
3. Query with `mcp__context7__query-docs` using the resolved library ID

**Library ID:** `/websites/ui_shadcn` _(website docs, 1729 snippets, High reputation)_

**Recommended Queries:**

- "shadcn button variants and sizes"
- "shadcn dialog modal usage"
- "shadcn command palette cmdk"
- "shadcn scroll-area custom styling"
- "shadcn select component"
