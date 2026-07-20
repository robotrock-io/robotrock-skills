# Tool result display (chat UI) — OpenUI

RobotRock dashboard chat uses the **[OpenUI](https://www.openui.com) spec** for
generative tool-result UI:

- **Language:** [OpenUI Lang](https://www.openui.com/docs/openui-lang)
- **Library:** OpenUI’s first-party shadcn GenUI catalog (`shadcnChatLibrary`)
- **Runtime:** `@openuidev/react-lang` `<Renderer>` in the dashboard

**Critical:** tools return **data only**. The Eve agent **must** emit OpenUI Lang
in a ```openui fence whenever the user should *see* that data. There are **no**
JSON “✔ Tool result” collapsible bars. Without OpenUI, successful tools only
appear as a compact Working / Used N tools trail — the payload is invisible.

Component signatures: [openui-lang.md](openui-lang.md).

## Pipeline

| Layer | Role |
|-------|------|
| **Tool output** | Domain JSON + optional `replyGuidance` (`formatTool*`) |
| **Eve assistant** | **Required:** ```openui program with data inlined; prose = delta only |
| **Dashboard** | Renders OpenUI; trail for tool activity; image OpenUI fallback if needed |

## Agent rules (must follow)

1. After any tool that returns user-facing structured data (lists, tables, search
   rows, metrics, profiles, charts), emit **one** ```openui block in the same
   turn. Start with `root = Card(...)`.
2. Prefer **one** OpenUI card per turn (the primary result). Intermediate tools
   (e.g. `get_my_access` before an admin mutation) stay trail-only — no OpenUI.
3. Prose: **delta only** (permissions, policy, next steps). Do not restate rows
   already in the OpenUI card. Follow `replyGuidance` when present.
4. Do **not** dump tables/lists as markdown instead of OpenUI.
5. Do not mention OpenUI, fences, or tool metadata in user-facing prose.
6. Links: use `Button("Label", { type: "open_url", url: "https://..." })` (or
   put `url` on list rows). Host opens `http(s)` in a new tab.
7. `generate_image`: emit OpenUI `Image` / `ImageBlock` with `url` and `storageId`
   from the tool result (include `storageId` for durable reopen). Use `Carousel` for
   multiple images.

## Pattern (tools)

```typescript
import {
  formatToolListResult,
  formatToolObjectResult,
  formatToolQueryResult,
} from "robotrock/eve";

return formatToolListResult("members", rows, {
  replyGuidance: "Do not restate names or emails from the list.",
});
```

Tools never return layouts. Optional `replyGuidance` steers narration only.

## Charts and images

- Charts: `toolChartResultSchema` + `formatToolChartResult` (wide series) → agent
  maps into OpenUI `BarChart` / `LineChart` / etc.
- Images: `generate_image` → OpenUI `Image` / `ImageBlock` (`url` + `storageId`).

## Linkable list items

Set **`url`** on each row when a navigable target exists.

## Third-party agents

Same contract: `formatTool*` for data; assistant emits ```openui for anything the
user should see. Do not ship React or register components. Follow
[openui-lang.md](openui-lang.md).
