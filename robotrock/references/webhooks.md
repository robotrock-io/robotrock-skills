# Webhooks

Configure `webhook` on `createClient`. It applies to every action when you call `sendToHuman`.

## Client setup

```typescript
// lib/robotrock.ts
import { createClient } from "robotrock";

export const robotrock = createClient({
  app: "my-service",
  webhook: {
    url: "https://your-app.com/api/robotrock/webhook",
    headers: {
      "x-request-id": "optional-custom-header",
    },
  },
});
```

`sendToHuman()` returns immediately with `mode: "created"`. Your app receives the result via HTTP POST.

## Webhook payload

```json
{
  "taskId": "task_123abc",
  "action": {
    "id": "approve",
    "title": "Approve",
    "data": { "notes": "Looks good!" }
  },
  "handledBy": "user@example.com",
  "handledAt": "2024-01-15T10:30:00Z",
  "handlerType": "webhook"
}
```

## Platform terminal actions

Humans can **mark as done** or **reject the request** from the inbox (not your task buttons). Webhook `action.id` values:

- `robotrock:mark-done` — `data: {}`
- `robotrock:reject-request` — `data: { feedback: string }`

**Stop your agent** when you receive either id. See [platform-actions.md](platform-actions.md).

```typescript
import {
  isPlatformTerminalAction,
  isPlatformRejectRequestAction,
  parsePlatformRejectRequestData,
} from "robotrock";

if (isPlatformRejectRequestAction(payload.action.id)) {
  const { feedback } = parsePlatformRejectRequestData(payload.action.data) ?? { feedback: "" };
  // abort workflow
}

if (isPlatformTerminalAction(payload.action.id)) {
  return NextResponse.json({ ok: true, stopped: true });
}
```

## Next.js route handler

```typescript
// app/api/robotrock/webhook/route.ts
import { NextResponse } from "next/server";
import {
  verifyRobotRockWebhook,
  RobotRockWebhookError,
  type RobotRockWebhookPayload,
} from "robotrock";

export async function POST(req: Request) {
  let payload: RobotRockWebhookPayload;
  try {
    payload = await verifyRobotRockWebhook(req);
  } catch (error) {
    if (error instanceof RobotRockWebhookError) {
      return NextResponse.json({ error: error.code }, { status: 401 });
    }
    return NextResponse.json({ error: "Unknown error" }, { status: 500 });
  }

  if (payload.action.id === "approve") {
    // run approved flow
  }

  return NextResponse.json({ ok: true });
}
```

## Verification

Always use the SDK helper:

```typescript
import { verifyRobotRockWebhook } from "robotrock";

const payload = await verifyRobotRockWebhook(request);
```

`verifyRobotRockWebhook()`:

- Validates `x-robotrock-signature`
- Reads `ROBOTROCK_WEBHOOK_SECRET` from env
- Returns typed payload including `payload.headers`
- Throws `RobotRockWebhookError` with machine-readable codes

Set `ROBOTROCK_WEBHOOK_SECRET` in your deployment environment. Create or rotate the secret in RobotRock workspace settings.

## Framework integrations set webhooks automatically

- **Trigger.dev:** `sendToHumanTask` sets the wait token URL as the webhook — no manual client webhook needed.
- **Vercel Workflow:** `sendToHumanInWorkflow` uses `createWebhook()` — no manual client webhook needed.

## No webhook?

Omit `webhook` on `createClient` and `sendToHuman()` polls until handled. See [polling.md](polling.md).
