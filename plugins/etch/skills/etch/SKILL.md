---
name: etch
description: Build and edit pages, templates, and components in the Etch page builder — Etch for WordPress (etchwp.com) and hosted Etch Studio (studio.etchbuilder.io) — HTML structure, CSS, reusable components, dynamic loops, custom fields, Etch Studio data sources/content/routes/assets, and publishing — by connecting live to an open Etch builder tab via the Etch Connector and driving window.etch. Use whenever the user wants something built/styled/fixed in Etch, or mentions Etch, EtchWP, Etch Studio, docs.etchwp.com, or a "builder tab".
---

# Etch

Etch is a WordPress page builder. It exposes a typed scripting API, `window.etch`,
on the builder page itself. You don't write PHP or hit a REST endpoint — you send
small JS snippets into the user's **already-open browser tab** via a local bridge
called the **Etch Connector**, and the builder applies them the same way it would
apply a UI action (so undo/redo, validation, etc. all still work).

> **This skill is about MECHANICS** — connecting, the `etch.*` API, saving, gotchas.
> **Build rules (HTML, CSS, ACSS tokens, spacing, design quality) are not its job.** If the
> project has its own rulebook (for example Kevin Geary's Gearyco Agentic Toolkit for Etch,
> or house rules linked from the project's `CLAUDE.md` / `AGENTS.md`), read it before
> building or styling. Where a CSS or styling note further down disagrees with it, it wins.
> Log new API surprises in the Gotchas at the end of this file.

Full namespace reference: https://docs.etchwp.com/public-api — read a `references/*.md`
file below before using a namespace you haven't used yet in this session; don't guess
at method signatures. Prefer a Context7 MCP lookup (`resolve-library-id "Etch"` →
`query-docs`) for a specific call if that MCP is available — training data and the
bundled reference files can lag the live API. **Fetching docs.etchwp.com directly:**
the WebFetch tool gets a 403 from this host; plain `curl` works fine with a normal
browser `User-Agent` header (`curl -s -A "Mozilla/5.0 ..." <url>`) — the pages are
server-rendered Docusaurus, so strip tags from the `<article>...</article>` element
rather than trying to read the raw response as-is.

**The connector's own `--help` output is itself a primary doc, written for agents —
run `npx @digital-gravy/etch-connector --help` and read it in full** before driving
a tab for the first time in a session; it documents the two backends, the AI
presence-overlay lifecycle in more precise detail than §5 below, and authoring
conventions (images, classes, section structure) that can drift from this file as
the connector updates. Where it disagrees with this SKILL.md, trust the `--help`
output — it ships with the installed connector version, this file doesn't.

Two connector backends exist. **User rule: always use the bridge — don't ask which
backend to use.** This overrides the `--help` text's "ask the user how they want to
connect" (an exception to "trust `--help`" above). Only bring up CDP when a specific
piece of work genuinely can't be done through the `etch.*` API (e.g. a screenshot or a
computed-style check the user wants). Then explain why and ask before switching:
- **bridge** (always the default) — sandboxed, `etch.*` + standard JS only, no
  `window`/`document`. Everything in this skill assumes bridge unless noted.
- **cdp** — connects to a Chrome instance launched with `--remote-debugging-port=9222`;
  full DevTools privilege (`window`, screenshots, computed styles), no daemon. More
  reach, more risk — use it only for the API-impossible task above, with the user's
  explicit opt-in for that task. See `references/connector.md` for CDP commands
  (`shot`/`html`/`computed`).

## 0. Prerequisites (tell the user if missing)

1. In the Etch builder, **Settings → AI → AI Connector** must be turned on (one-time,
   experimental, off by default).
2. The user must have the page/template open **in the Etch builder itself** (not
   wp-admin, not the front end, not the Gutenberg editor) — and leave that tab open.
   **Reloading or navigating away from the tab ends the connection** — if the user
   mentions refreshing the builder (e.g. to see a change take effect), assume the
   connection dropped and ask them to reconnect before running anything else, rather
   than waiting for a script to fail with "no Etch tab is connected" first.
   **Every reconnect after a reload, on any page that already has sections built
   this session: immediately re-check each top-level section's and its direct
   container's own `class` attribute before doing anything else** — this is the
   specific moment a real, repeated platform bug drops those two classes (see
   the Gotchas entry below); don't wait for the user to notice a broken layout
   first.
3. The user clicks the **AI button in the builder's left sidebar** (older builds:
   a sparkles button in the toolbar, "Connect external AI agent"). It connects
   the tab under the site name; clicking it again disconnects. Fallback from the
   browser console: `etch.connectAs('<name>')`.

## 1. Connect

Run:

```bash
npx @digital-gravy/etch-connector serve
```

This starts a local bridge (loopback only, `127.0.0.1`) and keeps running — run it
with `run_in_background: true`, don't block on it. Ask the user to click the AI
button in their builder tab now if they haven't. Confirm the connection with:

```bash
npx @digital-gravy/etch-connector tabs --json
```

The connection persists for as long as *this chat* stays open and the builder tab
stays open/on-screen. You do not need to re-run `serve` for every message — only if
the user tells you the connection dropped or they closed this chat before. Never
connect a second builder tab for the same site; one tab per site.

## 2. Run scripts against the tab

```bash
npx @digital-gravy/etch-connector eval -t <tab> "<js>"
```

- `-t <tab>` names the tab; omit it when only one tab is connected.
- Prefer `-f <file>` for anything more than a couple of lines — write the script to
  the scratchpad, then `eval -f`. Easier to iterate on than shell-quoted inline JS.
