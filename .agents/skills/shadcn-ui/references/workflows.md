# shadcn/ui Workflows Reference

## Contents

- Adding New shadcn Components
- Building Editor Panel Components
- Extending the Theme for EFIS
- Creating Form Sections
- Building Modal Dialogs
- Command Palette Integration

---

## Adding New shadcn Components

### Workflow Checklist

Copy this checklist and track progress:

- [ ] Step 1: Check if component exists: `ls src/components/ui/`
- [ ] Step 2: Install via CLI: `npx shadcn@latest add <component>`
- [ ] Step 3: Verify import works: `import { X } from "@/components/ui/x"`
- [ ] Step 4: Run lint: `pnpm run lint`

### Components Needed for Editor (Not Yet Installed)

```bash
# Required for Phase 6+ (context menus, dropdowns, toggles)
npx shadcn@latest add switch context-menu dropdown-menu
```

| Component       | Phase | Purpose                                   |
| --------------- | ----- | ----------------------------------------- |
| `switch`        | 9     | Toggle switches in properties panel       |
| `context-menu`  | 5, 6  | Right-click menus on files and checklists |
| `dropdown-menu` | 10    | Toolbar dropdown menus                    |

### Validation Loop

1. Install the component
2. Verify: `ls src/components/ui/<component>.tsx`
3. Test import in a component
4. If CLI fails, check `components.json` aliases match project structure
5. Only proceed when import resolves without TypeScript errors

---

## Building Editor Panel Components

Editor components follow a consistent structure: fixed header + scrollable body + optional footer.

```tsx
// src/components/editor/properties-panel.tsx
import { ScrollArea } from "@/components/ui/scroll-area";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { Separator } from "@/components/ui/separator";
import { cn } from "@/utils/tailwind";

interface PropertiesPanelProps {
  className?: string;
}

export function PropertiesPanel({ className }: PropertiesPanelProps) {
  return (
    <div
      className={cn(
        "border-border bg-surface flex h-full w-[280px] flex-col border-l",
        className,
      )}
    >
      {/* Fixed header */}
      <div className="border-border border-b px-3.5 py-3">
        <h2 className="text-muted-foreground text-[11px] font-semibold tracking-wide uppercase">
          Item Properties
        </h2>
      </div>

      {/* Scrollable body */}
      <ScrollArea className="flex-1">
        <div className="space-y-5 p-3.5">
          {/* Section */}
          <div className="space-y-2">
            <h3 className="text-muted text-[11px] font-semibold tracking-wide uppercase">
              Selected Item
            </h3>
            <div className="space-y-1.5">
              <Label className="text-muted-foreground text-[11px]">Type</Label>
              <Select>
                <SelectTrigger>
                  <SelectValue placeholder="Select type" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="challenge-response">
                    Challenge / Response
                  </SelectItem>
                  <SelectItem value="title">Title / Section Header</SelectItem>
                  <SelectItem value="note">Note</SelectItem>
                </SelectContent>
              </Select>
            </div>
          </div>

          <Separator />

          <div className="space-y-2">
            <h3 className="text-muted text-[11px] font-semibold tracking-wide uppercase">
              Formatting
            </h3>
            <div className="space-y-1.5">
              <Label className="text-muted-foreground text-[11px]">
                Challenge Text
              </Label>
              <Input
                className="bg-elevated text-xs"
                placeholder="Enter challenge..."
              />
            </div>
          </div>
        </div>
      </ScrollArea>
    </div>
  );
}
```

### Panel Structure Pattern

```
+------------------+
| SECTION HEADER   |  ← text-[11px] font-semibold uppercase tracking-wide
+------------------+
| Label            |  ← text-[11px] text-muted-foreground
| [Input/Select]   |  ← bg-elevated border-border text-xs
| Label            |
| [Input/Select]   |
+------------------+  ← Separator
| SECTION HEADER   |
+------------------+
```

Every panel follows: `bg-surface` background, `border-border` borders, `px-3.5` horizontal padding, section headers as `text-[11px] font-semibold uppercase tracking-wide text-muted`.

