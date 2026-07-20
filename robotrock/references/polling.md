# Polling

When no client `webhook` is configured, `sendToHuman()` **blocks and polls** until a human handles the task.

Use for: scripts, CLIs, long-lived servers, API routes with generous timeouts.

Avoid for: short serverless handlers — use Trigger.dev or Vercel Workflow instead.

## Setup

```typescript
// lib/robotrock.ts
import { createClient } from "robotrock";

export const robotrock = createClient({
  app: "my-service",
  polling: {
    intervalMs: 2_000,       // default: 2000
    timeoutMs: 10 * 60_000,  // default: 86400000 (24h)
  },
});
```

Omit `polling` to use defaults.

## Behavior

| Client config | `sendToHuman()` behavior |
|---------------|-------------------------|
| `webhook` set | Returns immediately (`mode: "created"`). Result via HTTP callback. |
| No `webhook` | Blocks, polls `getTask` until handled (`mode: "handled"`). |

## Task deadline (`validUntil`)

Polling stops at the **earlier** of:

1. `polling.timeoutMs` on the client
2. Task `validUntil`

```typescript
const result = await robotrock.sendToHuman({
  type: "approval",
  name: "Approve before standup",
  validUntil: new Date(Date.now() + 2 * 60 * 60 * 1000),
  actions: [
    { id: "approve", title: "Approve" },
    { id: "reject", title: "Reject" },
  ],
});
```

## Errors

```typescript
import { TaskExpiredError, TaskTimeoutError } from "robotrock";

try {
  const result = await robotrock.sendToHuman({ ... });
} catch (error) {
  if (error instanceof TaskExpiredError) {
    // task validUntil passed before anyone acted
  } else if (error instanceof TaskTimeoutError) {
    // exceeded polling.timeoutMs while task still open
  } else {
    throw error;
  }
}
```

## Typed results

With `as const` actions, TypeScript narrows `result.data` by `actionId`:

```typescript
if (result.mode === "handled" && result.actionId === "approve") {
  console.log(result.data.ticket); // typed from schema
}
```

## Platform terminal actions

If `result.actionId` is `robotrock:mark-done` or `robotrock:reject-request`, **stop the agent** — see [platform-actions.md](platform-actions.md).

```typescript
import { shouldStopAgentForHandledAction } from "robotrock";

if (result.mode === "handled" && shouldStopAgentForHandledAction(result.actionId)) {
  return;
}
```
