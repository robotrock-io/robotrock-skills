# sendToHuman

Call `robotrock.sendToHuman()` from your shared client module.

## Minimal example

```typescript
import { robotrock } from "@/lib/robotrock";

const result = await robotrock.sendToHuman({
  type: "budget-approval",
  name: "Approve Q4 budget update",
  actions: [
    { id: "approve", title: "Approve" },
    { id: "reject", title: "Reject" },
  ],
});
```

## Full payload

```typescript
await robotrock.sendToHuman({
  type: "string",           // required — task category
  name: "string",           // required — display title
  description: "string",    // optional
  validUntil: new Date(Date.now() + 2 * 60 * 60 * 1000), // optional deadline
  context: {                // optional — inbox display data
    data: {},
    ui: {},
  },
  actions: [],              // required — at least one action
  idempotencyKey: "key",    // optional — safe retries
  threadId: "thread_...",   // optional — group related tasks (see task-threads.md)
  assignTo: {               // optional — inbox visibility
    users: ["alice@acme.com"],
    groups: ["finance"],
  },
});
```

`app` and `version` are on `createClient`, not per task.

Omit `threadId` to start a new thread; the server returns one on `task.threadId`. See [task-threads.md](task-threads.md).

## Platform terminal actions

After `sendToHuman` resolves (polling) or in webhook handlers, check `actionId` for `robotrock:mark-done` and `robotrock:reject-request`. **Stop the agent** — see [platform-actions.md](platform-actions.md).

## Actions

Each action requires `id` and `title`. Add JSON Schema for structured reviewer input:

```typescript
const actions = [
  {
    id: "approve",
    title: "Approve",
    schema: {
      type: "object",
      required: ["ticket"],
      properties: {
        ticket: { type: "string" },
      },
    },
    ui: {
      ticket: { "ui:title": "Change ticket", "ui:widget": "string" },
    },
  },
  {
    id: "reject",
    title: "Reject",
    schema: {
      type: "object",
      required: ["reason"],
      properties: {
        reason: { type: "string" },
      },
    },
  },
] as const;
```

Use `as const` on `actions` for TypeScript to narrow `result.data` by `actionId`.

## Assignment (`assignTo`)

Top-level field — not inside `context`:

```typescript
assignTo?: {
  users?: string[];   // tenant member emails
  groups?: string[]; // group slugs, e.g. "finance"
};
```

| Value | Visibility |
|-------|------------|
| Omitted / `{ groups: ["all"] }` | All workspace members |
| `{ groups: ["admins"] }` | Tenant admins only (virtual group from admin role) |
| `{ users: [...], groups: ["finance", ...] }` | Listed users and group members |

Invalid emails or unknown group slugs return `400`.

## Return shapes

**With webhook client:**

```typescript
{ mode: "created", task: { taskId: "task_..." } }
```

**With polling client:**

```typescript
{
  mode: "handled",
  taskId: "task_...",
  actionId: "approve",
  data: { ticket: "CHG-123" },
  handledBy: "user@example.com",
  handledAt: "2024-01-15T10:30:00Z",
}
```

## Idempotency

Pass `idempotencyKey` for safe retries on network failures. Same key + same payload returns the original response.
