# Context widgets (inbox task UI)

Use `context.ui` on `sendToHuman` or `create_robotrock_task` to control how reviewers see `context.data` in the task detail view.

This catalog is for **inbox task context only**. Chat tool-result cards use OpenUI Lang from the Eve agent — see [tool-result-display.md](tool-result-display.md).

## Resolution order

1. Explicit `ui:widget` on the field (when registered)
2. Type-based default (see [task-context.md](task-context.md))
3. Fallback to `string`

Set `ui:title`, `ui:description`, and `ui:placeholder` on any field regardless of widget.

## When to set `ui:widget`

| Situation | Action |
|-----------|--------|
| Plain string, number, boolean, nested object | Usually omit — auto-selection works |
| Array of primitives | Auto → `list`; set explicitly only for clarity |
| Array of objects | Auto → `table`; nest `items` ui for column widgets |
| Money amounts | **Set** `currency` + `ui:options.currency` |
| ISO timestamps | **Set** `date` |
| URLs (clickable) | **Set** `link` |
| Image URLs | **Set** `image` |
| File download lists | **Set** `attachments` |
| Long formatted text | **Set** `markdown` |
| Code snippets | **Set** `code` + `ui:options.language` |
| Side-by-side options | **Set** `compare` |
| RobotRock entities (member, group, task) | **Set** the matching tenant widget |

## Widget catalog

### Basics (auto-selected when omitted)

| Widget | Data type | Notes |
|--------|-----------|-------|
| `string` | string | Default for strings |
| `number` | number | Default for numbers |
| `boolean` | boolean | Default for booleans |
| `object` | object | Nested fields; per-field `ui` under the object key |
| `list` | array of primitives | e.g. tags, group slugs as strings |
| `table` | array of objects | Use nested `items` for column-level widgets |

### Formatting (set explicitly)

| Widget | Data shape | `ui:options` |
|--------|------------|--------------|
| `currency` | number (amount in major units) | `currency` — ISO code (`USD`, `EUR`, `GBP`) |
| `date` | ISO 8601 string | none |
| `link` | URL string | none |
| `image` | Image URL string | none |
| `attachments` | `{ name, url }[]` | none |
| `markdown` | Markdown string | none |
| `code` | Source string | `language` — Shiki id (`typescript`, `json`, `bash`, `diff`) |
| `compare` | object array (options to compare) | none; nest `items` for fields like `image` |

### RobotRock-specific (set explicitly)

| Widget | Data shape | When to use |
|--------|------------|-------------|
| `productInfo` | `{ id?, name?, price?, currency?, inStock? }` | Product rows inside a `list` |
| `tenantMember` | `{ name?, email?, role?, membershipKind? }` | A workspace member |
| `tenantGroup` | `{ name?, slug?, description?, memberCount?, members? }` | A workspace group |
| `tenantTask` | `{ id?, name?, status?, type?, description?, validUntil?, createdAt? }` | Link card to another inbox task |
| `access` | `{ role?, workspace?, tenantAdmin?, groups?, manage* flags, email? }` | Capability summary (usually tool-authored) |

## Examples

### Refund approval (common agent task)

```typescript
context: {
  data: {
    chargeId: "ch_abc123",
    amount: 150000,
    currency: "USD",
    reason: "Customer duplicate charge",
    refundedAt: "2026-06-01T14:30:00.000Z",
  },
  ui: {
    chargeId: { "ui:title": "Charge ID" },
    amount: {
      "ui:title": "Amount",
      "ui:widget": "currency",
      "ui:options": { currency: "USD" },
    },
    reason: { "ui:title": "Reason" },
    refundedAt: { "ui:title": "Requested at", "ui:widget": "date" },
  },
}
```

### Table with currency columns

```typescript
context: {
  data: {
    lineItems: [{ label: "Tools", amount: 5000 }],
  },
  ui: {
    lineItems: {
      "ui:widget": "table",
      items: {
        amount: {
          "ui:widget": "currency",
          "ui:options": { currency: "USD" },
        },
      },
    },
  },
}
```

### Product list

```typescript
context: {
  data: {
    products: [{ id: "p1", name: "Widget", price: 29.99, currency: "USD", inStock: true }],
  },
  ui: {
    products: {
      "ui:widget": "list",
      items: { "ui:widget": "productInfo" },
    },
  },
}
```

### Assignee group reference

```typescript
context: {
  data: {
    assigneeGroup: { name: "Finance", slug: "finance", memberCount: 4 },
  },
  ui: {
    assigneeGroup: { "ui:title": "Routed to", "ui:widget": "tenantGroup" },
  },
}
```

## Nested `ui` keys

For `list`, `table`, `object`, and `compare`, put per-item or per-column hints under `items`:

```typescript
ui: {
  members: {
    "ui:widget": "list",
    items: {
      email: { "ui:widget": "link" },
    },
  },
}
```

## Action form widgets

Action `schema` / `ui` on approve/reject buttons use the same JSON Forms conventions. For 2–6 discrete choices in chat (not inbox), use `request_action_input` with `ui:widget: "radio"` — see [vercel-ai.md](vercel-ai.md).

## Related

- [task-context.md](task-context.md) — `context` shape and auto-selection
- [send-to-human.md](send-to-human.md) — full task payload
- [tool-result-display.md](tool-result-display.md) — chat OpenUI tool cards (not pickable by the agent)
