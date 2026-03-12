---
name: zod
description: |
  Validates IPC handler inputs and data schemas with Zod v4.
  Use when: defining IPC handler input schemas, validating checklist data structures,
  parsing imported file data, or creating type-safe validation for any data boundary.
allowed-tools: Read, Edit, Write, Glob, Grep, Bash, mcp__context7__resolve-library-id, mcp__context7__query-docs
---

# Zod v4 Validation

This project uses **Zod v4** (`^4.2.1`) for schema validation at IPC boundaries. Every oRPC handler that accepts input validates it through a Zod schema defined in a co-located `schemas.ts` file. Schemas also serve as the single source of truth for TypeScript types via `z.infer<>`.

## Quick Start

### IPC Input Schema (Primary Pattern)

```typescript
// src/ipc/<domain>/schemas.ts
import z from "zod";

export const writeChecklistFileInputSchema = z.object({
  filePath: z.string(),
  content: z.string(),
});

// src/ipc/<domain>/handlers.ts
import { os } from "@orpc/server";
import { writeChecklistFileInputSchema } from "./schemas";

export const writeChecklistFile = os
  .input(writeChecklistFileInputSchema)
  .handler(async ({ input }) => {
    // input is typed as { filePath: string; content: string }
  });
```

### Zod v4 Import

```typescript
// Default import — project convention
import z from "zod";

// Named import also works
import { z } from "zod";
```

Both are valid. Pick one per file and stay consistent. The project currently uses both — prefer `import z from "zod"` for new files.

## Key Concepts

| Concept        | Usage                       | Example                                     |
| -------------- | --------------------------- | ------------------------------------------- |
| `z.object()`   | Validate structured input   | `z.object({ name: z.string() })`            |
| `z.enum()`     | Restrict to literal values  | `z.enum(["light", "dark", "system"])`       |
| `z.infer<>`    | Extract TS type from schema | `type Mode = z.infer<typeof modeSchema>`    |
| `z.url()`      | Validate URL strings (v4)   | `z.url()` — replaces `z.string().url()`     |
| `z.email()`    | Validate email (v4)         | `z.email()` — replaces `z.string().email()` |
| `.optional()`  | Make field optional         | `z.string().optional()`                     |
| `.default()`   | Provide default value       | `z.number().default(0)`                     |
| `.parse()`     | Validate and throw on error | `schema.parse(data)`                        |
| `.safeParse()` | Validate without throwing   | `schema.safeParse(data)`                    |

## Common Patterns

### Enum Schema for Constrained Values

**When:** Handler accepts one of a fixed set of values.

```typescript
export const checklistFormatSchema = z.enum(["Ace", "Json", "Pdf"]);

type ChecklistFormat = z.infer<typeof checklistFormatSchema>;
// "Ace" | "Json" | "Pdf"
```

### Object Schema with Nested Types

**When:** Handler accepts structured data with multiple fields.

```typescript
export const exportFileInputSchema = z.object({
  fileId: z.string(),
  format: checklistFormatSchema,
  outputPath: z.string(),
});
```

### Top-Level Validators (v4)

**When:** Validating common string formats. Use v4 top-level validators instead of chained methods.

```typescript
// Zod v4 — preferred
z.url();
z.email();
z.uuid();

// Deprecated — avoid
z.string().url();
z.string().email();
z.string().uuid();
```

## See Also

- [patterns](references/patterns.md) — Schema design patterns, anti-patterns, oRPC integration
- [workflows](references/workflows.md) — Adding new IPC schemas, validation workflows

## Related Skills

- See the **orpc** skill for the handler → router → action IPC pattern that consumes Zod schemas
- See the **typescript** skill for type inference patterns and strict mode conventions

## Documentation Resources

> Fetch latest Zod v4 documentation with Context7.

**How to use Context7:**

1. Use `mcp__context7__resolve-library-id` to search for "zod"
2. Prefer website documentation (`/websites/`) over source code repositories
3. Query with `mcp__context7__query-docs` using the resolved library ID

**Library ID:** `/websites/zod_dev_v4`

**Recommended Queries:**

- "zod v4 object string enum validation"
- "zod v4 infer type safeParse parse"
- "zod v4 transform pipe coerce"
- "zod v4 breaking changes from v3"
