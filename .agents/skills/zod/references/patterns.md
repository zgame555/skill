# Zod Patterns Reference

## Contents

- IPC Schema File Convention
- Schema Naming Convention
- Type Inference from Schemas
- Top-Level Validators (v4)
- Composing Schemas
- Anti-Patterns

---

## IPC Schema File Convention

Every oRPC domain has a `schemas.ts` file co-located with `handlers.ts`. Schemas are the **single source of truth** for both runtime validation and TypeScript types.

```
src/ipc/<domain>/
├── handlers.ts   # Imports schemas, wires to os.input()
├── schemas.ts    # Zod schemas + exported inferred types
└── index.ts      # Barrel export
```

```typescript
// src/ipc/checklist/schemas.ts
import z from "zod";

export const importFileInputSchema = z.object({
  filePath: z.string(),
  format: z.enum(["Ace", "Json"]).optional(),
});

export type ImportFileInput = z.infer<typeof importFileInputSchema>;
```

```typescript
// src/ipc/checklist/handlers.ts
import { os } from "@orpc/server";
import { importFileInputSchema } from "./schemas";

export const importFile = os
  .input(importFileInputSchema)
  .handler(async ({ input }) => {
    // input: { filePath: string; format?: "Ace" | "Json" }
  });
```

See the **orpc** skill for the full handler → router → action wiring pattern.

---

## Schema Naming Convention

Follow the project convention: `camelCase` + `Schema` suffix.

```typescript
// GOOD — matches project convention
export const setThemeModeInputSchema = z.enum(["light", "dark", "system"]);
export const openExternalLinkInputSchema = z.object({ url: z.url() });
export const exportFileInputSchema = z.object({
  /* ... */
});

// BAD — inconsistent naming
export const SetThemeMode = z.enum(["light", "dark", "system"]);
export const EXPORT_SCHEMA = z.object({
  /* ... */
});
export const exportInput = z.object({
  /* ... */
});
```

For IPC schemas specifically, use the pattern: `{handlerName}InputSchema`.

---

## Type Inference from Schemas

NEVER duplicate types manually. Derive them from schemas with `z.infer<>`.

```typescript
// GOOD — single source of truth
export const checklistItemSchema = z.object({
  id: z.string(),
  type: z.enum([
    "ChallengeResponse",
    "ChallengeOnly",
    "Title",
    "Note",
    "Warning",
    "Caution",
  ]),
  challengeText: z.string(),
  responseText: z.string().optional(),
  indent: z.number().min(0).max(3),
  centered: z.boolean().default(false),
});

export type ChecklistItem = z.infer<typeof checklistItemSchema>;
```

```typescript
// BAD — manual type that drifts from schema
interface ChecklistItem {
  id: string;
  type:
    | "ChallengeResponse"
    | "ChallengeOnly"
    | "Title"
    | "Note"
    | "Warning"
    | "Caution";
  challengeText: string;
  responseText?: string;
  indent: number;
  centered: boolean;
}
// Now you have TWO sources of truth that WILL diverge
```

---

## Top-Level Validators (v4)

Zod v4 promotes string format validators to the top-level `z` namespace. These are less verbose and more tree-shakable.

```typescript
// Zod v4 — GOOD
z.url(); // validates URL
z.email(); // validates email
z.uuid(); // validates UUID
z.iso.date(); // validates ISO date string
z.iso.datetime(); // validates ISO datetime

// Deprecated — AVOID in new code
z.string().url();
z.string().email();
z.string().uuid();
```

The project already uses `z.url()` in `src/ipc/shell/schemas.ts`.

---

## Composing Schemas

### Extending Objects

```typescript
const baseFileSchema = z.object({
  id: z.string(),
  name: z.string(),
});

const checklistFileSchema = baseFileSchema.extend({
  format: z.enum(["Ace", "Json", "Pdf"]),
  groups: z.array(checklistGroupSchema),
});
```

### Reusing Enum Schemas

```typescript
// Define once, reuse everywhere
export const checklistFormatSchema = z.enum(["Ace", "Json", "Pdf"]);

export const exportInputSchema = z.object({
  format: checklistFormatSchema,
  outputPath: z.string(),
});

export const importInputSchema = z.object({
  format: checklistFormatSchema.optional(),
  filePath: z.string(),
});
```

### Arrays of Objects

```typescript
export const checklistGroupSchema = z.object({
  id: z.string(),
  name: z.string(),
  category: z.enum(["Normal", "Emergency", "Abnormal"]),
  checklists: z.array(checklistSchema),
});
```

---

## Anti-Patterns

### WARNING: Validation at Wrong Boundary

**The Problem:**

```typescript
// BAD — validating inside React components
function PropertiesPanel({ item }: { item: unknown }) {
  const parsed = checklistItemSchema.parse(item);
  return <div>{parsed.challengeText}</div>;
}
```

**Why This Breaks:**

1. Throws runtime errors in the UI layer — poor UX
2. Data should already be validated at the IPC boundary
3. Components should receive typed props, not raw data

**The Fix:**

```typescript
// GOOD — validate at IPC boundary, type flows through
// src/ipc/checklist/handlers.ts
export const importFile = os
  .input(importFileInputSchema)
  .handler(async ({ input }) => {
    const data = await readFile(input.filePath, "utf-8");
    return checklistFileSchema.parse(JSON.parse(data)); // validate HERE
  });

// Component receives typed data — no validation needed
function PropertiesPanel({ item }: { item: ChecklistItem }) {
  return <div>{item.challengeText}</div>;
}
```

### WARNING: Schema Outside schemas.ts

**The Problem:**

```typescript
// BAD — schema defined inline in handler
export const myHandler = os
  .input(z.object({ name: z.string() }))
  .handler(({ input }) => {
    /* ... */
  });
```

**Why This Breaks:**

1. Schema can't be reused or exported for type inference
2. Violates the project's co-location convention
3. Makes it impossible to share the type with action wrappers

**The Fix:**

```typescript
// schemas.ts
export const myHandlerInputSchema = z.object({ name: z.string() });
export type MyHandlerInput = z.infer<typeof myHandlerInputSchema>;

// handlers.ts
import { myHandlerInputSchema } from "./schemas";
export const myHandler = os.input(myHandlerInputSchema).handler(/* ... */);
```

### WARNING: Using `.parse()` for User Input Without Error Handling

**The Problem:**

```typescript
// BAD — .parse() throws ZodError on invalid input
const result = schema.parse(userInput);
```

**Why This Breaks:** Unhandled ZodError crashes the handler. Use `.safeParse()` when input may be invalid and you need to handle the error gracefully.

**The Fix:**

```typescript
// GOOD — safeParse for uncertain input
const result = schema.safeParse(userInput);
if (!result.success) {
  console.error("Validation failed:", result.error.issues);
  return { error: "Invalid input" };
}
return result.data; // typed correctly
```

Note: For oRPC handlers, `.parse()` is fine — oRPC catches validation errors and returns them as typed error responses. Use `.safeParse()` for manual validation outside the IPC layer.
