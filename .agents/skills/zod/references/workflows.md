# Zod Workflows Reference

## Contents

- Adding a New IPC Schema
- Defining Checklist Data Schemas
- Validating Imported File Data
- Migrating Zod v3 Patterns to v4

---

## Adding a New IPC Schema

Every new oRPC handler that accepts input needs a Zod schema. Follow this checklist:

Copy this checklist and track progress:

- [ ] Step 1: Create or edit `src/ipc/<domain>/schemas.ts`
- [ ] Step 2: Define schema with `InputSchema` suffix
- [ ] Step 3: Export inferred type with `z.infer<>`
- [ ] Step 4: Import schema in `handlers.ts` and wire to `os.input()`
- [ ] Step 5: Verify type flows to action wrapper in `src/actions/`

### Step-by-Step

```typescript
// 1. src/ipc/dialog/schemas.ts
import z from "zod";

export const openFileDialogInputSchema = z.object({
  filters: z
    .array(
      z.object({
        name: z.string(),
        extensions: z.array(z.string()),
      }),
    )
    .optional(),
  defaultPath: z.string().optional(),
  title: z.string().optional(),
});

export type OpenFileDialogInput = z.infer<typeof openFileDialogInputSchema>;
```

```typescript
// 2. src/ipc/dialog/handlers.ts
import { os } from "@orpc/server";
import { dialog } from "electron";
import { openFileDialogInputSchema } from "./schemas";

export const openFileDialog = os
  .input(openFileDialogInputSchema)
  .handler(async ({ input }) => {
    const result = await dialog.showOpenDialog({
      filters: input.filters,
      defaultPath: input.defaultPath,
      title: input.title,
      properties: ["openFile"],
    });
    return result.canceled ? null : result.filePaths[0];
  });
```

```typescript
// 3. src/actions/dialog.ts
import { ipc } from "@/ipc/manager";
import type { OpenFileDialogInput } from "@/ipc/dialog/schemas";

export async function openFileDialog(options?: OpenFileDialogInput) {
  return ipc.client.dialog.openFileDialog(options ?? {});
}
```

See the **orpc** skill for the full handler → router → action pattern.

---

## Defining Checklist Data Schemas

For the checklist data model, schemas serve dual purpose: runtime validation of imported data AND TypeScript type generation.

```typescript
// src/ipc/checklist/schemas.ts
import z from "zod";

// Reusable enums
export const checklistItemTypeSchema = z.enum([
  "ChallengeResponse",
  "ChallengeOnly",
  "Title",
  "Note",
  "Warning",
  "Caution",
]);

export const checklistGroupCategorySchema = z.enum([
  "Normal",
  "Emergency",
  "Abnormal",
]);

export const checklistFormatSchema = z.enum(["Ace", "Json", "Pdf"]);

// Item schema — flat with indent level (NOT nested tree)
export const checklistItemSchema = z.object({
  id: z.string(),
  type: checklistItemTypeSchema,
  challengeText: z.string().default(""),
  responseText: z.string().default(""),
  indent: z.number().min(0).max(3).default(0),
  centered: z.boolean().default(false),
  collapsible: z.boolean().default(false),
});

export const checklistSchema = z.object({
  id: z.string(),
  name: z.string(),
  items: z.array(checklistItemSchema),
});

export const checklistGroupSchema = z.object({
  id: z.string(),
  name: z.string(),
  category: checklistGroupCategorySchema,
  checklists: z.array(checklistSchema),
});

export const checklistFileMetadataSchema = z.object({
  aircraftRegistration: z.string().default(""),
  makeModel: z.string().default(""),
  copyright: z.string().default(""),
});

export const checklistFileSchema = z.object({
  id: z.string(),
  name: z.string(),
  format: checklistFormatSchema,
  filePath: z.string().optional(),
  groups: z.array(checklistGroupSchema),
  metadata: checklistFileMetadataSchema,
  lastModified: z.number(),
});

// Export inferred types — these ARE the app's data model types
export type ChecklistItemType = z.infer<typeof checklistItemTypeSchema>;
export type ChecklistItem = z.infer<typeof checklistItemSchema>;
export type Checklist = z.infer<typeof checklistSchema>;
export type ChecklistGroup = z.infer<typeof checklistGroupSchema>;
export type ChecklistFile = z.infer<typeof checklistFileSchema>;
```

---

## Validating Imported File Data

When importing files from disk, use `.safeParse()` to handle malformed data gracefully.

```typescript
// src/ipc/checklist/handlers.ts
import { checklistFileSchema } from "./schemas";

export const importJsonFile = os
  .input(z.object({ filePath: z.string() }))
  .handler(async ({ input }) => {
    const raw = await readFile(input.filePath, "utf-8");
    let parsed: unknown;

    try {
      parsed = JSON.parse(raw);
    } catch {
      throw new Error("File is not valid JSON");
    }

    const result = checklistFileSchema.safeParse(parsed);
    if (!result.success) {
      const issues = result.error.issues
        .map((i) => `${i.path.join(".")}: ${i.message}`)
        .join("; ");
      throw new Error(`Invalid checklist file: ${issues}`);
    }

    return result.data;
  });
```

1. Parse JSON first (catches syntax errors)
2. Validate structure with `.safeParse()` (catches schema violations)
3. Format error messages from `result.error.issues` for user-friendly toast
4. Only proceed when validation passes

---

## Migrating Zod v3 Patterns to v4

This project uses Zod v4 (`^4.2.1`). If you encounter v3 patterns in examples or AI suggestions, apply these conversions:

### Import Style

```typescript
// Both work in v4
import z from "zod"; // default import — preferred for new files
import { z } from "zod"; // named import — also fine
```

### Top-Level Validators

```typescript
// v3 → v4
z.string().url()      →  z.url()
z.string().email()    →  z.email()
z.string().uuid()     →  z.uuid()
z.string().datetime() →  z.iso.datetime()
```

### Default Value Behavior

```typescript
// v4: .default() must match OUTPUT type, not input type
const schema = z
  .string()
  .transform((s) => s.length)
  .default(0);
// default is 0 (number) because output type is number

// Use .prefault() for v3 behavior (matches input type)
const schema = z
  .string()
  .transform((s) => s.length)
  .prefault("hello");
// prefault is "hello" (string) — gets parsed through transform
```

### Error Messages

```typescript
// v3 — separate params
z.string({ invalid_type_error: "Not a string", required_error: "Required" });

// v4 — unified error param (v3 form still works but deprecated)
z.string({
  error: (issue) => (issue.input === undefined ? "Required" : "Not a string"),
});
```

### Record with Enum Keys

```typescript
// v4: enum keys are now REQUIRED in records
const schema = z.record(z.enum(["a", "b"]), z.number());
// Type: { a: number; b: number } — all keys required
```

Iterate-until-pass validation workflow:

1. Write or update schema
2. Validate: `pnpm run lint` (catches type errors from inference)
3. If lint fails, check that `z.infer<>` types align with usage
4. Only proceed when lint passes
