# Block JSON shapes & `EtchApiError` codes

Sources: https://docs.etchwp.com/public-api (block type union lives inline on each
namespace's page, not a single dedicated page) and error codes encountered directly
in build sessions. **This file is a living, partial list** — it was reconstructed
2026-09-04 after discovering SKILL.md referenced this file three times without it
existing; add to it as new codes/types are actually encountered rather than
guessing ahead of need.

## Error shape

```ts
interface EtchApiError extends Error {
  code: string; // see below
}
function isEtchApiError(err: unknown): err is EtchApiError;
```

`isEtchApiError` is the safe way to narrow a caught error before reading `.code` —
don't assume every thrown error is an `EtchApiError` (a genuine JS bug in your
script throws a plain `Error`/`TypeError` with no `.code`).

```js
try {
  etch.blocks.create({ /* ... */ });
} catch (err) {
  if (isEtchApiError(err) && err.code === "WRONG_BLOCK_TYPE") {
    // handle the specific case
  } else {
    throw err; // don't swallow unknown failures
  }
}
```

## Known codes (confirmed via direct use, not exhaustive)

| Code | When it's thrown |
|---|---|
| `NOT_AVAILABLE` | `getEtch()` called when `window.etch` isn't present (builder not loaded) — not really reachable from inside a connector script, since the connector already confirmed the tab has Etch, but documented for completeness. |
| `INVALID_ARGUMENT` | Setting an undeclared component prop key via `setAttribute`/`update`; passing `etch.ai.setState()` a value outside `"idle"/"thinking"/"working"/"waiting"`. |
| `WRONG_BLOCK_TYPE` | Putting a block under a parent that can't contain it — text under a void element (`img`/`br`/`hr`), a block-level child inside a text-only container. Thrown by `create`, `move`, and indexed `pasteAsync`. |
| `OPERATION_FAILED` | Seen when acting on a component whose definition hasn't finished loading yet (e.g. setting a prop before the component data arrived) — generic "the operation couldn't complete" fallback; check the message for specifics. |
| `NOT_AVAILABLE` (runtime) | Calling a namespace the runtime doesn't back — e.g. `etch.loops`/`etch.fields` on Etch Studio; unsupported asset options. Check `etch.environment.capabilities` first. |
| `READONLY` | Writing to a derived value, e.g. `etch.content.removeFieldAsync` on a `relationshipBacklink` ("Remove the relationship this backlink mirrors"). |
| `DEPLOY_BLOCKED` | `etch.deploy.deployAsync()` refused; the message lists the blocking reasons. |
| `ASSET_NOT_FOUND` / `COLLECTION_NOT_FOUND` / `ASSET_IN_USE` | `etch.assets` calls on a missing asset/collection, or deleting an asset a page still uses. |
| `INVALID_ARGUMENT` (patch) | `etch.blocks.update()` with a key outside `name`/`hidden`/`attributes`/`text`/`target` (e.g. `conditionString`); `etch.content.updateFieldAsync` changing a field's `type`. |

## PublicBlockJson (the shape every block round-trips as)

```ts
interface PublicBlockJson {
  type: string;          // "etch/element" | "etch/text" | "etch/svg" | "etch/dynamic-image" | "etch/loop" | "etch/condition" | "etch/component" | ...
  version: number;
  context: Record<string, unknown>; // free-form metadata, e.g. { name: "Hero Section" }
  children: PublicBlockJson[];
  attributes?: Record<string, string>; // HTML attributes — current builds accept "class" here and link it; older ones dropped it (SKILL.md §3.5)
  styles: string[];      // READ-ONLY — this is what "class" actually resolves to; set via addClass()/removeClass()
  tag?: string;           // for etch/element: "section" | "div" | "h1"-"h6" | "p" | "a" | "img" | "ul" | "li" | "button" | "strong" | "details" | "summary" | ...
  text?: string;          // etch/text only
  id: string;              // present on read (getJson/getTree), not on the object you pass to create()
  parentId: string | null; // present on read
}
```

- `styles` is populated by the compiler from classes you've applied with
  `addClass()` — you cannot author it directly on `create`/`update` (rejected).
- `attributes.class` in authored JSON: dropped on older builds, linked to its
  style record on current ones (connector 0.5.x) — see SKILL.md §3.5. Don't use
  the `styles` array to judge whether a class is linked; check the render.
- Void elements (`img`, `br`, `hr`) never take `children` — see `WRONG_BLOCK_TYPE`
  above.
