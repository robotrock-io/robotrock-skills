# Threads

Group related tasks together with a `threadId`. Tasks that share a `threadId` are grouped in the inbox (one row with a count badge) and stacked newest-first in the task detail.

## How it works

1. Create a task. If you omit `threadId`, the server generates one (`thread_<uuid>`) and returns it on `task.threadId`.
2. Reuse that `threadId` on later `sendToHuman()` calls to group them.
3. The inbox collapses tasks that share a `threadId` into a single row with a task-count badge; opening it stacks every task newest-on-top.

Threads are tenant-scoped — any task created with your API keys can share a `threadId` regardless of `app`.

## Start a thread

Omit `threadId`, then read it back from the response:

```typescript
import { robotrock } from "@/lib/robotrock";

const first = await robotrock.sendToHuman({
  type: "deploy-approval",
  name: "Approve staging deploy",
  actions: [{ id: "approve", title: "Approve" }],
});

const threadId = first.task.threadId; // reuse this
```

## Add to a thread

```typescript
await robotrock.sendToHuman({
  type: "deploy-approval",
  name: "Confirm production rollout",
  threadId, // groups with the first task
  actions: [{ id: "approve", title: "Approve" }],
});
```

Or supply your own stable id (e.g. an internal workflow id):

```typescript
await robotrock.sendToHuman({
  type: "deploy-approval",
  name: "Approve staging deploy",
  threadId: `deploy_${deploymentId}`,
  actions: [{ id: "approve", title: "Approve" }],
});
```

## Field reference

`threadId` is a **top-level task field** — like `assignTo`, it is not inside `context`.

| Field | Type | Notes |
|-------|------|-------|
| `threadId` (request) | `string` (optional) | Omit to start a new thread; pass an existing value to group. |
| `threadId` (response) | `string` | Always returned on `task.threadId`, even for single-task threads. |

`getTask` also returns `threadId`, so you can confirm which thread a task belongs to.

## Notes

- A thread with one task looks like a normal ungrouped task.
- The inbox badge counts tasks visible in the current filter; the stacked detail shows the full thread.
- Each task in a stacked thread keeps its own actions and handled state.
