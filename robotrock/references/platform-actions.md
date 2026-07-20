# Platform terminal actions

Reviewers can close a task from the RobotRock inbox **without** choosing one of your defined actions. These use reserved `action.id` values on webhooks, polling, `getTask()`, and MCP `get_task`.

**Agents must always check for these ids and stop** — do not continue the workflow or call `sendToHuman` again for the same intent.

## Action IDs

| SDK constant | Value | Meaning | `data` |
|--------------|-------|---------|--------|
| `PLATFORM_MARK_DONE_ACTION_ID` | `robotrock:mark-done` | Human marked the task done | `{}` |
| `PLATFORM_REJECT_REQUEST_ACTION_ID` | `robotrock:reject-request` | Human rejected the request (bad output, loop, etc.) | `{ feedback: string }` |

These are **not** the same as a task action you defined with `id: "reject"`.

## SDK helpers

```typescript
import {
  PLATFORM_MARK_DONE_ACTION_ID,
  PLATFORM_REJECT_REQUEST_ACTION_ID,
  isPlatformTerminalAction,
  isPlatformRejectRequestAction,
  shouldStopAgentForHandledAction,
  parseHandledOutcome,
  parsePlatformRejectRequestData,
  assertNotPlatformRejectRequest,
  PlatformRejectRequestError,
} from "robotrock";
```

| Helper | Use |
|--------|-----|
| `isPlatformTerminalAction(id)` | `true` for mark-done or reject-request |
| `shouldStopAgentForHandledAction(id)` | Same — quick guard before continuing |
| `parseHandledOutcome({ actionId, data, ... })` | Discriminate platform vs task action |
| `parsePlatformRejectRequestData(data)` | Read `{ feedback }` safely |
| `assertNotPlatformRejectRequest(id, data)` | Throws `PlatformRejectRequestError` on platform reject |

## Polling

```typescript
const result = await robotrock.sendToHuman({ ... });

if (result.mode !== "handled") {
  return;
}

if (shouldStopAgentForHandledAction(result.actionId)) {
  const outcome = parseHandledOutcome({
    actionId: result.actionId,
    data: result.data,
    handledBy: result.handledBy,
    handledAt: result.handledAt,
  });
  if (outcome.kind === "reject-request") {
    console.error(outcome.data.feedback);
  }
  return; // stop agent
}

// normal task action (approve, reject, etc.)
```

## Webhooks

```typescript
const payload = await verifyRobotRockWebhook(request);

if (isPlatformRejectRequestAction(payload.action.id)) {
  const { feedback } = parsePlatformRejectRequestData(payload.action.data) ?? { feedback: "" };
  await abortAgentRun({ feedback });
  return Response.json({ ok: true, stopped: true });
}

if (isPlatformTerminalAction(payload.action.id)) {
  return Response.json({ ok: true, stopped: true });
}
```

## MCP `get_task`

After polling until `status === "handled"`:

```typescript
if (task.handled?.actionId === PLATFORM_REJECT_REQUEST_ACTION_ID) {
  const { feedback } = parsePlatformRejectRequestData(task.handled.data) ?? { feedback: "" };
  return { stopped: true, feedback };
}
if (isPlatformTerminalAction(task.handled?.actionId)) {
  return { stopped: true };
}
```

## Trigger.dev / Workflow

Wait token payloads use the same `action.id`. Branch on platform ids before treating the result as approve/reject from your task definition.
