# Updates

Send short status updates (1-2 sentences) to a thread. The newest update shows in a status bar at the top of the inbox task detail, with an icon and color from an optional `status`. Every update is logged and viewable in an expandable history.

Updates are **thread-scoped** — they attach to a `threadId`, not a single task.

## Send a standalone update

`client.sendUpdate({ threadId, message, status? })` logs an update against an existing thread and returns it.

```typescript
import { robotrock } from "@/lib/robotrock";

const update = await robotrock.sendUpdate({
  threadId: "thread_123", // from task.threadId
  message: "Deployment started, running smoke tests.",
  status: "running",
});

// update: { id, threadId, message, status, source, createdAt }
```

The thread must already have at least one task in your tenant, otherwise the API returns `404`.

## Send an update at task creation

Pass `update` to `sendToHuman` (or `createTask`). It logs an initial update against the new task's thread.

```typescript
await robotrock.sendToHuman({
  type: "deploy-approval",
  name: "Approve production rollout",
  threadId: `deploy_${deploymentId}`,
  update: { message: "Build finished, awaiting approval.", status: "waiting" },
  actions: [{ id: "approve", title: "Approve" }],
});
```

`update` is a **top-level field** — like `threadId` and `assignTo`, it is not inside `context`.

## Status values

`status` is optional and defaults to `info`:

| Status | Meaning |
|--------|---------|
| `info` | Neutral informational message (default) |
| `queued` | Queued, not started |
| `running` | In progress |
| `waiting` | Blocked / waiting on input |
| `succeeded` | Completed successfully |
| `failed` | Errored / failed |
| `cancelled` | Aborted / cancelled |

## Field reference

| Field | Type | Notes |
|-------|------|-------|
| `threadId` | `string` | Required for `sendUpdate`. |
| `message` | `string` | 1-2 sentences. Max 500 characters. |
| `status` | enum (optional) | One of the values above. Defaults to `info`. |

## Typical pattern

Report progress against the same thread an agent created:

```typescript
const { task } = await robotrock.createTask({
  type: "job",
  name: "Nightly export",
  actions: [{ id: "ack", title: "Acknowledge" }],
  update: { message: "Export queued.", status: "queued" },
});

const threadId = task.threadId;

await robotrock.sendUpdate({ threadId, message: "Export running.", status: "running" });
// ... later
await robotrock.sendUpdate({ threadId, message: "Export complete.", status: "succeeded" });
```

## Integrations

`sendUpdate` is fire-and-forget (no human wait), so each integration wraps it differently than `sendToHuman`.

### Vercel Workflow — `sendUpdateInWorkflow`

Network calls inside a `"use workflow"` function must run in a step. `sendUpdateInWorkflow()` is a durable, retried step. It returns immediately (does not suspend).

```typescript
import { sendUpdateInWorkflow } from "robotrock/workflow";

export async function deploy(version: string) {
  "use workflow";
  const threadId = `deploy_${version}`;
  await sendUpdateInWorkflow({ threadId, message: "Build started.", status: "running" });
  // ... sendToHumanInWorkflow(...) etc.
}
```

### Vercel AI SDK — `createSendUpdateTool`

Let an agent post progress to the thread it works on. The thread is resolved from the tool input `threadId`, falling back to a session `threadId` in the options — pass at least one.

```typescript
import { createSendUpdateTool } from "robotrock/ai";

// Polling client + session thread fallback
const sendUpdate = createSendUpdateTool(robotrock, { threadId: "thread_123" });

// Durable modes: pass a context instead of a client
const sendUpdateTrigger = createSendUpdateTool({ mode: "trigger", app: "my-agent" });
const sendUpdateWorkflow = createSendUpdateTool({ mode: "workflow", app: "my-agent" });
```

Auto-thread tasks and updates by giving both tools a shared session thread:

```typescript
import { createRobotRockAiTools } from "robotrock/ai";

const tools = createRobotRockAiTools({ client: robotrock, threadId: "session_42" });
// tools.sendToHuman({ actions }) creates tasks on session_42
// tools.sendUpdate() posts updates to session_42 (model may override with its own threadId)
```

### Trigger.dev

No wrapper needed — `sendUpdate` returns immediately (no wait token). Call the SDK directly inside any task:

```typescript
import { createClient } from "robotrock";

const robotrock = createClient({ apiKey: process.env.ROBOTROCK_API_KEY! });
await robotrock.sendUpdate({ threadId, message: "Job running.", status: "running" });
```
