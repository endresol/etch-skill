# etch.blocks — full reference

Source: https://docs.etchwp.com/public-api/blocks

The core namespace: selects, reads, creates, mutates, deletes the blocks making up
the document. Mutations are **buffered** — call `await etch.saveAsync()` to persist.

## Reading and selection

```ts
select(blockId: string): void;
deselect(): void;
getSelectedId(): string | null;
getJson(blockId: string): PublicBlockJson;   // block + subtree, with id/parentId
getTree(): PublicBlockJson[];                // whole document as top-level blocks
find(predicate: FindBlocksPredicate): string[]; // ids of matching blocks
```

```ts
interface FindBlocksPredicate {
  type?: string;       // exact type match, e.g. "etch/text"
  class?: string;      // has this CSS class
  attribute?: string;  // has this HTML attribute set
}
```

```js
const textIds = etch.blocks.find({ type: "etch/text" });
const buttons = etch.blocks.find({ class: "btn" });
const linked  = etch.blocks.find({ attribute: "href" });
```

`getJson()` returns a discriminated union — narrow on `type` before touching
type-specific fields (full union in `types-errors.md`):

```js
const block = etch.blocks.getJson(id);
if (block.type === "etch/text") {
  console.log(block.text);
} else if (block.type === "etch/element") {
  console.log(block.tag, block.attributes, block.styles); // styles is read-only
}
```

## Mutations

```ts
create(json: EtchBlockJson, parentId?: string | null, index?: number): string; // new id
delete(blockId: string): void;                 // removes block + subtree
duplicate(blockId: string): string;            // deep copy; returns new id
move(blockId: string, newParentId: string | null, index?: number): void;
replace(blockId: string, json: EtchBlockJson): string; // returns new id
update(blockId: string, patch: BlockPatch): void;      // patch in place
copy(blockId: string): CopyObject;             // serialize block + subtree
pasteAsync(payload: CopyObject, targetId?: string | null, index?: number): Promise<string>;
```

- `create`/`duplicate`/`replace`/`pasteAsync` all return the new block's id.
- Omit the parent id (or pass `null`) to target the document root.
- Authoring is type-checked against the block JSON union (`types-errors.md`) — the
  `styles` array on reads is rejected if you try to author it.
- A parent that can't contain the child (text block under a void element like
  `img`, a block-level child under a text container) throws `WRONG_BLOCK_TYPE` for
  `create`, `move` (block stays in place), and indexed `pasteAsync`.

```js
const id = etch.blocks.create({
  type: "etch/element",
  version: 1,
  context: {},
  children: [],
  tag: "div",
  attributes: { id: "main" },
});
```

### `update()` — the editable surface only

```ts
interface BlockPatch {
  name?: string;                                  // structure-panel label
  hidden?: boolean;                                // hide/show on canvas
  attributes?: Record<string, string | undefined>; // merge; undefined removes a key
  text?: string;                                   // text blocks only
}
```

`attributes` works on HTML blocks (HTML attributes) and on component instances
(their props — see below).

### `copy()` / `pasteAsync()` — programmatic clipboard

Serializes a block + subtree, plus the global styles/loops/components it
references, into a portable `CopyObject` (plain JSON — no system clipboard
involved, so this works headlessly). `pasteAsync` re-creates and re-maps those
referenced styles/loops/components to fresh ids on insert.

```js
const payload = etch.blocks.copy(sourceId);                 // serializable
const newId = await etch.blocks.pasteAsync(payload);         // append at root
await etch.blocks.pasteAsync(payload, containerId);           // into a container
await etch.blocks.pasteAsync(payload, containerId, 2);        // as 3rd child
await etch.saveAsync();
```

Placement rules mirror `create()`:
- no `targetId` → appended at root (or at `index` there).
- `targetId` + `index` → child of target at `index`; throws `WRONG_BLOCK_TYPE` if
  the target can't contain it (an explicit index means "must be a child here").
- `targetId`, no `index` → into target if it accepts children, else immediately
  after it.
- `index` is clamped to range; negative counts from the end (`-1` = append).

## Text, naming, attributes, classes

```ts
setText(blockId: string, text: string): void;              // text blocks only
rename(blockId: string, name: string): void;                // structure-panel label
getAttribute(blockId: string, key: string): string | undefined;
setAttribute(blockId: string, key: string, value?: string): void; // undefined clears
removeAttribute(blockId: string, key: string): void;
addClass(blockId: string, className: string): void;
removeClass(blockId: string, className: string): void;
hasClass(blockId: string, className: string): boolean;
```

```js
const [id] = etch.blocks.find({ type: "etch/text" });
etch.blocks.setText(id, "Hello world");
etch.blocks.addClass(id, "lead");
etch.blocks.setAttribute(id, "data-track", "hero");
await etch.saveAsync();
```

### Special element attribute keys

