# etch.styles, etch.stylesheets, etch.components — full reference

Sources: https://docs.etchwp.com/public-api/styles ,
https://docs.etchwp.com/public-api/stylesheets ,
https://docs.etchwp.com/public-api/components

## etch.styles — CSS rules + `:root` variables

**Buffered** — persist with `await etch.saveAsync()`.

**This is the default way to style pages** — one dedicated class per element,
applied with `addClass(blockId, "literal-class-name")`. Verified end-to-end
(canvas + published frontend) on runtime 1.6.1 (API 0.x), 2026-07-04.

> **The one gotcha:** `create()` returns an *opaque style id* (e.g. `"6jx1uil"`),
> not the class name — pass the literal class-name string to `addClass()`, never
> the returned id (passing the id produces a sanitized-id class with no matching
> CSS). See SKILL.md §3.5 for the full writeup and a re-probe script. A prior
> report claimed this pattern doesn't compile on Etch 1.5.3, but that's unverified
> and likely a build-process mistake in that session rather than a real
> version-gated limitation — don't reach for `etch.stylesheets` on the strength of
> that report alone; re-probe and check your own script first.
>
> **Responsive breakpoints belong nested inside that same class's `css`** — a
> selector can't itself be `@media {...}` (throws `Invalid style selector`), but
> writing the `@media` rule inside the `css` string for a normal class selector
> compiles correctly (confirmed on 1.6.1, editor + published frontend). Do not
> create a stylesheet to work around this — see SKILL.md §3.5.

```ts
list(filter?: StyleListFilter): StyleSummary[];
create(selector: string, css?: string): string;   // returns id
update(styleId: string, patch: StylePatch): void;
delete(styleId: string): void;
```

```ts
interface StyleSummary {
  id: string;           // pass this to blocks.addClass() etc.
  selector: string;      // e.g. ".lead"
  type: StyleSelectorType | undefined;
  collection: string;    // internal; always "default" for the public API
  css: string;
}
interface StyleListFilter { type?: StyleSelectorType; }
interface StylePatch { selector?: string; css?: string; }
type StyleSelectorType = "class" | "id" | "tag" | "element" | "attribute" | "custom";
```

```js
const styleId = etch.styles.create(".lead", "font-size: 1.25rem;");
const classStyles = etch.styles.list({ type: "class" });
const existing = etch.styles.list().find(s => s.selector === ".lead");
etch.styles.update(styleId, { css: "font-size: 1.5rem; font-weight: 600;" });
etch.styles.delete(styleId);
```

`create()`'s return value (`styleId` above) is only for later `update()`/`delete()`
calls on that same style record — **do not** pass it to `etch.blocks.addClass()`.
`addClass()` takes the literal class-name string instead (`"lead"`, not `styleId`)
— see the gotcha callout above.

### CSS variables

Optional `collection` arg for multi-collection `:root` setups; omit for default.

```ts
listVariables(collection?: string): Record<string, string>;
getVariable(name: string, collection?: string): string | undefined;
setVariable(name: string, value: string, collection?: string): void;
removeVariable(name: string, collection?: string): void;
```

```js
etch.styles.setVariable("--brand", "#0af");
etch.styles.getVariable("--brand");     // "#0af"
etch.styles.listVariables();            // { "--brand": "#0af", ... }
etch.styles.removeVariable("--brand");
// named collection:
etch.styles.setVariable("--brand", "#0af", "theme-a");
```

## etch.stylesheets — global stylesheets + `@custom-media`

**Not for element or page styling — use `etch.styles.create()` + `addClass()`
instead (see above and SKILL.md §3.5), including for responsive breakpoints.**
This namespace is for genuinely global concerns the class-per-element pattern
doesn't cover: `@custom-media` definitions, or a site-wide reset/utility sheet the
user has explicitly asked for outside of page content. Reaching for it to style a
page element or to work around a media-query error is the wrong tool, not a
fallback.

**Persists immediately** — every mutating method is `*Async`; no `saveAsync()`.

```ts
list(): StylesheetSummary[];
get(stylesheetId: string): StylesheetSummary;
createAsync(input: StylesheetInput): Promise<string>;
updateAsync(stylesheetId: string, patch: StylesheetPatch): Promise<void>;
appendAsync(stylesheetId: string, css: string): Promise<void>; // inserts a newline
deleteAsync(stylesheetId: string): Promise<void>;
```

