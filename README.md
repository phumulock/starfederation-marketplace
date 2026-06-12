# starfederation-marketplace

A [Claude Code](https://code.claude.com/docs) plugin marketplace for building with the
**[Datastar](https://data-star.dev)** hypermedia stack — the SSE-driven, signals-based
framework, its `datastar-go` backend SDK, and Datastar Pro's **Rocket** web-component API.

It ships one plugin, **`datastar-stack`**, which teaches Claude Code to write *idiomatic*
Datastar instead of defaulting to htmx/Alpine/React patterns that look similar but don't work.

---

## What's inside

The `datastar-stack` plugin bundles two complementary [Agent Skills](https://code.claude.com/docs/en/skills):

| Skill | Triggers on | What it covers |
|---|---|---|
| **`datastar`** | `data-*` attributes, `@get`/`@post` actions, `datastar-patch-elements`/`-signals`, the `datastar-go` SDK, or any "SSE-driven UI talking to a Go backend with morph updates" | The core OSS framework (v1.x): the signals + SSE mental model, every `data-*` attribute and modifier, the action API, the SSE wire format, the Go SDK, common UI patterns, and "The Tao of Datastar" design opinions. |
| **`rocket`** | `rocket(tag, def)`, `$$` instance signals, codecs like `string.trim.lower`, `data-if`/`data-for` templates, or "a reactive web component on top of Datastar" | Datastar **Pro's** `rocket()` custom-element API: codec-normalized props, instance-scoped `$$` signals, the `html`/`svg` tagged templates, the full lifecycle (`setup` → `render` → `onFirstRender` → `cleanup`), and worked component examples. |

The skills are designed to be used **together** when a Rocket component talks to a Go SSE backend.

Each skill is progressive-disclosure: a compact `SKILL.md` loads first, and detailed
`references/*.md` files are pulled in only when the task needs them.

### Why this exists

Datastar's model differs enough from htmx and Alpine that an LLM's first instinct is often
*subtly* wrong — Alpine-style `.debounce` instead of `__debounce`, pre-v1 event names like
`datastar-merge-fragments`, or mirroring backend state in client signals. These skills encode
the v1 semantics, the free-core-vs-Pro split, and the framework's design taste so generated
code is correct by default.

---

## Installation

In an interactive Claude Code session:

```text
# 1. Register this marketplace (GitHub owner/repo form)
/plugin marketplace add phumulock/starfederation-marketplace

# 2. Install the plugin
/plugin install datastar-stack@starfederation-marketplace
```

Or browse interactively with `/plugin` (a tabbed UI for **Discover / Installed / Marketplaces / Errors**).

### Non-interactive (settings.json)

To pin the marketplace and enable the plugin via configuration instead of slash commands:

```json
{
  "extraKnownMarketplaces": {
    "starfederation-marketplace": {
      "source": { "source": "github", "repo": "phumulock/starfederation-marketplace" }
    }
  },
  "enabledPlugins": {
    "datastar-stack@starfederation-marketplace": true
  }
}
```

### Managing it

```text
/plugin                # open the UI to enable / disable / uninstall
/plugin uninstall datastar-stack@starfederation-marketplace
```

---

## Usage

Once installed, the skills activate **automatically** when your prompt or the code in context
matches their triggers — you don't invoke them by hand. Ask Claude Code to build a live-search
box in Go, convert an htmx snippet to Datastar, or author a `<tag-input>` Rocket component, and
the relevant skill loads its guidance before any code is written.

To confirm the plugin is active, run `/plugin` and check the **Installed** tab.

---

## Repository structure

```text
.
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest (lists the datastar-stack plugin)
└── datastar-stack/
    ├── plugin.json               # plugin manifest
    └── skills/
        ├── datastar/
        │   ├── SKILL.md
        │   ├── references/       # attributes, actions, sse-events, go-sdk, patterns, tao
        │   └── evals/            # behavioral evals + fixture files
        └── rocket/
            ├── SKILL.md
            ├── references/       # codecs, setup-context, templates, lifecycle, examples
            └── evals/            # behavioral evals
```

### Reference files at a glance

**`datastar`**
- `attributes.md` — every `data-*` attribute, exact modifier syntax, and the free-vs-Pro split
- `actions.md` — `@get`/`@post`/… options, signal filtering, utility actions (`@peek`, `@setAll`, `@toggleAll`)
- `sse-events.md` — the two v1 SSE events and their wire format
- `go-sdk.md` — the `datastar-go` public API with complete handler examples
- `patterns.md` — battle-tested templates (active search, infinite scroll, CQRS real-time, …)
- `tao.md` — the maintainers' design opinions, with the reasoning behind each

**`rocket`**
- `codecs.md` · `setup-context.md` · `templates.md` · `lifecycle.md` · `examples.md`

---

## Scope & accuracy notes

- **Datastar v1.x.** The skills target v1. Pre-v1 names (`datastar-merge-fragments`,
  `datastar-merge-signals`, etc.) are flagged as obsolete throughout.
- **Free core vs. Pro.** Pro-only attributes/actions (`data-persist`, `data-replace-url`,
  `data-view-transition`, `@clipboard`, the entire Rocket API, …) are clearly marked, with
  free-core alternatives where they exist. The `rocket` skill assumes the Pro bundle
  (`/bundles/datastar-pro.js`).
- The Go SDK reference is verified against
  [`starfederation/datastar-go`](https://github.com/starfederation/datastar-go); the frontend
  and Rocket references against the official docs at [data-star.dev](https://data-star.dev).

---

## Links

- Datastar — <https://data-star.dev>
- The Tao of Datastar — <https://data-star.dev/guide/the_tao_of_datastar>
- Datastar Go SDK — <https://github.com/starfederation/datastar-go>
- Datastar Pro / Rocket reference — <https://data-star.dev/reference/rocket>
- Claude Code plugins — <https://code.claude.com/docs/en/discover-plugins>

---

## Contributing

This is a documentation-only plugin — accuracy is the whole product. If you find a claim that
doesn't match the current Datastar release or SDK, please open an issue or PR with a link to the
authoritative source (docs page or SDK source line). The `evals/` directories hold behavioral
checks for the skills; extend them when you add or correct guidance.
