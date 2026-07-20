# OpenTelemetry trace recording (Trigger.dev / Vercel Workflow)

RobotRock does **not** ingest OTLP or store OpenTelemetry snapshots. Human feedback analysis uses **agent version** and task context only.

For durable integrations, you can still record human-handle events on **your agent's active OTel trace**:

```bash
bun add robotrock @opentelemetry/api
export ROBOTROCK_OTEL_RECORD_HANDLED=true
export AGENT_VERSION=1.2.0
```

## Trigger.dev + Vercel Workflow

```typescript
import { sendToHumanTask } from "robotrock/trigger";
// or: sendToHumanInWorkflow from "robotrock/workflow"

await sendToHumanTask.triggerAndWait({
  type: "deploy-approval",
  name: "Approve production deploy",
  actions: [
    { id: "approve", title: "Approve" },
    { id: "reject", title: "Reject" },
  ],
  recordOtel: true, // or rely on ROBOTROCK_OTEL_RECORD_HANDLED
});
```

When enabled, the SDK:

1. Starts child span `robotrock.wait_for_human`
2. On human handle, sets `robotrock.action.id`, `robotrock.handled_by`, `robotrock.human_wait_ms`, and event `robotrock.task_handled`
3. On timeout, ends the wait span with `robotrock.outcome=timeout`

Manual recording (any integration):

```typescript
import {
  captureRobotRockOtelHandle,
  startRobotRockHumanWaitSpan,
  endRobotRockHumanWaitSpan,
} from "robotrock";
```

Action feedback (`action.data`) is **not** written to spans unless `otelIncludeActionData: true`.

## Agent version for feedback analysis

Set the agent release on the client (not OTel):

```typescript
export const robotrock = createClient({
  app: "my-service",
  version: process.env.AGENT_VERSION,
});
```

RobotRock stores `agent.version` for Statistics and weekly feedback analysis. See `get_feedback_analysis` (MCP) before improving agent code.