```ts
interface StylesheetSummary { id: string; name: string; css: string; type: StylesheetType; }
interface StylesheetInput { name: string; css: string; type?: StylesheetType; } // defaults "default"
interface StylesheetPatch { name?: string; css?: string; type?: StylesheetType; }
type StylesheetType = "default" | "@custom-media";
```

```js
const id = await etch.stylesheets.createAsync({ name: "Utilities", css: ".u-hidden { display: none; }" });
await etch.stylesheets.appendAsync(id, ".u-flex { display: flex; }");
await etch.stylesheets.updateAsync(id, { name: "Helpers" });
const sheet = etch.stylesheets.get(id);
await etch.stylesheets.deleteAsync(id);
```

### Custom media

```ts
listCustomMedia(): Record<string, string>;
addCustomMediaAsync(name: string, query: string): Promise<void>; // add or look up
```

```js
await etch.stylesheets.addCustomMediaAsync("--md", "(min-width: 768px)");
etch.stylesheets.listCustomMedia(); // { "--md": "(min-width: 768px)", ... }
```

## etch.components — reusable component definitions

**Persists immediately** — `createAsync`/`updateAsync`/`deleteAsync` don't need
`saveAsync()`. To edit a component's internal block tree, use component edit mode
(see `blocks.md`) — this namespace only manages the definition's metadata/props.

**Gotcha — `updateAsync({ blocks })` silently drops `class` from an element's
`attributes`.** Observed on runtime 1.6.1, 2026-07-04: authoring a component's tree
directly via `updateAsync(componentId, { blocks: [...] })` with elements carrying
`attributes: { class: "some-class" }` persists every other attribute correctly
(`href`, etc. survive) but the `class` key vanishes — `components.getJson()`
afterward shows `attributes: {}` on that element and no compiled CSS anywhere.
Not yet confirmed whether this is intended (e.g. component-tree authoring expects
classes to arrive only through the `styles` linkage / edit-mode `addClass()`, never
as a raw attribute) or a bug — flagged to the Etch team, treat as unconfirmed until
they weigh in. Also note this endpoint's validator *requires* `styles: []` present
on every block object (unlike `blocks.create()`, which rejects an authored `styles`
key outright) — pass `styles: []` on every node or the whole call throws
`Invalid block JSON`.

**Working pattern:** author the tree via `updateAsync({ blocks })` for structure/
text/conditions, then apply any classes in a second pass through component edit
mode with `addClass()` (the normal per-element pattern from `etch.styles` above,
which does work correctly here — verified end-to-end: canvas, `components.getJson()`,
and the published frontend all showed the class and matching CSS):

```js
// 1) structure, no classes yet
const componentId = await etch.components.createAsync("Hero Content");
await etch.components.updateAsync(componentId, {
  properties: [ /* ... */ ],
  blocks: [ /* etch/element / etch/text / etch/condition tree, no `class` attrs */ ],
});

// 2) classes via edit mode — see blocks.md "Component edit mode" for the
//    find-after-enter gotcha (the instance id you already have does NOT
//    resolve the internal tree during edit mode; you need a fresh lookup)
const [instanceId] = etch.blocks.find({ type: "etch/component" });
etch.blocks.enterComponentEditMode(instanceId);
const [editId] = etch.blocks.find({ type: "etch/component" }); // re-find, not instanceId
const root = etch.blocks.getJson(editId).children[0];
etch.blocks.addClass(root.id, "hero__content");
// ...addClass() on any other nested node ids from that same subtree...
await etch.blocks.saveComponentEditModeAsync();
await etch.saveAsync();
etch.blocks.exitComponentEditMode();
```

### Conditional content — `etch/condition` blocks

To let a component prop show/hide part of its tree (e.g. an optional CTA or lede),
wrap that part in an `etch/condition` block rather than trying to bind `hidden` on
the element itself:

```ts
interface EtchConditionBlockJson {
  type: "etch/condition";
  version: 1; context: {}; styles: []; // styles: [] required, see gotcha above
  children: EtchBlockJson[];           // rendered only if conditionString is truthy
  conditionString: string;             // bare expression, no {#if}/{/if} wrapper
}
```

