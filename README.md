# Etch skill for Claude Code

A [Claude Code](https://claude.com/claude-code) skill that lets Claude build and edit
pages, components, styles and loops in the [Etch](https://etchwp.com?aff=e1d00537)
WordPress page builder. It drives your open builder tab live through the official
**Etch Connector**.

It covers:

- connecting to the builder tab and running scripts against `window.etch`
- the section/container scaffold and one-BEM-class-per-element styling
  (works well with [Automatic.css](https://automaticcss.com/ref/185/))
- components, props, slots, conditions, loops, custom fields and navigation
- the Etch quirks and gotchas found while building real sites with it

## Requirements

- [Claude Code](https://claude.com/claude-code)
- Node.js with `npx` (the connector runs as `npx @digital-gravy/etch-connector`)
- An Etch site with **Settings → AI → AI Connector** turned on

## Install

### Option A: plugin (recommended)

In Claude Code, run:

```
/plugin marketplace add endresol/etch-skill
/plugin install etch@normadic-etch
```

To update later, run `/plugin marketplace update normadic-etch`.

### Option B: copy the skill folder

For all your projects:

```bash
git clone https://github.com/endresol/etch-skill.git
mkdir -p ~/.claude/skills
cp -R etch-skill/plugins/etch/skills/etch ~/.claude/skills/etch
```

For one project only, copy it into `<your-project>/.claude/skills/etch` instead.

Restart Claude Code (or start a new session). The skill shows up as `etch`.

## Use it

1. Open the page or template in the Etch builder and leave the tab open.
2. In Claude Code, ask for what you want, for example *"In Etch, build a hero section
   with a heading, a short intro and two buttons"*.
3. Claude starts the connector. When it asks, click the **AI sparkles button** in the
   builder toolbar to connect the tab.

Claude builds in the canvas and does **not** save unless you tell it to. Review the
changes and click Save yourself, or tell Claude to save.

## Notes

- The Etch public API is `0.x` and experimental. The skill tells Claude to
  feature-detect and to trust `npx @digital-gravy/etch-connector --help` over its
  own notes when they disagree.
- Some notes reflect one person's conventions: BEM classes on every element,
  ACSS tokens, and `etch/svg` for icons. Edit `SKILL.md` to match yours.

## Links

- Etch: https://etchwp.com?aff=e1d00537
- Automatic.css: https://automaticcss.com/ref/185/
- Etch API docs: https://docs.etchwp.com/public-api

The Etch and Automatic.css links are affiliate links. Buying through them supports
this project at no extra cost to you.

Made by [Normadic Studio](https://normadicstudio.com). MIT licensed.