- The body runs as an **async function body**: use `await`, and `return` the value
  you want back. The return value comes back as JSON on stdout.
- **Safe mode**: only `etch.*` and standard JS are available — no `window`, no
  `document`, no network, no storage. Reference `etch` directly, not
  `window.etch`.
- Exit codes: `0` success, `2` the script threw, `1` operational failure (tab not
  found, timeout, connector unreachable). Check exit code, not just stdout.
- **Default timeout is 30 s.** Anything that walks several pages
  (`openPostAsync`/`openTemplateAsync` in a loop), resolves many data sources, or
  deploys needs `--timeout <ms>` (e.g. `--timeout 280000`). A timeout does not
  mean nothing happened — the tab may still finish; check state before re-running.
- A script that throws mid-way leaves everything before the throw applied — and
  can leave the builder **inside component edit mode**. Check
  `etch.blocks.isInComponentEditMode()` and state before retrying.

Example:

```js
const [id] = etch.blocks.find({ type: "etch/text" });
etch.blocks.setText(id, "Hello world");
await etch.saveAsync();
return etch.blocks.getJson(id);
```

## 3. The mental model

`getEtch()` (already done for you — you just call `etch.*` inside the eval script)
returns one object with these namespaces. Read the matching reference file the
first time you touch a namespace in a session:

| Namespace | Reference | Covers |
|---|---|---|
| `etch.blocks` | `references/blocks.md` | select/read/create/mutate/delete blocks, component edit mode |
| `etch.styles` | `references/styles-components.md` | CSS rules (classes/ids/etc.) and `:root` variables |
| `etch.stylesheets` | `references/styles-components.md` | `@custom-media` definitions only — not for element/page styling, see §3.5 |
| `etch.components` | `references/styles-components.md` | reusable component definitions + props |
| `etch.loops` | `references/loops-fields-nav.md` | WP query / terms / users / JSON data sources bound to `etch/loop` blocks |
| `etch.fields` | `references/loops-fields-nav.md` | custom field groups, fields, per-post values |
| `etch.navigation` | `references/loops-fields-nav.md` | switch builder area / open a different post or template |
| `etch.ui` / `etch.history` | `references/loops-fields-nav.md` | color scheme, chrome visibility, undo/redo |
| `etch.skills` | below | Etch's own bundled authoring-convention guides |
| `etch.ai` | below | presence overlay so the human sees you're active |
| block JSON shapes, `EtchApiError` codes | `references/types-errors.md` | the type union every block/mutation is checked against |
| `etch.dataSources`, `etch.content`, `etch.routes`, `etch.assets`, `etch.deploy` | `references/etch-studio.md` | **Etch Studio runtime**: JS/JSON/API data sources (`queryContent`, `relationDepth`), content types/items, the route tree + SEO, the asset store, publishing |

**Two runtimes, different surfaces.** `etch.environment.product` is `"etchwp"`
(WordPress) or `"etch-studio"` (hosted, studio.etchbuilder.io). On **Etch Studio**,
`etch.loops.*` and `etch.fields.*` throw `NOT_AVAILABLE` — loops bind a `target`
expression like `data('key', arg1, arg2)` instead, and content lives in
`etch.content`. Read `npx @digital-gravy/etch-connector tabs` (no `--json`) at the
start of a session: it prints the runtime, its capabilities and its authoring
rules. Branch on `etch.environment.capabilities`, never on the product name.

**Persistence is split by namespace** — get this wrong and changes silently don't
save:
- `etch.blocks`, `etch.styles`, `etch.loops` are **buffered** — need
  `await etch.saveAsync()` to persist.
- `etch.stylesheets.*Async`, `etch.components.*Async`, `etch.fields.*Async`
  **persist immediately** — no `saveAsync()` needed (and calling it is harmless but
  irrelevant to those). Same for Etch Studio's `etch.dataSources`, `etch.content`,
  `etch.routes` and `etch.assets` writes, and component edit mode's
  `saveComponentEditModeAsync()` — those are live the moment they resolve, so treat
  them with the care of a database write (back up a definition before rewriting it).
- `etch.deploy.deployAsync()` **publishes the whole site** — see
  `references/etch-studio.md`. Only on the user's explicit request.

> **Default: do NOT call `saveAsync()` (or otherwise persist) unless the user has
> explicitly asked you to save.** Per `etch-connector --help`, authored changes are
> meant to stay pending in the builder until the human reviews and saves them — the
> same way you wouldn't `git commit` unasked. Build in the canvas, describe what you
> did, and let the user click Save themselves; only call `saveAsync()` in a given
> session once they've said to (e.g. "save it", "looks good, publish it") — and even
> then, treat that as covering the current change, not a standing blanket okay for
> every future one, unless they say otherwise. The "6. `await etch.saveAsync()`"
> step later in this file and the worked examples throughout predate this — read
> every `saveAsync()` call in this file as conditional on that go-ahead, not routine.
>
> **Standing exception (user, 2026-11-04): save before switching pages.** When the
> work needs to move to another page or template (`openPostAsync`,
> `openTemplateAsync`), call `await etch.saveAsync()` first instead of stopping to
> ask — navigating away with unsaved buffered changes risks losing them. Mention
> which pages were saved that way in the summary. Everything else stays unsaved for
> the user to review.
>
> Separately: `saveAsync()` persisting blocks/styles to the buffer is not the same
> as the page's **compiled CSS** being current — see the callout under §3.5 below.
> A manual Save/Publish click in the builder has, at least once, been the only thing
> that made newly-created `etch.styles` class rules compile into the live frontend's
> `<style id="etch-page-styles">`, even though `saveAsync()` had already written the
> block/style JSON to the published post. Don't treat `saveAsync()` resolving as
> proof the page is visually finished — verify against the live frontend (§5.6,
> §6 step 7) and be ready to ask for a manual Save/Publish click if new class styles
> aren't showing up there.

