# Etch Connector — CLI reference, usage, security

Source: https://docs.etchwp.com/etch-connector/ (usage, cli-reference, security,
ai-agent-guide). You run `etch-connector` via `npx @digital-gravy/etch-connector`;
no global install needed.

## Commands

```
etch-connector serve [--ws-port 7331] [--control-port 7332] [--ws-host 127.0.0.1]
etch-connector tabs [--json] [--control-port 7332] [--cdp]
etch-connector eval [code] [-t|--tab name] [-f|--file path] [--timeout ms] [--cdp]
etch-connector shot [-t name] [-s|--selector css] [--full] [--jpeg] [-o|--out file] [--freeze=false]   (CDP)
etch-connector html <selector> [-t name]                                                              (CDP)
etch-connector computed <selector> [-t name] [--props a,b,c]                                           (CDP)
```

### `serve`
Starts the connector and keeps it running — the tab connects to this. Run in the
background; don't block waiting on it.

| Option | Default | Notes |
|---|---|---|
| `--ws-port` | 7331 | port the Etch tab connects on; change if in use |
| `--control-port` | 7332 | port other commands use to talk to the running connector |
| `--ws-host` | 127.0.0.1 | leave as-is unless you have a specific reason not to |

### `tabs`
Lists connected tabs. Use `--json` for parseable output. Use this to confirm the
connection landed, and to get the tab name for `-t` when multiple sites are
connected at once.

### `eval`
Runs a script in a connected tab — this is how you actually make changes.

| Option | Notes |
|---|---|
| `-t`, `--tab` | which tab; optional if only one is connected |
| `-f`, `--file` | run a script from a file instead of inline `code` |
| `--timeout` | ms to wait before giving up |

Output contract:
- The script's return value is printed as the main output (JSON).
- Any `console.log`/etc. the script produced is printed separately.
- Exit code: `0` success, `2` the script threw (check the error), `1` operational
  failure (tab not found / timed out / connector unreachable — not a script bug).

The eval body is the **body of an async function** — write plain statements, use
`await`, and `return` a value if you want one back.

### CDP mode (screenshots & inspection — optional, advanced)
A second mode that talks to Chrome DevTools Protocol instead of the Etch tab
directly. Needs Chrome started with `--remote-debugging-port=9222`, plus `--cdp` on
the command. Useful for an agent to visually verify its own work:

```
etch-connector shot --cdp -t etch -s .orbit -o orbit.png
etch-connector computed --cdp -t etch -s .orbit__chip --props transform,animation-duration
```

- `shot` — screenshot the page or one element (`-s` selector, `--full` for full
  page, `--jpeg`, `-o` output file).
- `html <selector>` — rendered HTML for an element.
- `computed <selector> [--props a,b,c]` — final computed CSS for an element.

## Connecting (what to tell the user)

1. One-time: builder → Settings → AI → turn on **AI Connector**.
2. Open the target page/template **in the Etch builder** and leave the tab open.
3. User clicks the **AI sparkles button** in the builder toolbar.
4. You run `npx @digital-gravy/etch-connector serve` (background). Once you confirm
   via `tabs`, the sparkles button lights up on their end.

Connection lifetime:
- Stays alive as long as **the chat where `serve` was run** stays open — new chats
  can reuse the same running connector without re-running `serve`.
- Ends if that original chat closes, or if the builder tab is closed/navigated away.
- One connected tab per site — never connect two tabs on the same site
  simultaneously. Multiple *different* sites can each have their own connected tab.

Troubleshooting:
- Sparkles button missing → AI Connector setting is off; point the user at
  Settings → AI.
- Was working, now unreachable → the original chat that ran `serve` was closed;
  run `serve` again (in any chat) to restart the bridge.

## Security model

- Listens on `127.0.0.1` only — not reachable from other devices or the network.
- Scripts run in **safe mode**: `etch.*` plus standard JS only. `window`,
  `document`, network requests, and browser storage are blocked.
- This is a guardrail against accidents, not a security boundary against a
  deliberately malicious script — treat the connector like any other local dev
  tool and only run scripts/agents you trust.
