# Vercel AI SDK integration

Use `robotrock/ai` when building agents with the Vercel AI SDK (`generateText`, `streamText`, `ToolLoopAgent`).

```bash
npm install robotrock ai
```

`ai` is a **peer dependency** of `robotrock/ai`.

## Two patterns

1. **Callable tools** — model calls `approveByHuman` or `sendToHuman` when it needs a person
2. **Tool approval bridge** — dangerous tools (e.g. `deleteFile`) pause until someone approves in the inbox

## Client setup (polling mode)

```typescript
// lib/robotrock.ts
import { createClient } from "robotrock";

export const robotrock = createClient({
  app: "my-agent",
  polling: {
    intervalMs: 2_000,
    timeoutMs: 30 * 60_000,
  },
});
```

Tools block in `execute` until a human handles the task.

## Execution modes

| Mode | Where | How it waits |
|------|-------|--------------|
| `polling` (default) | Scripts, long-lived servers | `client.sendToHuman()` polls |
| `mode: "trigger"` | Trigger.dev workers | Wait tokens via `sendToHumanTask` |
| `mode: "workflow"` | Vercel Workflow / DurableAgent | `createWebhook()` suspends |

Prefer durable modes in production workers — polling does not checkpoint the wait.

## Callable tools

### `approveByHumanTool`

```typescript
import { generateText, stepCountIs } from "ai";
import { robotrock } from "@/lib/robotrock";
import { approveByHumanTool } from "robotrock/ai";

const result = await generateText({
  model: "anthropic/claude-sonnet-4",
  tools: {
    approveByHuman: approveByHumanTool(robotrock, { defaultType: "release-gate" }),
  },
  stopWhen: stepCountIs(10),
  prompt: "Draft a production rollout plan and get human approval before finalizing.",
});
```

Tool result shape returned to the model:

```typescript
{
  taskId: string;
  actionId: string;
  data: unknown;
  handledBy?: string;
  handledAt: string;
  approved?: boolean;
}
```

### `createSendToHumanTool`

Define actions at factory time — the model only fills presentation fields:

```typescript
import { createSendToHumanTool } from "robotrock/ai";

const askHuman = createSendToHumanTool(robotrock, {
  defaultType: "ai-agent-input",
  actions: [
    {
      id: "answer-questions",
      title: "Answer questions",
      schema: {
        type: "object",
        required: ["priority"],
        properties: {
          priority: { type: "string", enum: ["low", "high"] },
        },
      },
    },
  ] as const,
});
```

### `createSendUpdateTool`

Lets the agent post progress updates to the thread it is working on. **Fire-and-forget** — no human wait. The thread is resolved from the model-supplied `threadId`, falling back to a session `threadId` in the options; provide at least one.

```typescript
import { createSendUpdateTool } from "robotrock/ai";

// Polling client + session thread fallback (model may override with its own threadId)
const sendUpdate = createSendUpdateTool(robotrock, { threadId: "thread_123" });

// Durable modes: pass a context object instead of a client
const sendUpdateTrigger = createSendUpdateTool({ mode: "trigger", app: "my-agent" });
const sendUpdateWorkflow = createSendUpdateTool({ mode: "workflow", app: "my-agent" });
```

Tool input: `{ threadId?, message, status? }`. `status` is one of `info | queued | running | waiting | succeeded | failed | cancelled` (default `info`).

Auto-thread tasks and updates with `createRobotRockAiTools({ ..., threadId })` — the shared `threadId` is applied to both `sendToHuman` (created tasks) and `sendUpdate` (updates):

```typescript
const tools = createRobotRockAiTools({ client: robotrock, threadId: "session_42" });

const result = await generateText({
  model: "anthropic/claude-sonnet-4",
  tools: {
    sendToHuman: tools.sendToHuman({ actions: [{ id: "ack", title: "Acknowledge" }] as const }),
    sendUpdate: tools.sendUpdate(),
  },
  prompt: "Work the task and post progress updates as you go.",
});
```

## Trigger.dev mode

Register SDK tasks (see [trigger.md](trigger.md)):

```typescript
// trigger/robotrock.ts
export { sendToHumanTask, approveByHumanTask } from "robotrock/trigger";
```

```typescript
import { approveByHumanTool } from "robotrock/ai/trigger";

const approveByHuman = approveByHumanTool({
  mode: "trigger",
  app: "my-agent",
});
```

## Workflow mode

```typescript
import { approveByHumanTool } from "robotrock/ai/workflow";

const approveByHuman = approveByHumanTool({
  mode: "workflow",
  app: "my-agent",
});
```

## Tool approval bridge (AI SDK 7+)

Pause dangerous tool execution until a human approves in the inbox:

```typescript
import { generateText, tool } from "ai";
import { z } from "zod";
import {
  createRobotRockToolApproval,
  runWithRobotRockApprovals,
} from "robotrock/ai";

const deleteFile = tool({
  description: "Delete a file path",
  inputSchema: z.object({ path: z.string() }),
  execute: async ({ path }) => {
    await removeFile(path);
    return { ok: true };
  },
});

const toolApproval = createRobotRockToolApproval({
  tools: ["deleteFile"],
});

const result = await runWithRobotRockApprovals({
  client: robotrock,
  maxRounds: 20,
  generate: (messages) =>
    generateText({
      model: "anthropic/claude-sonnet-4",
      tools: { deleteFile },
      toolApproval,
      messages,
      prompt: "Remove old logs in /tmp",
    }),
});
```

In Trigger.dev workers, pass context instead of a client:

```typescript
await runWithRobotRockApprovals({
  context: { mode: "trigger", app: "my-agent" },
  generate: (messages) => generateText({ model, tools, toolApproval, messages, prompt }),
});
```

## Where to run

| Environment | Recommended mode |
|-------------|------------------|
| Trigger.dev worker | `mode: "trigger"` |
| Vercel Workflow / DurableAgent | `mode: "workflow"` |
| Long-running Node | `polling` |
| Vercel serverless (< 60s) | **Avoid polling** — use Trigger.dev or Workflow |

Run the approval bridge on the **server** (API route or worker), not in client-side React.