```js
{
  type: "etch/condition", version: 1, context: {}, styles: [], children: [
    { type: "etch/element", version: 1, context: {}, styles: [], tag: "p",
      attributes: {}, children: [
        { type: "etch/text", version: 1, context: {}, styles: [], children: [], text: "{props.lede}" },
      ] },
  ],
  conditionString: "props.showLede", // or "!props.showLede" to invert
}
```

Bind the gating prop as a `{ primitive: "boolean" }` component property, and on
each component **instance** set its value as the literal string `"true"`/`"false"`
(the instance's `attributes` map is `Record<string, string>`, same as any other
block attribute — there is no separate boolean encoding). Verified end-to-end:
flipping an instance's `showCta`/`showLede` attribute between `"true"`/`"false"`
and re-`saveAsync()`-ing actually added/removed the element from the published
frontend output, not just the buffered editor state.

```ts
list(): PublicComponentSummary[];               // no block trees
getJson(componentId: number): PublicComponentJson; // with block tree
createAsync(name: string): Promise<number>;      // empty; returns numeric id
updateAsync(componentId: number, patch: ComponentPatch): Promise<void>;
deleteAsync(componentId: number): Promise<void>;
```

```ts
interface PublicComponentSummary {
  id: number; name: string;
  key: string;              // PascalCase reference key, e.g. "Card"
  description?: string;
  properties: ComponentProperty[];
}
interface PublicComponentJson extends PublicComponentSummary {
  blocks: PublicBlockJson[];
}
interface ComponentPatch {
  name?: string;
  key?: string;              // auto-converted to PascalCase
  description?: string;
  properties?: ComponentProperty[]; // replacement, not merge
  blocks?: EtchBlockJson[];         // replacement, not merge
}
```

```js
const all = etch.components.list();
const card = all.find(c => c.key === "Card");
const full = etch.components.getJson(card.id);

const id = await etch.components.createAsync("Callout");
await etch.components.updateAsync(id, {
  description: "A highlighted callout box",
  properties: [{ name: "Heading", key: "heading", type: { primitive: "string" }, default: "Note" }],
});
await etch.components.deleteAsync(id);
```

### Component properties

Discriminated union keyed by `type.primitive` (+ optional `type.specialized`).
Every property shares:

```ts
interface ComponentPropertyBase { name: string; key: string; description?: string; }
```

Variants (`ComponentProperty = ComponentPropertyBase & one of these`):

```ts
// string, with optional specialization
{ type: { primitive: "string"; specialized?: "color"|"url"|"image"|"select"|"array"|"wpMediaId" };
  default?: string; options?: string[]; selectOptionsString?: string; }

// number — reserved, not yet implemented in the builder UI
{ type: { primitive: "number" }; default?: number; options?: number[]; }

// boolean — default can be a string for expression-driven defaults
{ type: { primitive: "boolean" }; default?: boolean | string; }

// object / array (generic)
{ type: { primitive: "object"; specialized?: string }; default?: Record<string, unknown> | unknown[]; }
{ type: { primitive: "array"; specialized?: string }; default?: unknown[]; }

// array specialized "class" — CSS class list
{ type: { primitive: "array"; specialized: "class" }; default?: string[]; }

// object specialized "group" — nested properties
{ type: { primitive: "object"; specialized: "group" }; properties: ComponentProperty[]; }

// array specialized "repeater" — nested properties, repeated per row
{ type: { primitive: "array"; specialized: "repeater" }; properties: ComponentProperty[]; }

// string specialized "condition" — gated group shown when condition is true
{ type: { primitive: "string"; specialized: "condition" }; properties: ComponentProperty[]; default?: string; }
```

`selectOptionsString` (for `specialized: "select"`) is newline-separated
`Label : Value` entries; a line with no `:` uses its text as both; the first line
is the default:

```js
const colorProp = {
  name: "Color", key: "color",
  type: { primitive: "string", specialized: "select" },
  selectOptionsString: ["Red : #ff0000", "Green : #00ff00", "Blue : #0000ff"].join("\n"),
};
```