```js
// etch/dynamic-image — bind to a media source
etch.blocks.setAttribute(imgBlockId, "mediaId", "{post.featured_image_id}");
etch.blocks.setAttribute(imgBlockId, "useSrcSet", "true");

// etch/svg — load and recolor an SVG
etch.blocks.setAttribute(svgBlockId, "src", "/icons/logo.svg");
etch.blocks.setAttribute(svgBlockId, "stripColors", "true");
```

### Creating a new component instance

**Not documented anywhere in the public API docs as of 2026-09-04 — reconstructed
by probing.** To place a new instance of an existing component definition, `create()`
an `etch/component` block whose **`componentId` is the numeric component id**
(`PublicComponentSummary.id` from `components.list()`/`getJson()`) — NOT the
PascalCase `key` string. `key`/`componentKey`/`target` all throw `INVALID_ARGUMENT`
("expected number, received undefined" at `componentId`) — confirmed by probing all
four field names against a live component:

```js
const compId = 1649; // numeric id, e.g. from components.list().find(c => c.key === "ResourceCard").id
const instanceId = etch.blocks.create({
  type: "etch/component", version: 1, context: {}, children: [],
  componentId: compId,
  attributes: { title: "Card title", excerpt: "...", href: "#" }, // closed schema — see below
}, parentId); // normal create() parentId/index args apply
```

`attributes` here follows the same closed-schema rule as `setAttribute` below — set
whichever declared prop keys you want at creation time instead of creating-then-
setAttribute-ing each one separately.

### Component props

An `etch/component` instance exposes its bound props as block attributes — set with
`setAttribute`/`update`, read with `getAttribute`, same as HTML attributes. Unlike
free-form HTML attributes, component props are a **closed schema**: the key must be
declared on the component definition, or it throws `INVALID_ARGUMENT` (or
`OPERATION_FAILED` if the definition hasn't loaded yet).

```js
const [cardId] = etch.blocks.find({ type: "etch/component" });
etch.blocks.setAttribute(cardId, "title", "Hello world");
etch.blocks.getAttribute(cardId, "title"); // "Hello world"
etch.blocks.update(cardId, { attributes: { title: "Hi", variant: "primary" } });
etch.blocks.setAttribute(cardId, "notAProp", "x"); // throws INVALID_ARGUMENT
```

Values are stored as-is; typed props (classes, repeater/group data) need the
component's internal format for that prop (see `styles-components.md`). This does
not enter component edit mode — that's for editing the component's internal tree.

## Component edit mode

Edits the internal block tree of a component definition (affects every instance).

```ts
enterComponentEditMode(blockId: string): void;
exitComponentEditMode(options?: { revert?: boolean }): void; // revert: discard
isInComponentEditMode(): boolean;
saveComponentEditModeAsync(): Promise<void>; // persist the component definition
```

```js
const [compId] = etch.blocks.find({ type: "etch/component" });
etch.blocks.enterComponentEditMode(compId);
const tree = etch.blocks.getTree();
etch.blocks.setText(tree[0].children[0].id, "Updated label");
await etch.blocks.saveComponentEditModeAsync(); // persist component definition
await etch.saveAsync();                          // persist the page
etch.blocks.exitComponentEditMode();
// or discard: etch.blocks.exitComponentEditMode({ revert: true });
```

**Gotcha — the id you entered edit mode with stops resolving the internal tree,
and `getTree()`'s `tree[0]` is only the instance if it happens to sit at the
document root.** Verified on runtime 1.6.1, 2026-07-04, with an instance nested a
few levels down (inside a section > container): after
`enterComponentEditMode(instanceId)`, `etch.blocks.getJson(instanceId)` (the
*original* id) kept returning `children: []` — as if still a plain, unopened
instance — for the entire duration of edit mode. The node that actually shows the
populated internal tree gets a **different, freshly-generated id each time you
enter edit mode**, unrelated to the persisted instance id. Re-run
`find({ type: "etch/component" })` (or walk `getTree()` looking for the
`etch/component` node) *after* calling `enterComponentEditMode()`, and use that id,
not the one you opened edit mode with:

```js
const [instanceId] = etch.blocks.find({ type: "etch/component" });
etch.blocks.enterComponentEditMode(instanceId);
const [editId] = etch.blocks.find({ type: "etch/component" }); // NOT instanceId — re-find after entering
const root = etch.blocks.getJson(editId).children[0];          // the component's actual root block
etch.blocks.addClass(root.id, "some-class");
await etch.blocks.saveComponentEditModeAsync();
await etch.saveAsync();
etch.blocks.exitComponentEditMode();
```

Ids on the leaf blocks *inside* that subtree (text nodes, nested elements) were
stable across repeated edit-mode entries in this test — only the top wrapping
node's id churns. Not yet confirmed whether this churn is intentional; treat it as
an observed quirk to work around (always re-find after entering), not a documented
contract.

**Second gotcha — with more than one component instance anywhere in the tree,
`find({ type: "etch/component" })` after entering edit mode returns ALL of them,
not just the one you opened; blindly taking `[0]` can silently grab the wrong
instance's internals and throw a confusing downstream error** (verified
2026-09-04: with a header instance and a footer instance both present, entering
edit mode on the footer and destructuring `const [editId] = find(...)` picked up
a node belonging to the *header* instead — `addClass()` on it then failed with
`"Block ... is not an HTML block"`, since the mismatched node's `children` didn't
have the shape the script expected). **Fix: after `find()`, `getJson()` every
candidate and filter for `componentId === <the id you're editing>`** — the
opened instance's edit-mode representation still carries its `componentId`, so
this reliably picks the right one even with several instances (of the same or
different components) on the page:

