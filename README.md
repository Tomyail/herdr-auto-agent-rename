# auto-agent-rename

A [Herdr](https://herdr.dev) plugin that **automatically renames agent sidebar rows from their tab labels**. Name a tab, and every agent in that tab shows the same name in the sidebar — including CJK text, spaces, and mixed case, which `herdr agent rename`'s `[a-z0-9_-]` naming rules do not allow.

```
tab label:  "Fix flaky egress test"       (typed by you, or written by a tab auto-namer)
sidebar:    agent row  ->  "Fix flaky egress test"
```

One-way only: the tab label is the source of truth. The plugin never renames tabs, so it composes cleanly with tab auto-naming plugins such as [qu8n/herdr-automatic-rename](https://github.com/qu8n/herdr-automatic-rename) — that plugin decides tab names, this one relays them onto agent rows.

## Install

```sh
herdr plugin install tomyail/herdr-auto-agent-rename
herdr server reload-config
```

Requires herdr >= 0.9.0 and `jq` on `PATH`. Linux and macOS.

## What it does

| Event | Behavior |
| --- | --- |
| `tab.renamed` | Every agent pane in that tab gets its sidebar display name set to the tab label. |
| `pane.agent_detected` | An agent detected in an already-named tab picks the label up immediately. |
| startup | All existing named tabs are backfilled after a server restart or live handoff. |

Rules:

- Labels that are purely digits (unnamed tabs showing their number) are skipped.
- A leading jump-key prefix written by numbering plugins — `[3] Fix flaky test` — is stripped, so the sidebar shows `Fix flaky test`.
- Writes are idempotent: an unchanged display name is not rewritten.
- Plain terminal panes without an agent are never touched.
- Display names are set via `herdr pane report-metadata --source auto-agent-rename`, i.e. display-only metadata. The agent's CLI addressable name is not modified.

To undo the display names:

```sh
herdr pane report-metadata <pane-id> --source auto-agent-rename --clear-display-agent
```

## Configuring the name format

The display name is controlled by a template and ordered `sed -E` rewrites in:

```
~/.config/herdr/plugins/config/tomyail.auto-agent-rename/config.sh
```

```bash
# Tokens:
#   {label}  the tab label, jump-key prefix stripped, REWRITES applied
#   {agent}  agent kind (pi / claude / codex / ...)
#   {cwd}    basename of the pane's working directory
#   {space}  the pane's workspace label, jump-key prefix stripped
#   {tab}    tab id (e.g. w1S:tC)
FORMAT='{label}'
REWRITES=()
```

Examples:

```bash
FORMAT='{label} · {agent}'   # "Fix flaky egress test · pi"
FORMAT='{space}/{label}'     # "yt-audio/Fix flaky egress test"

# Strip a "context › " prefix added by herdr-automatic-rename's TAB_CONTEXT:
REWRITES=(
  's|^[^›]+› ||'             # "api › fix-auth" -> "fix-auth"
)
```

Changes apply on the next rename event, or force a full pass:

```sh
herdr tab rename <tab-id> <same-label>   # re-fire tab.renamed
```

## Uninstall

```sh
herdr plugin uninstall tomyail.auto-agent-rename
```

Run the `--clear-display-agent` command above for each affected pane first if you want the sidebar names reverted rather than left in place.

## License

MIT
