# Trigger.dev integration

Use `robotrock/trigger` for **durable waits** inside Trigger.dev workflows.

```bash
npm install robotrock @trigger.dev/sdk
```

## Register SDK tasks

Re-export once from your `trigger/` directory:

```typescript
// trigger/robotrock.ts
export { sendToHumanTask, approveByHumanTask } from "robotrock/trigger";
```

## Environment variables

Inside the Trigger worker:

- `ROBOTROCK_API_KEY` (required)
- `ROBOTROCK_BASE_URL` or `ROBOTROCK_API_URL` (optional)
- `ROBOTROCK_APP` (optional default inbox bucket)

## Why not `client.sendToHuman()` polling?

- `sendToHuman()` polling blocks the worker and does not checkpoint the wait
- `sendToHumanTask` creates a wait token, sets it as the webhook URL, and suspends durably

## `sendToHumanTask` (full control)

```typescript
import { task } from "@trigger.dev/sdk";
import { sendToHumanTask } from "./robotrock";

export const deploymentTask = task({
  id: "deployment",
  run: async (payload: { version: string }) => {
    const actions = [
      {
        id: "approve",
        title: "Deploy now",
        schema: {
          type: "object",
          required: ["changeTicket"],
          properties: { changeTicket: { type: "string" } },
        },
      },
      {
        id: "reject",
        title: "Block deploy",
        schema: {
          type: "object",
          required: ["reason"],
          properties: { reason: { type: "string" } },
        },
      },
    ] as const;

    const waitResult = await sendToHumanTask.triggerAndWait({
      type: "deployment",
      name: `Deploy ${payload.version}`,
      actions,
    });

    if (!waitResult.ok) {
      throw waitResult.error;
    }

    const result = waitResult.output;
    return result.actionId === "approve"
      ? { allowed: true, ticket: result.data.changeTicket }
      : { allowed: false, reason: result.data.reason };
  },
});
```

## `approveByHumanTask` (yes/no gate)

```typescript
import { task } from "@trigger.dev/sdk";
import { approveByHumanTask } from "./robotrock";

export const releaseGate = task({
  id: "release-gate",
  run: async (payload: { releaseId: string }) => {
    const waitResult = await approveByHumanTask.triggerAndWait({
      type: "release-gate",
      name: `Approve release ${payload.releaseId}`,
    });

    if (!waitResult.ok) {
      throw waitResult.error;
    }

    return waitResult.output.actionId === "approve";
  },
});
```

Fixed actions: `approve` / `decline`.

## OpenTelemetry

Set `ROBOTROCK_OTEL_RECORD_HANDLED=true` or pass `recordOtel: true` on the task payload. The SDK records human decisions on the Trigger run trace (`robotrock.wait_for_human` span, `robotrock.task_handled` event) and auto-fills `agent` telemetry at task create unless you pass `agent` explicitly.

See [opentelemetry.md](opentelemetry.md).

## Wait duration

Wait token timeout follows `validUntil` on the task. Default: **one week** when `validUntil` is omitted.

## Critical rules

1. Always check `waitResult.ok` before `waitResult.output`
2. Never use `Promise.all` with `triggerAndWait()` or `wait.*`
3. Export tasks from files in your `trigger/` directory