```js
etch.blocks.enterComponentEditMode(instanceId);
const candidates = etch.blocks.find({ type: "etch/component" }).map(id => etch.blocks.getJson(id));
const editJson = candidates.find(j => j.componentId === myComponentId);
if (!editJson) throw new Error("couldn't find the opened edit-mode node");
const root = editJson.children[0];
```

**Third gotcha — `exitComponentEditMode({ revert: true })` can regenerate the
outer instance's id even though the instance itself survives.** After a botched
edit-mode pass reverted, the instance block was still present in `getTree()` with
the same `componentId` and attributes, but under a brand-new id — the old id then
throws `"Block ... is not a component block"` on the next `enterComponentEditMode`
call. Re-`find()`/re-read the tree after any revert before addressing the
instance again; don't assume an id you captured before the revert still resolves.

**Fourth gotcha — a component INSTANCE block cannot take its own `addClass()`.**
Verified 2026-09-05: `etch.blocks.addClass(instanceId, "some-class")` on an
`etch/component`-type block throws `"Block ... is not an HTML block"` — instances
are a reference/pointer type, not an HTML element, until rendered. Only elements
*inside* the definition (reached via edit mode) can be classed; that class then
applies to every instance uniformly, there's no per-instance class override.
Practical effect: if different instances need different positioning/variants,
either (a) bake the shared treatment into the definition's own class if all
instances actually want the same thing (often true — don't assume you need
per-instance variance until a real case calls for it), or (b) add a component prop
the internal template reads to switch behavior (e.g. a `variant` string prop
driving a dynamic class expression on an internal element), not an instance-level
class.

**Fifth gotcha — the before/after `find()` diff (second gotcha, above) can surface
MORE than one new id when the component being edited contains a nested component
instance of its own** (e.g. a card component whose CTA is itself a shared
link component). Entering edit mode exposes the internal tree, so the nested
instance's own id becomes newly enumerable too, alongside the outer wrapper's edit
node — both look "new" relative to the before-set. Filter the new ids by
`componentId === <the id you're actually editing>` (not just "is new") to land on
the right one:

```js
const before = new Set(etch.blocks.find({ type: "etch/component" }));
etch.blocks.enterComponentEditMode(instanceId);
const after = etch.blocks.find({ type: "etch/component" });
const newIds = after.filter(id => !before.has(id));
const match = newIds.map(id => etch.blocks.getJson(id)).find(j => j.componentId === myComponentId);
```

**Nesting one component inside another's definition works structurally** —
`updateAsync({ blocks })` accepts an `etch/component` node (with `componentId` +
`attributes`) anywhere in the tree, same as any other block type, `styles: []`
required same as its siblings. **Dynamic prop passthrough to the nested instance**
(e.g. the outer component's own `{props.href}` written directly into the nested
instance's `attributes.href`, so instantiating the outer component with a real
href flows through to the inner one) was schema-accepted and read back correctly
via `getJson()`, but **not yet independently confirmed to actually interpolate
correctly on the rendered/published frontend** — verify against a real published
page before relying on it, same as any render-dependent claim in this file.

## Slots (`etch/slot-placeholder` / `etch/slot-content`)

**Not documented anywhere in the public API docs as of 2026-09-04 — verified by
probing.** Lets a component definition declare a named gap that each instance can
fill with arbitrary markup (e.g. a header component with a `nav` slot so different
pages can drop in different navigation content, without editing the component
itself).

In the **component definition**'s `blocks` (via `components.updateAsync`), mark the
gap with a bare placeholder — no `children` needed, it's a leaf:

```js
{ type: "etch/slot-placeholder", version: 1, context: {}, styles: [], children: [], slotName: "nav" }
```

On an **instance** (`etch.blocks.create({ type: "etch/component", ... })`), pass an
`etch/slot-content` block as a **direct child of the instance**, with a matching
`slotName` and the real markup as its `children` — confirmed end-to-end (`create()`
accepted it, `getJson()` read back the exact nested tree under the slot):

```js
const instanceId = etch.blocks.create({
  type: "etch/component", version: 1, context: {}, attributes: {},
  componentId: someComponentId,
  children: [
    { type: "etch/slot-content", version: 1, context: {}, styles: [], slotName: "nav", children: [
      // ...whatever markup should render in that slot...
    ] },
  ],
});
```

A component can be given multiple named slots this way; match `slotName` between
the placeholder and the fill. Rendering on the live frontend not yet independently
confirmed (only schema acceptance + editor read-back) — verify against the
published page the same as any other build.