## 3.5 Styling: apply classes directly with `etch.styles.create()` + `addClass()`

**Default pattern — one dedicated class per meaningfully distinct element, scoped
per section/component** (e.g. `.wl-hero__grid`, `.wl-services__card`, not a single
`.card`/`.grid-3` reused across unrelated sections). This is what lets the user
select any element in the Etch canvas and see/edit *its own* style directly in the
builder's Style Manager, instead of a shared class silently affecting other,
unrelated elements — that per-element editability is the reason to prefer this over
a hand-authored `etch.stylesheets` CSS blob.

**User preference: give every element a BEM class on creation, even if it needs no
CSS right now.** `addClass(blockId, "hero__inner")` with no matching
`etch.styles.create()` call is expected and fine — don't skip the class just
because there's nothing to style yet. This applies to wrapper/container `div`s too,
not only elements that obviously need visual rules (buttons, grids, cards). The
payoff: every element is already addressable in the Style Manager and via
`find({ class: ... })` the moment the user wants to style it later, with no
retrofitting pass needed. The `data-etch-element="section"/"container"/"flex-div"`
structural wrappers from §5.5 still also get their own BEM class on top of the
`data-etch-element` attribute — the two are orthogonal (see §5.5's last bullet).

```js
const className = "wl-hero__grid"; // no leading dot — this is a plain class name
etch.styles.create("." + className, "display: grid; grid-template-columns: var(--grid-2); gap: var(--grid-gap);");
etch.blocks.addClass(blockId, className); // literal class-name string
await etch.saveAsync();
```

**Verified end-to-end on a live 0.x runtime (Etch 1.6.1, 2026-07-04):** created ~70
classes this way across an 11-section page — `getJson()` showed the correct literal
class on every block, `etch.styles.list()` showed the matching rule linked, and
curling the live published frontend (cache-busted) confirmed both the class
attribute *and* a matching compiled CSS rule for every one of them. This is the
default to reach for; don't reach for `etch.stylesheets` first.

**The one real gotcha:** `etch.styles.create(selector, css)` returns an *opaque
style id* (e.g. `"6jx1uil"`), not the class name. Passing that id straight into
`addClass()` — instead of the literal class-name string you used in the selector —
sets the block's `class` attribute to a sanitized version of the id (e.g.
`"jx1uil"`) with no matching CSS anywhere. Always pass the literal name string:

```js
const styleId = etch.styles.create(".probe-test", "color: red;"); // styleId is opaque, e.g. "6jx1uil" — don't use it here
etch.blocks.addClass(blockId, "probe-test"); // correct: the literal name, not styleId
```

If something looks wrong on a new site/version, re-run this probe before concluding
`etch.styles` itself is at fault: create a throwaway class, addClass with the
literal name, `saveAsync()`, then curl the live frontend with a cache-busting query
param and grep for both the class attribute and the matching CSS rule.

**Responsive breakpoints go inside the same class's `css`, nested — never a
separate stylesheet.** `etch.styles.create(selector, css)` rejects a selector that
is itself an `@media {...}` block (throws `Invalid style selector`). The fix is to
nest the `@media` rule *inside* the `css` argument of the normal class selector —
confirmed compiling correctly end-to-end on 1.6.1 (`etch.styles.list()` and the
published frontend's `<style>` block both show the nested rule under the class):

```js
const id = etch.styles.create(".hero__grid",
  "display: grid; gap: var(--grid-gap); @media (min-width: 768px) { grid-template-columns: var(--grid-2); }"
);
etch.blocks.addClass(blockId, "hero__grid");
await etch.saveAsync();
```

Same when editing an existing rule: `etch.styles.update(styleId, { css: "...@media (...) { ... }" })`.
This keeps the breakpoint on the one dedicated, per-element class — still editable
in the Style Manager — with no stylesheet involved.

**Do not use `etch.stylesheets` for element or responsive styling — it is not an
alternative to `etch.styles.create()` + `addClass()`, use the class-per-element
pattern above for every case in this section.** A session previously reached for
`etch.stylesheets.createAsync()` with a hand-written `@media` CSS blob to solve
the exact "selector can't be a media query" problem above — that workaround is
wrong: it moves the rule off the element's own class and onto an untracked global
stylesheet the Style Manager doesn't surface per-element, and it was only ever
needed because the nested-`@media`-in-`css` approach above hadn't been tried yet.
With that confirmed working, there is no remaining reason to create a stylesheet
for page content. If you ever find `etch.styles` genuinely can't express something
(vs. just not having tried nesting), stop and flag it to the user rather than
silently falling back to a stylesheet.

**Update — this is now a confirmed, reproducible platform behavior, not a build
mistake (verified 2026-09-04 on Etch 1.6.5/1.6.6):** a prior report claimed the
class-per-element pattern doesn't compile on Etch 1.5.3 — data model correct in
`getJson()`/`styles.list()`, but the compiled `<style id="etch-page-styles">` block
never picked up the rule. An earlier version of this file called that "unverified,
likely a build-process mistake." It has since reproduced cleanly on a different
site/version: real section CSS (12 rules across a two-card hero) AND a trivial
one-line throwaway probe rule (`.probe-test { color: red; }`) both persisted
correctly in `etch.styles.list()`, both rendered correct block class attributes on
the live frontend, and **neither** rule's CSS appeared in the compiled
`<style id="etch-page-styles">` output — ruled out caching (the post's REST
`modified`/`modified_gmt` timestamp updated live with each edit) and ruled out
needing more `saveAsync()` calls or a wait (retried repeatedly over 15+ seconds).
Pre-existing template classes on the same page compiled fine, so the pipeline isn't
globally broken. **What resolved it: a manual Save/Publish click by the user in the
actual builder tab** — the compiled CSS appeared immediately after, with no other
change. `etch.styles.setVariable()` (`:root` variables) was NOT affected — that
compiled live instantly with no manual click, so this looks specific to the
class-selector `create()`/`update()` path. If you hit missing compiled CSS again:
run the one-line probe rule to confirm it's this (not something specific to your
CSS), then ask the user for a Save/Publish click rather than re-diagnosing from
scratch — don't reach for a stylesheet as a workaround.

