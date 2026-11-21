# Etch Studio runtime — data sources, content, routes, assets, deploy

Etch Studio is the hosted Etch runtime (`studio.etchbuilder.io/<site>/…`,
`etch.environment.product === "etch-studio"`). Same builder and `etch.blocks` /
`etch.styles` / `etch.components` API as EtchWP, but the data layer is different:
no WordPress, no `etch.loops`, no `etch.fields` (both throw `NOT_AVAILABLE`).

None of these namespaces are on docs.etchwp.com yet (checked 2026-11-05: only
`assets` is). Sources, in order of trust: the `etch-connector tabs` output for the
connected tab (authoring rules per runtime), probing the live API
(`Object.keys(etch.<ns>)`, `String(etch.<ns>.<fn>)` shows the minified source),
and what is verified below. Everything marked **verified** was run against the
NormadicStudio site and checked on the published frontend.

Writes in all of these namespaces **persist immediately** — no `saveAsync()`.

## Capabilities

```js
etch.environment // { product: "etch-studio", capabilities: { loops: false, fields: false,
                 //   dataSources: true, content: true, routes: true, templates: true,
                 //   assets: true, deploy: true, wpMedia: false }, blockTypes: [...] }
```

Block types available: `etch/component`, `etch/condition`, `etch/dynamic-element`,
`etch/dynamic-image`, `etch/element`, `etch/head`, `etch/loop`, `etch/raw-html`,
`etch/render`, `etch/slot-content`, `etch/slot-placeholder`, `etch/svg`, `etch/text`.

## Data sources — `etch.dataSources`

```ts
list(): DataSource[];                       // sync
get(key: string): DataSource;               // sync
findDataSource(query: string): DataSource[];// sync, fuzzy
createAsync(def): Promise<…>;
updateAsync(key: string, patch: { name?, metadata? }): Promise<void>;
deleteAsync(key: string): Promise<void>;
resolveAsync(key: string, ...args): Promise<unknown>; // run it, return the value
```

`DataSource = { key, name, type: "json" | "js" | "api", metadata }`:
- `json`: `metadata.data` is the value (e.g. a navigation array).
- `js`: `metadata.code` is the **body of an async function**; `return` the value.
  It receives `args` (the extra arguments from the template call) and has
  `queryContent()` in scope.

**Verified:** `updateAsync(key, { metadata: { code } })` replaces the code, and
`get(key).metadata.code === code` right after. Back up `list()` to a file before
bulk-editing — there is no undo.

**`resolveAsync` takes the template arguments positionally:**
`resolveAsync('altLang', 6, '/fb/')`. Passing an array (`resolveAsync(k, [6, '/fb/'])`)
silently gives the script `args[0] = [6, '/fb/']`, `args[1] = undefined`, so
every result falls back to its default — easy to mistake for a working run.
Always call `resolveAsync` before authoring property paths into a template; don't
guess the shape.

### In templates

A loop carries **only** a `target` expression yielding a list — no
loopId/loopParams:

```js
{ type: "etch/loop", target: "data('testimonials', 'nb', 3)", itemId: "t", … }
// also: data('products').items, item.tags (nested), range(1, 10)
```

`data('key', a, b)` → the source's `args = [a, b]`. Use `this.id` for the current
page/item: `data('projectCase', this.id, 'nb')`. Because the data source decides
the shape, put per-language labels, URLs and computed flags (`isTranslated`,
`hasProject`, counts) in the source and keep templates dumb — one component
then serves every language.

### `queryContent()` inside a `js` source — always set `relationDepth`

```js
await queryContent({ type: 'services', where: { 'fields.language': 'en' }, relationDepth: 0 });
```

By default `queryContent` expands every relationship **three levels deep**. On a
site where content types link to each other (translations, services ↔ projects ↔
packages) that repeats the same items over and over: on NormadicStudio, 14
packages came back as **2.3 MB**, 3 services as 880 KB. With `relationDepth: 0`:
8 KB and 48 KB (verified 2026-11-01). Advice from the Etch team, adopted as a rule:

- `relationDepth: 0` unless the template shows something from a linked item.
- `relationDepth: 1` if it does (e.g. a testimonial card showing
  `t.fields.project.title`).

**At depth 0 a relationship field is an id (or an array of ids), not an object.**
Code that reads `f.translation.id` or `rel.title` breaks silently. Look items up by
id from a list you already queried:

```js
function relId(r) { if (Array.isArray(r)) r = r[0]; if (r == null || r === '') return null;
  return Number(typeof r === 'object' ? r.id : r); }
var byId = {}; services.forEach(function (s) { byId[s.id] = s; });
var other = byId[relId(f.translation)];
```

Image/asset fields come back as objects with `url` (and the asset id) even at
depth 0; content item fields hold the bare asset id (e.g. `gallery_1: 16`).

Before changing a source that pages depend on, capture its output for every
relevant id/language, change it, and diff — the safest way to prove a refactor
changed nothing.

### Sandbox limits seen in `js` sources

No `Intl` (format numbers by hand), no network. Plain ES2020 otherwise.

## Content — `etch.content`

```ts
listTypesAsync(); getTypeAsync(key);
createTypeAsync({ name, key?, isTemplate?, fields? }); updateTypeAsync(key, { name?, key? }); deleteTypeAsync(key);
addFieldAsync(typeKey, field); updateFieldAsync(typeKey, fieldKey, patch); removeFieldAsync(typeKey, fieldKey);
listItemsAsync(postType?); getItemAsync(id);
createItemAsync({ postType, title, fields? }); updateItemAsync(id, { title?, fields? }); deleteItemAsync(id);
```

- A content type is a **postType** (its key); an item is a **post** (its id).
- `updateItemAsync` **merges** fields per key; `null` clears one. Relationship
  values are item ids (`{ project: 14 }`).
