# Focus Layer Claude Plugins

A [Claude plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugins)
published by [Focus Layer](https://github.com/sprinter-tech). Add it once and
install any plugin below into **Claude Code** or **Claude Desktop**; installed
plugins auto-update from the marketplace at startup.

## Add the marketplace

**Claude Code**

```
/plugin marketplace add focus-layer/claude-plugins
/plugin install <plugin>@focus-layer
```

**Claude Desktop** — Settings → Extensions → **Add marketplace** → enter
`focus-layer/claude-plugins`, then install a plugin from it.

## Plugins

| Plugin                                          | What it does                                |
|-------------------------------------------------|---------------------------------------------|
| [`network-diagnostics`](./network-diagnostics/) | Network diagnostics + device ID (read-only) |

`network-diagnostics` provides read-only investigation skills — Wi-Fi link
health, device/printer triage, switch-port location, and network issue reports —
backed by Focus Layer MCP tools. See its own [`README.md`](./network-diagnostics/)
for full details; each plugin documents itself in its subdirectory.

## Layout

This repo is a marketplace catalog plus one subdirectory per plugin:

```
claude-plugins/
├── .claude-plugin/
│   └── marketplace.json        catalog — lists every plugin (at the repo root)
└── network-diagnostics/        a plugin payload (source: "./network-diagnostics")
    ├── .claude-plugin/plugin.json
    ├── skills/
    ├── .mcp.json
    └── README.md
```

The published tree is generated from the
[Focus Layer monorepo](https://github.com/sprinter-tech) (`plugins/`) by its
`make publish-plugin` target — edit there, not here.