Also note: some older Etch versions (seen on 1.5.3) don't expose `etch.skills` or
`etch.ai` at all — `typeof etch.skills`/`typeof etch.ai` are `"undefined"`, not just
empty. Feature-detect before calling either, and skip the presence-overlay calls in
§5 entirely if `etch.ai` is undefined rather than wrapping them in try/catch.

**Classes in authored JSON — changed in current builds (connector 0.5.x, Etch
Studio 2026.11):** the connector's `--help` now says to *set* `attributes.class`
in JSON passed to `blocks.create()`, `blocks.replace()` and
`components.updateAsync()` — each class links its style record, and a missing one
is created empty. (Older builds silently dropped `attributes.class`; that is what
the previous version of this note said. If a class vanishes after authoring, you
are on an older build — fall back to `addClass()` after creation.) Passing
`styles` yourself is still ignored/rejected; `addClass()` on existing blocks still
works. Re-adding a class name that used to exist re-links to its existing style
record, so you can't orphan or duplicate the CSS rule.

**Don't judge class↔rule linkage by the `styles` array.** Observed 2026-11-05
(Etch Studio): component elements whose classes were added in edit mode *before*
their `etch.styles.create()` rules existed kept reporting `styles: []` in
`getJson()`/`components.getJson()` — even after remove/re-add — yet the published
page carried the class and the compiled rule applied correctly. In edit mode
`styles` reads `[]` even immediately after `addClass()`. The render is the only
trustworthy check: curl the frontend's compiled CSS (Etch Studio:
`/css/etch-block-styles.css`) and the page HTML. Cleaner habit regardless: create
the `etch.styles` rules *first*, then author the classes.

**Don't defensively zero out margins (`margin: 0;`) on headings/paragraphs as a
reflex.** User feedback, 2026-09-09: this had become a habit — adding `margin: 0`
to most new heading/paragraph classes whether or not anything actually needed it.
Flex/grid `gap` on the parent already governs the spacing between siblings in
practically every layout built this way, and ACSS's own heading/paragraph
defaults are generally fine left alone. Only zero a margin when something
concrete is actually wrong — visible double-spacing, an element not sitting
where a `gap`-based layout expects — not as a blanket precaution on every new
class. This is the same spirit as not hand-rolling section/container spacing
(§5.5): reach for an override because you found a real problem, not by default.

## 4. Before building anything: check for authoring conventions

Etch ships read-only guides for the page's active integrations (e.g. Automatic.css).
Check these *before* hand-rolling CSS or spacing choices — the site may already have
a documented convention (utility classes, spacing scale, etc.) you should follow
instead of inventing your own:

```js
const skills = etch.skills.list(); // → [] if none apply to this page
```