- Field types seen: `text`, `textarea`, `number`, `boolean`, `select` (with
  `options` + `defaultValue`), `image`, `relationship` (`target`),
  `relationshipBacklink` (`source: { postType, fieldKey }`, read-only).
- **A field's type can't be changed** — `updateFieldAsync(t, k, { type: 'text' })`
  on a `select` throws `INVALID_ARGUMENT`. Add a new field and migrate.
- **A select's `defaultValue` is not stored on items.** An item whose language
  select was never touched has no value, so `where: { 'fields.language': 'en' }`
  doesn't match it even though the editor shows "English". Set values explicitly.
- Removing a `relationshipBacklink` directly throws `READONLY`; remove the
  `relationship` it mirrors and the backlink goes with it.
- **`createTypeAsync` auto-creates a `js` data source with the type's key**
  (`return await queryContent({ type: '<key>' })`, no `relationDepth`). So a later
  `dataSources.createAsync({ key: '<same key>' })` throws "already exists" —
  `updateAsync` the generated one instead, and add `relationDepth: 0` (2026-11-07).
- **A self-referencing relationship can't go in `createTypeAsync`'s `fields`**
  (`target` = the type being created throws `No content type "<key>"`, and the
  type is not created). Create the type, then `addFieldAsync` the relationship.
  The auto backlink it adds is keyed after the type (e.g. `faqs`), not
  `translation_of` as on older types.
- Writing content is a real data change — the user's content. Confirm before
  bulk writes or deletes; verify with `getItemAsync` afterwards.
- Item blocks (a page's layout) are edited by opening the item
  (`etch.navigation.openPostAsync(id)`) and using `etch.blocks`.

## Raw HTML — `etch/raw-html`

`{ type: "etch/raw-html", content: "<div>…</div>", unsafe: "true" }` (both string keys).
**A `<script>` inside `content` is escaped on publish** — it renders as visible text
(`&lt;script …&gt;`) and never runs, even with `unsafe: "true"` (seen 2026-11-12 with
Termageddon embeds). Put markup in the raw-html block and the script as its own
`etch/element` with `tag: "script"` and `attributes.src` next to it (or in the
template's `etch/head`). Check the live HTML for `&lt;script` after publishing.

## Routes and SEO — `etch.routes`

```ts
listAsync(); createAsync({ segment, parentId?, source?, templateId? }); updateAsync(id, patch); deleteAsync(id);
getPathsAsync();   // every built path — check links against this
listSeoAsync();    // every published page's resolved SEO, one call
```

- Segments: static (`'about'`), group (`'(marketing)'`, adds nothing to the URL),
  dynamic (`'{this.title.toSlug()}'`, one page per record of its source).
- A route renders through its `templateId` or the nearest ancestor's.
- SEO lives on the route (`title`, `description`, `language`, `canonical`,
  `socialImageId`, `faviconId` root only, `searchVisibility`, `redirect`,
  `notFoundId`); `null` inherits. Don't hand-write `<title>`/meta in `etch/head`
  except on the 404 page.
- **Don't build links from `.slug`.** Records served by a dynamic route have none
  (it reads as `''`, i.e. the homepage). Spell the link from the route's segment —
  `/projects/{p.title.toSlug()}/` — or compute it in a data source, and check it
  against `getPathsAsync()`.

## Assets — `etch.assets`

The runtime's own asset store (Etch Studio) or WordPress media (EtchWP dev builds,
via granular `assets*` capability flags — see https://docs.etchwp.com/public-api/assets).

```ts
listAsync({ mimeType?: 'image/', collectionId? }); getAsync(id);
uploadAsync({ data /* base64 */, fileName, alt }); downloadAsync(id); // { fileName, mimeType, data }
updateAsync(id, { alt?, collectionIds? }); compressAsync(id, { format, quality, resize? }); deleteAsync(id);
listCollectionsAsync(); createCollectionAsync(name, parentId?); updateCollectionAsync(id, patch); deleteCollectionAsync(id);
```

From the shell, prefer the connector commands — they handle the base64:

```bash
npx @digital-gravy/etch-connector upload ./hero.jpg --alt "Team on the roof" --compress webp:80
npx @digital-gravy/etch-connector download 12 --out /tmp/hero.webp
```

Bind images with `etch/dynamic-image` and the numeric asset id as a string:
`attributes: { mediaId: "{img.id}", useSrcSet: "true", alt: "{img.alt}" }`.
On Etch Studio it renders a plain `<img>` carrying the block's class (verified),
so `object-fit` etc. go straight on that class. Asset records carry no
width/height — a script can't tell a portrait screenshot from a landscape one.
Never call `/wp/v2/media` on Etch Studio.

## Publishing — `etch.deploy.deployAsync()`

```js
const { url } = await etch.deploy.deployAsync(); // e.g. "https://normadicstudio.com"
```

- No arguments. Publishes the **whole site** from its saved state — everything
  saved but unpublished goes live, not just the page you worked on. Unsaved
  buffered changes are not included: `saveAsync()` first if they should be.
- Took ~43 s on NormadicStudio (run with `--timeout 280000`).
- Throws `DEPLOY_BLOCKED` (message lists the reasons) or `OPERATION_FAILED`.
- Not in the docs or the connector's `--help` (found via `Object.keys(etch)`,
  2026-11-04). Feature-detect: `etch.environment.capabilities.deploy`.
- **Outward-facing: only when the user asks to publish**, per publish — an
  earlier "publish it" doesn't cover later changes.
- Verify afterwards against the live site: curl the pages with a cache-busting
  query, grep for real copy text and classes, and check compiled CSS in
  `/css/etch-block-styles.css`.