---

## Extending the Theme for EFIS

EFIS color tokens are added to `src/styles/global.css` in the `@theme inline` block. See the **tailwind** skill for the full token reference.

### Adding a New Token

```css
/* In src/styles/global.css, inside @theme inline { ... } */
@theme inline {
  /* Existing tokens... */
  --color-bg-base: var(--bg-base);
  --color-bg-surface: var(--bg-surface);
  --color-bg-elevated: var(--bg-elevated);
  /* Add new EFIS tokens as needed */
}
```

Then define the CSS variable value:

```css
.dark {
  --bg-base: #0d1117;
  --bg-surface: #161b22;
  --bg-elevated: #1c2333;
}
```

### Using EFIS Tokens in Components

```tsx
// These become available as Tailwind classes
<div className="bg-base">           {/* #0d1117 */}
<div className="bg-surface">        {/* #161b22 */}
<div className="bg-elevated">       {/* #1c2333 */}
<div className="bg-hover">          {/* #292e3e */}
<div className="bg-accent-dim">     {/* #1f3a5f */}
<span className="text-green">       {/* #3fb950 */}
<span className="text-purple">      {/* #a371f7 */}
```

---

## Creating Form Sections

Properties panel and metadata forms follow a strict pattern.

```tsx
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

function FormField({
  label,
  children,
}: {
  label: string;
  children: React.ReactNode;
}) {
  return (
    <div className="space-y-1.5">
      <Label className="text-muted-foreground text-[11px]">{label}</Label>
      {children}
    </div>
  );
}

// Usage
<FormField label="Aircraft Registration">
  <Input
    className="bg-elevated border-border text-xs"
    value={registration}
    onChange={(e) => setRegistration(e.target.value)}
  />
</FormField>;
```

### Input Styling Rules

| State   | Classes                                                                                |
| ------- | -------------------------------------------------------------------------------------- |
| Default | `bg-elevated border border-border rounded-[4px] px-2.5 py-1.5 text-xs text-foreground` |
| Focus   | `border-accent outline-none` (handled by shadcn Input)                                 |
| Error   | `border-destructive` (add via className)                                               |

---

## Building Modal Dialogs

The export modal uses a 2-column grid of clickable cards inside a Dialog.

```tsx
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { cn } from "@/utils/tailwind";

interface FormatOption {
  name: string;
  extension: string;
  description: string;
  disabled?: boolean;
}

function ExportModal({
  open,
  onOpenChange,
}: {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}) {
  const formats: FormatOption[] = [
    {
      name: "Garmin G3X",
      extension: ".ace",
      description: "G3X, G3X Touch, GTN",
    },
    {
      name: "JSON Backup",
      extension: ".json",
      description: "Lossless internal format",
    },
    { name: "Printable PDF", extension: ".pdf", description: "Paper backup" },
    {
      name: "AFS / Dynon",
      extension: ".afd",
      description: "Coming soon",
      disabled: true,
    },
  ];

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="bg-elevated sm:max-w-[480px]">
        <DialogHeader>
          <DialogTitle className="text-base font-semibold">
            Export Checklists
          </DialogTitle>
        </DialogHeader>
        <div className="grid grid-cols-2 gap-2">
          {formats.map((fmt) => (
            <button
              key={fmt.extension}
              disabled={fmt.disabled}
              className={cn(
                "border-border flex flex-col gap-1 rounded-lg border p-3 text-left transition-colors",
                fmt.disabled
                  ? "cursor-not-allowed opacity-40"
                  : "hover:border-accent hover:bg-accent-dim cursor-pointer",
              )}
            >
              <span className="text-foreground text-[13px] font-bold">
                {fmt.name}
              </span>
              <span className="text-muted-foreground font-mono text-[11px]">
                {fmt.extension}
              </span>
              <span className="text-secondary text-[11px]">
                {fmt.description}
              </span>
            </button>
          ))}
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

### Dialog Sizing

| Use Case       | Max Width               | Example             |
| -------------- | ----------------------- | ------------------- |
| Confirm dialog | `sm:max-w-sm` (default) | Delete confirmation |
| Export modal   | `sm:max-w-[480px]`      | Format grid         |
| Large form     | `sm:max-w-lg`           | Settings            |

---

## Command Palette Integration

The command palette uses the existing `Command` component (wraps `cmdk`). It's triggered by Ctrl+K globally.

```tsx
// src/components/editor/command-palette.tsx
import {
  CommandDialog,
  Command,
  CommandInput,
  CommandList,
  CommandGroup,
  CommandItem,
  CommandEmpty,
  CommandShortcut,
} from "@/components/ui/command";

interface ChecklistCommandPaletteProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
}

export function ChecklistCommandPalette({
  open,
  onOpenChange,
}: ChecklistCommandPaletteProps) {
  return (
    <CommandDialog
      open={open}
      onOpenChange={onOpenChange}
      title="Search checklists and items"
      description="Navigate to any checklist or item"
    >
      <Command>
        <CommandInput placeholder="Search checklists and items..." />
        <CommandList>
          <CommandEmpty>No results found.</CommandEmpty>
          <CommandGroup heading="Checklists">
            <CommandItem
              onSelect={() => {
                /* navigate to checklist */
              }}
            >
              Preflight Inspection
              <CommandShortcut>Normal</CommandShortcut>
            </CommandItem>
          </CommandGroup>
          <CommandGroup heading="Items">
            <CommandItem
              onSelect={() => {
                /* navigate to item */
              }}
            >
              Parking Brake ... SET
              <CommandShortcut>Preflight</CommandShortcut>
            </CommandItem>
          </CommandGroup>
        </CommandList>
      </Command>
    </CommandDialog>
  );
}
```

### Key Integration Points

- Mount in `src/layouts/editor-layout.tsx`
- State managed by **zustand** ui-store (see the **zustand** skill)
- Keyboard shortcut registered in `src/hooks/use-keyboard-shortcuts.ts`
- Results populated from checklist store data

### WARNING: CommandDialog vs Dialog

**The Problem:**

```tsx
// BAD — using raw Dialog for command palette
<Dialog>
  <DialogContent>
    <Command>...</Command>
  </DialogContent>
</Dialog>
```

**Why This Breaks:**

1. `CommandDialog` adds `overflow-hidden rounded-xl! p-0` and hides the close button by default
2. It includes accessible `sr-only` title/description
3. Raw Dialog has padding and close button that break command palette layout

**The Fix:** Always use `CommandDialog` for command palette patterns. Use `Dialog` for standard modals.

---

## Component File Organization

```
src/components/
├── ui/                          # shadcn primitives (DO NOT MODIFY)
├── shared/                      # Reusable across pages
│   ├── drag-window-region.tsx
│   └── index.ts                 # Barrel export
├── editor/                      # Editor-specific components
│   ├── toolbar.tsx
│   ├── toolbar-button.tsx       # Wraps Button with tooltip
│   ├── files-sidebar.tsx
│   ├── checklist-tree.tsx
│   ├── checklist-editor.tsx
│   ├── checklist-item-row.tsx
│   ├── properties-panel.tsx
│   ├── status-bar.tsx
│   ├── export-modal.tsx         # Wraps Dialog
│   ├── command-palette.tsx      # Wraps CommandDialog
│   ├── format-badge.tsx         # Wraps Badge
│   ├── group-icon.tsx
│   ├── type-indicator.tsx
│   ├── indent-guides.tsx
│   └── index.ts                 # Barrel export
└── home/                        # Welcome page
    └── ...
```

### Naming Convention

- Files: `kebab-case.tsx`
- Components: `PascalCase` named export
- Props: `<ComponentName>Props` interface
- Barrel: `index.ts` with `export { X } from "./x"`

See the **react** skill for component patterns and the **tailwind** skill for styling conventions.