If something relevant is listed, `etch.skills.get(name)` for the guide body, and
`etch.skills.getReference(name, file)` for a specific sub-doc, only as needed. If
the site uses Automatic.css but `etch.skills.list()` comes back without an `acss`
entry (older ACSS version, or the guide just isn't bundled on that install), fall
back to https://docs.automaticcss.com/ directly rather than guessing token names —
same curl-with-browser-UA approach as docs.etchwp.com if WebFetch 403s on it too.
A local copy of the ACSS + Etch doc repos lives in
`~/.claude/training/etch-acss/upstream/` — grep it for a token or feature before
fetching the web docs.

**Also check the runtime's own capabilities before assuming what's available.**
`npx @digital-gravy/etch-connector tabs` (no `--json`) prints, per connected tab,
which runtime backs it (from `etch.environment`, when present) followed by that
runtime's own authoring rules — read this at the start of a session, not just
`tabs --json`, which strips it. On an Etch build that predates `etch.environment`
the CLI says so explicitly and falls back to a WordPress assumption — treat that as
an assumption to confirm with the user, not a fact, before relying on WP-specific
surface (loops, fields, templates, WP media) or assuming every block type exists.
Branch on a reported capability, never on "it's Etch so it must be WordPress."

## 5. Show presence while you work

This overlay is **for the human watching the builder tab**, not for the chat user —
per the connector's own agent docs: never mention, narrate, or explain these state
changes in chat unprompted. (If the chat user asks you to use it or asks about it
directly, answering them is fine — you're responding to their question, not
volunteering an implementation detail.)

States: `"idle"` (overlay hidden — same as `hide()`), `"thinking"` (reasoning before
acting), `"working"` (a script is actually running against the canvas), `"waiting"`
(paused, needs input from the human). Any other string throws `INVALID_ARGUMENT`.
Calls are synchronous, in-memory, and **not persisted** — no `saveAsync()` involved,
and nothing survives a reload.

Precise lifecycle (more granular than "set working, then hide"):
- The **moment** the user asks for something built/changed — before reading docs,
  before writing any script — set `"thinking"`.
- Stay in `"thinking"` while reasoning/planning between scripts.
- Set `"working"` right before an `eval` actually runs against the canvas; drop back
  to `"thinking"` once it returns if there's more reasoning before the next one.
- `hide()` the instant the task is actually finished — never leave it up on
  completion.
- Set `"waiting"` only when you are not done and are genuinely handing back to the
  human for input or a decision (not a routine end-of-turn); resume
  `"thinking"`/`"working"` once they reply.

```js
etch.ai.setState("working");
try {
  // ...make changes...
} finally {
  etch.ai.setState("thinking"); // or etch.ai.hide() if the task is actually done
}
```

## 5.5 Section scaffold — the default pattern for new sections

**Verified against real Etch-authored content on this site** (existing hero/about
sections, not something I inferred from docs): a top-level section is always at
least two nested `etch/element` blocks, each carrying a `data-etch-element`
attribute that hooks into base layout CSS the site already ships — these aren't
decorative, they carry real rules (`inline-size`, `flex`, centering,
`max-inline-size: var(--content-width, ...)`).

```js
const sectionId = etch.blocks.create({
  type: "etch/element", version: 1, context: {}, children: [],
  tag: "section",
  attributes: { "data-etch-element": "section" },
});
etch.blocks.update(sectionId, { name: "<Descriptive> Section" }); // e.g. "Hero Section"
etch.blocks.addClass(sectionId, "hero"); // BEM root class — even before any CSS exists for it

const containerId = etch.blocks.create({
  type: "etch/element", version: 1, context: {}, children: [],
  tag: "div",
  attributes: { "data-etch-element": "container" },
}, sectionId);
etch.blocks.update(containerId, { name: "Inner" }); // default name — override only if the user asks for something else
etch.blocks.addClass(containerId, "hero__inner"); // same: class it now, style it later if ever needed
```

- `data-etch-element="section"` on the outer `section` tag — full-bleed, flex
  column, centers its children. Every top-level section gets this.
- `data-etch-element="container"` on the direct child `div` — constrains and
  centers content width (`var(--content-width, 1366px)`). Name it **"Inner"** by
  default (structure-panel label via `update()`/`rename()`) unless told otherwise.
- If you need a further-nested flex grouping *inside* the container (mirroring
  the site's own "Hero inner" pattern), add a third `div` with
  `data-etch-element="flex-div"` — generic full-width flex column, no width
  constraint. Not required for a simple section; add it only when you actually
  need another flex/gap context nested inside the container.
- A BEM root class (e.g. `hero`, `about`) is a separate, orthogonal concern from
  `data-etch-element` — apply it the normal way (§3.5) for anything visual;
  `data-etch-element` is structural/layout only and its three values above are
  fixed, not something to invent variants of.
- **Per the connector's own agent docs: do NOT hand-roll wrapper widths,
  max-widths, side padding, or inter-section margins on the section/container
  pair** — `data-etch-element="section"` + `="container"` already supplies
  vertical spacing between sections, horizontal edge padding, and a constrained,
  centered content width for free; fighting or duplicating that is a defect, not a
  style choice. A section needs **at least one** container as its direct child,
  with all real content nested inside containers, not loose in the section.
  **Sibling containers are fine** (Gearyco toolkit `AGENTS.md`) and are the right
  answer when parts belong together: ACSS spaces them with `--container-gap`, so
  merge related parts into one section instead of trimming two sections' padding.
  The one legitimate exception is a *deliberate* layout effect the defaults can't
  express — e.g. pulling one section up to overlap the section before it — and
  even then, scope the override to your own BEM class on that specific section
  (`.my-section { margin-block-start: calc(var(--section-space-s) * -1); }`), not a
  blanket override of the section/container defaults, and say in your summary to
  the user that you deliberately overrode the default spacing and why.

## 5.6 Large builds: batch it, don't send one giant script

**Observed on runtime 1.6.1:** a single `eval` script creating ~40+ blocks (a
two-column hero with a header, a 4-item repeated countdown, several badges) can
report success (`saveAsync()` resolves, `getTree()` shows the correct buffered
structure, no thrown error) while the frontend renders **nothing** for that
section — not a caching artifact, verified with cache-busting query params and by
diffing the raw HTML between `</header>` and `<footer>`. Trimming the same build
down to smaller pieces made it render correctly every time; the exact trigger
(likely something in how many rapid `create`/`addClass` calls land in one
`saveAsync()`) wasn't fully pinned down and may not repro on every runtime — but
splitting the work sidesteps it regardless of cause, so treat this as the default
posture for anything nontrivial rather than something to reach for only after a
failure.

**Practical rule:** for a section with more than ~20-25 total `create`/`addClass`
calls (a repeated countdown/list/grid of items is the usual culprit), split it into
2+ `eval` calls (each still ending in `await etch.saveAsync()` **only if** the user
has told you to save — see §3's callout; otherwise each batch just builds on the
prior one's buffered state, no save needed between them):
1. Build the outer scaffold + static content in the first call.
2. Fetch the anchor block from the first call's result (or re-`find()` it) and
   `create()` the repeated/large piece into it in a second call, using the
   `index` argument on `create()` to insert at the right position among existing
   siblings if it's not going at the end.
3. **After every batch, verify against the live frontend** — curl the page with a
   cache-busting query param and check for your actual copy text, not just for a
   class name (a class name can appear in the compiled CSS `<style>` block even
   when the block that should carry it never rendered). Do not trust
   `saveAsync()` resolving or `getTree()` looking right as proof the batch landed.
4. If a batch's content is missing from the frontend despite the editor reporting
   success, don't assume you found a permanent content-shaped cause — retrying the
   *exact same* batch, or splitting it smaller, has been enough to get it to land.

## 6. Typical build flow

1. `etch.skills.list()` — see if a convention guide applies.
2. `etch.blocks.getTree()` or `find()` to locate where you're inserting/editing.
3. Build structure with `etch.blocks.create()` (nest via `parentId`) starting from
   the section scaffold in §5.5, text via `setText`, classes via `addClass`. Give
   every element a BEM class as you create it, even ones with no styling yet
   (§3.5) — don't wait until step 4 to decide which elements "need" one.
4. Define any new CSS with `etch.styles.create(".class-name", css)` and apply it
   with `etch.blocks.addClass(blockId, "class-name")` — literal class-name string,
   not the id `create()` returns (see §3.5). One dedicated class per element,
   BEM-scoped per section. Nest any responsive breakpoint inside that same class's
   `css` (§3.5) — never create a stylesheet for this. Prefer existing
   variables/utility classes discovered in step 1 over inventing new ones.
5. For repeating structure use `etch.components` (a reusable definition) or
   `etch.loops` (data-bound repetition) — see the reference doc before designing
   props/queries from scratch. For anything large, batch per §5.6. If a component
   needs a part of its tree to be toggleable (an optional lede/CTA/badge), use an
   `etch/condition` block bound to a boolean prop, and add classes to the
   component's internal blocks through component edit mode, not through
   `updateAsync({ blocks })` directly (it silently drops `class` attributes) — see
   `references/styles-components.md` "Conditional content" and the "Gotcha" there,
   plus the edit-mode id gotcha in `references/blocks.md`.
6. **Don't `saveAsync()` unless the user has already told you to save** (see the
   callout under §3 — this is the resolved default, not open per-session). If they
   have, call it now.
7. If you did save, verify against the **live frontend**, not just the editor's
   buffered state — curl the page with a cache-busting query param and check for
   your actual copy text (see §5.6, point 3 — a passing `saveAsync()` isn't proof it
   rendered). If Chrome remote debugging is available, `references/connector.md`
   also has `shot` / `html` / `computed` (CDP mode) for visual checks. If you
   *didn't* save, verify against `etch.blocks.getTree()`/`getJson()` instead — the
   buffered editor state, not the frontend, since nothing's published yet — and
   tell the user what's built and pending, so they know to look at the canvas.

## Gotchas

- The surface is `0.x`/experimental — feature-detect
  (`typeof etch.blocks.x === "function"`) rather than trusting a method exists;
  don't hardcode behavior against a specific version.
- `PublicBlockJson.styles` is **read-only** on reads; you can't set it via
  `create`/`update` — apply classes with `addClass()` instead.
- Component props are a **closed schema** — setting an undeclared prop key throws
  `INVALID_ARGUMENT`.
- Putting a block under a parent that can't contain it (text under a void element,
  block-level child in a text container) throws `WRONG_BLOCK_TYPE` for `create`,
  `move`, and indexed `pasteAsync`.
- Wrap risky scripts in try/catch and check `err.code` via `isEtchApiError` — see
  `references/types-errors.md` for the full error-code list.
- **Icons / SVG graphics: always use `etch/svg`, never `etch/raw-html`, inline
  `<svg>` markup, or `etch/element` with `tag: "svg"`.** User requirement
  (2026-10-14), after a raw-html icon (`content: "{item.iconSvg}"`) was rejected.
  `etch/svg` loads an uploaded asset file and inlines it:
  `{ type: "etch/svg", attributes: { src: "/assets/<file>.svg", stripColors: "true"|"false" } }`
  (Etch Studio: asset path from the Asset Manager; give it its own BEM class
  via `addClass`). `stripColors: "true"` forces every fill/stroke to
  `currentColor` (fine for one-colour icons); use `"false"` to keep a two-tone
  icon whose SVG colours via CSS variables (`fill="var(--icon-accent, currentColor)"`).
  Asset upload has no API, so write the `.svg` files locally and ask the user to
  upload them (or use the Asset Manager UI) before expecting them to render.
  For a per-item icon inside a loop, add one `etch/condition`
  (`item.icon === "x"`) per icon, each wrapping its own static-`src` `etch/svg`.
  Don't depend on a dynamic `src` expression unless you've verified it.
- **Images: always use `etch/dynamic-image` (the Dynamic Image element), never a
  plain `etch/element` with `tag: "img"`.** Bind it by WordPress media ID, not a
  URL: `attributes: { mediaId: "{props.image}", useSrcSet: "true", alt: "{props.name}" }`.
  A component prop backing a media ID needs
  `type: { primitive: "string", specialized: "wpMediaId" }` — the plain
  `specialized: "image"` variant is a URL-based picker, wrong for `mediaId`.
  There's no `etch.media` API to resolve a URL → media ID: when the runtime backs
  the WP media library (check the `wpMedia` capability via `tabs`, §4), resolve it
  yourself with the WP REST API:
  `curl '<site>/wp-json/wp/v2/media?search=<term>&_fields=id,source_url'`
  (or `?rest_route=/wp/v2/media&...` on plain, non-pretty permalinks).
- If `eval` fails with tab-not-found/timeout, don't assume the API changed — the
  builder tab was probably closed/navigated away (including from a page reload the
  user did to view a change — see §0), or the connector was never confirmed
  connected. Ask the user to check the tab and the AI button before debugging
  the script itself.
- **A top-level section and its direct container child have, four times now,
  lost their `addClass()`-applied classes** — confirmed as a connector/platform
  issue, not a forgotten call, each time by checking `getJson()` immediately
  after `addClass()` (classes present, correct) and again later with no further
  script action taken (classes gone). **2026-09-06/07 finding, sharper than
  before:** on the clearest occurrence, **every block's `id` in the whole tree
  had changed** (section, container, AND their children, all the way down) —
  not just the top two. So this is NOT "section/container specifically get
  regenerated while children keep stable ids" — it's a full tree
  re-serialization/reload, and content, structure, and classes on *every other*
  block type (headings, paragraphs, nested component instances, deeply-nested
  classed elements) round-trip through that reload perfectly. **Only
  `data-etch-element="section"`/`"container"` blocks' `class` attribute fails to
  survive it.** Timing lines up with a **tab reload/reconnect** — the break was
  present as soon as the tab was reconnected after being disconnected, and never
  observed to happen spontaneously mid-script on a still-connected tab. Working
  theory (not proven — no visibility into the builder's internals): these two
  block types carry a distinct default-template linkage (visible as the baked-in
  `etch-section-style`/`etch-container-style` entries in their `styles` array)
  that gets re-asserted whenever the tree reloads, and whatever re-asserts it
  doesn't know to carry forward a class bolted on afterward via `addClass()` —
  even though content clearly persists across that same reload through *some*
  layer (`saveAsync()` was not called before several of these occurrences, yet
  the content survived), so there is an autosave/draft-style persistence path
  independent of the explicit Save/Publish action, and its section/container
  handling looks like the actual defect. **Practical rule: re-verify section and
  container `class` attributes right after *any* reconnect** (not just right
  after building) — that's the moment this has actually appeared, not "after
  building" or "after a component edit-mode session" as earlier (weaker, now
  superseded) versions of this note suggested. `getJson()` both ids, check
  `attributes.class` isn't empty, and re-`addClass()` if it is — verified as a
  full, durable fix every time so far, until the next reconnect.
- **2026-09-09 finding — the same class-loss also hits a `data-etch-element="section"`
  block living *inside a component definition's own template*** (not just raw
  page-level sections) — confirmed on `DTF Method Step`, a component whose root
  is itself a `section`. Same fix (re-`addClass()` via edit mode), same trigger
  (a reconnect in between). **Separately, and more seriously: an entire
  `components.updateAsync()` — properties, blocks, AND subsequent
  `etch.styles.update()` calls on its CSS — fully reverted to its pre-edit state**,
  not just one class. This was NOT preceded by any reconnect; it followed several
  rapid `etch.navigation.openPostAsync()` calls made shortly after the edit (the
  user separately reported having a different, unintended page open in the
  browser around the same time). Despite `components.updateAsync()` being
  documented as persisting immediately, this suggests the actual write may not
  be flushed to the server as instantly as that implies, and a page navigation
  (via the API, and possibly via the human's own in-browser navigation) shortly
  after can truncate/lose it even though the call itself resolved without error.
  **Practical rule: after any `components.createAsync()`/`updateAsync()` you
  intend to keep, verify it with a fresh `getJson()` read before navigating to a
  different post/template (via `openPostAsync`/`openTemplateAsync`) or otherwise
  moving on** — don't chain a component edit straight into a navigation call and
  trust the edit already landed. If the user mentions having had a different
  page open than expected around the time of an edit, treat that as a strong
  hint recent component edits may need re-verifying, not just section/container
  classes.
- **`etch.styles.setVariable()` corrupts values that contain brackets** (seen 2026-11-12, DEEP,
  Etch WP): `setVariable('--x', 'oklch(from var(--secondary) calc(l - 0.055) c h)')` rewrote the
  whole `:root` block into garbage (`--dtoklch(from …) #5160c1;`), and the next save published it.
  For any value with `(`/`)` (`oklch(from …)`, `calc()`, `var()`), edit the block directly:
  read `etch.styles.list().find(s => s.id === 'etch-global-variable-style').css`, change the text,
  `etch.styles.update('etch-global-variable-style', { css })`, read it back and compare **before**
  saving. Plain values (`#021720`) went through `setVariable` fine.
- **`saveAsync()` right after `openTemplateAsync()` + `create()` in the same script can save
  nothing** (Etch WP 1.8.3, 2026-11-17, an index template): it resolved, the
  template's `post_modified` changed, `getTree()` showed the new component instances, yet
  `post_content` in the database didn't contain them. A second `saveAsync()` in a separate
  `eval` stored them. After any save that follows a page/template switch, verify in the
  database (`wp post get <id> --field=post_content`) or on the frontend, and save again if missing.
- **Etch WP 1.8.3 findings (a single-post template, 2026-11-17):**
  - `etch.blocks.find({ tag: "main" })` returned a component instance, not the `<main>`
    element. Locate structural blocks through `getTree()` + `getJson()` and check `type`/`tag`.
  - `etch/text` always HTML-escapes. Rich text (ACF/SCF wysiwyg) needs `etch/raw-html` with
    `content: "{this.acf.field}"` — and `unsafe` is a **required string** in its JSON (`"false"`
    keeps WP's kses filtering), or create() throws "Invalid block JSON … unsafe".
  - Creating a block with `attributes.class` before its rule exists makes Etch auto-create an
    **empty** rule for that class; a later `styles.create()` guarded by "already exists?" then
    skips it. Create rules first, or `update()` the existing (empty) one.
  - Loop over a field on the current post: `{ type: "etch/loop", target: "this.acf.repeater", itemId }`.
    Named WP-query loop: `{ type: "etch/loop", loopId, itemId, loopParams: { $region: "this.region.at(0).id" } }`;
    `post__not_in: ["$exclude"]` (a param inside an array) works.
  - Category checks: `this.kategori.pluck("slug").includes("vin")` (post terms are on `this.<taxonomy>`,
    alphabetical; SCF taxonomy fields are on `this.acf.<field>`).
  - Unstyled paragraphs inherit ACSS's muted body colour; set `color: var(--text-dark)` where the
    design wants near-black.
  - Test URLs: don't use `?m=` as a cache-buster on WordPress, it's the month query var and 404s.
- **A condition's expression can't be changed with `update()`.** `BlockPatch`
  only accepts `name`, `hidden`, `attributes`, `text`, `target` — patching
  `conditionString` throws `INVALID_ARGUMENT: Unknown patch key(s)`. What
  worked (2026-11-04): nest a second `etch/condition` inside the existing one
  and `move()` the children into it (conditions AND together). Documented but
  not yet tried here: `etch.blocks.replace(conditionId, json)` with the new
  `conditionString` (returns a new id; the children come from the json you pass).
  Loop targets *can* be patched: `update(loopId, { target: "data('x', 'en', 3)" })`.
- **Expression syntax** (conditions, loop targets, text): methods are called with
  parentheses — `items.length() > 0`, `title.toSlug()`,
  `price.numberFormat(0, ",", " ")`. Write `.length()`, not `.length`, to match
  what the builder itself authors. `&&` / `!` / `===` work.
- **Component props can take whole objects.** Declare
  `{ type: { primitive: "object" } }` and pass the loop item through:
  `attributes: { testimonial: "{t}" }`; inside the component read
  `{props.testimonial.title}`, loop `props.testimonial.paragraphs`, etc.
  Verified end-to-end on the published Etch Studio site, 2026-11-04. This is
  the clean way to turn a loop's card into a component: the loop stays on the
  page, the card becomes `etch/component` with one object prop. A boolean prop
  set to `"false"` on an instance works in `conditionString`
  (`props.testimonial.hasProject && props.showProject`).
- **Checking responsive layouts without CDP.** Bridge mode can't screenshot. For
  a visual pass, open the **published** page in your own Chrome tab (Claude in
  Chrome), never the builder canvas (the connector forbids touching it). To
  preview unpublished CSS/markup, inject it into that live-page tab with
  `javascript_tool` — it's a throwaway copy, nothing is saved. Chrome won't
  shrink a window below ~600 px, so test phone widths in an iframe:
  `document.body.innerHTML = '<iframe src="/same/path/" style="width:390px;height:844px">'`,
  then read `contentWindow.getComputedStyle(...)` and screenshot. Scroll the
  iframe only after its images load, or it lands in the wrong place.
- **Heading colours on an ACSS site: set ACSS's variables, not `color` on the
  heading.** (NormadicStudio, 2026-11-07.) ACSS colours headings itself:
  `h1…h6 { color: var(--heading-color) }`, and inside a `.bg--light`/`.bg--dark`
  etc. section `.bg--light :where(h1…h6) { color: var(--relative-heading-color) }`.
  None of it is layered, and `acss-framework-*.css` loads **after**
  `etch-block-styles.css`. So a heading's own one-class rule ties on specificity
  (0,1,0) and loses on order, and a `color` set on a parent card never reaches
  the heading. That is why the user ended up reaching for `!important`. For a
  dark card or panel, set the variables on the container instead; headings
  inside it pick them up:
  `--heading-color: var(--bg-ultra-dark-heading); --relative-heading-color: var(--bg-ultra-dark-heading); --relative-text-color: var(--bg-ultra-dark-text); color: var(--bg-ultra-dark-text);`
- **CSS that compiles fine inside `etch.styles` `css` strings** (verified on the
  published frontend): nested `@media`, `@container name (min-width: …)`,
  `container: name / inline-size`, `@starting-style`, `:has()`,
  `&:not(:has(...))`, `.ancestor:is(:hover, :focus-visible) &`,
  `hsl(from var(--base) h s l / 30%)`, `color-mix()`. A container query can't
  style the element it measures — add an inner wrapper and query from it.
- **`addClass()` silently fails in the same eval as `move()`/`removeClass()` on
  section/container blocks** (Etch WP, 2026-11-21). Renaming or
  moving a section's/container's class in one script left the block with NO class,
  also after `saveAsync()`. Doing the `addClass()` calls alone in a second eval
  worked. Restructure in one eval, add classes in the next, then read back
  `getJson(id).attributes.class` and check the frontend HTML.
  **`getJson()` is not proof it saved:** on the homepage the same day, classes added in
  the second eval showed in `getJson()` but were NOT in the stored post after
  `saveAsync()` (REST `wp/v2/pages/<id>` content had bare `<section>`s). What fixed it:
  one eval that `removeClass()`es them, then another that `addClass()`es and saves.
  Always confirm against the REST content or frontend HTML, never only the builder.
