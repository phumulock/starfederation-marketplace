---
name: datastar
description: Build features with the Datastar hypermedia framework (data-star.dev) — the SSE-driven, signals-based alternative to htmx and Alpine. Use whenever you're writing or modifying code that involves Datastar, `data-*` reactivity attributes, the `datastar-patch-elements` or `datastar-patch-signals` SSE events, a Datastar backend SDK (especially `datastar-go`), or any UI pattern that talks to a Go backend over Server-Sent Events with morph-based DOM updates. Trigger on phrases like "datastar", "data-bind", "data-on:click", "patch elements", "patch signals", "@get / @post action", "SSE-driven UI", "hypermedia", or when the project loads `datastar.js`. Trigger even if the user just mentions a related concept (signals, SSE morphing, hypermedia) without saying "datastar" by name — Datastar's mental model differs enough from htmx/Alpine that defaulting to those frameworks' patterns will produce subtly wrong code.
---

# Datastar

Datastar is a ~12 KiB hypermedia framework that fuses **htmx-style backend-driven HTML patching** with **Alpine-style frontend reactivity** into a single tool, transported over **Server-Sent Events**. Backend is "bring your own"; the Go SDK (`github.com/starfederation/datastar-go`) is the reference implementation.

This skill targets **Datastar v1.x**. If you see `datastar-merge-fragments`, `datastar-merge-signals`, `datastar-remove-fragments`, `datastar-remove-signals`, or `datastar-execute-script` anywhere — in old docs, blog posts, or your own first instinct — those are **pre-v1 names and no longer exist**. v1 collapsed them into just two events: `datastar-patch-elements` and `datastar-patch-signals`. Treat anything else as outdated.

## The mental model

Two halves, one stream:

1. **Frontend signals.** Every `data-signals`, `data-bind`, `data-computed` (and implicitly, anything you reference with `$name`) creates a reactive signal. Attributes like `data-text`, `data-show`, `data-class`, `data-attr`, `data-effect` re-evaluate whenever their referenced signals change. This is Alpine, basically.

2. **Backend over SSE.** Any DOM event handler can invoke an action like `@get('/path')` or `@post('/save')`. Datastar serializes **the current signals** (everything except underscore-prefixed ones, by default) and sends them with the request (JSON body for POST/PUT/PATCH, query string for GET/DELETE). The server responds with an **SSE stream** that interleaves two event types:
   - `datastar-patch-elements` — patch the DOM. Default mode is `outer` morph by `id`. Other modes: `inner`, `replace`, `append`, `prepend`, `before`, `after`, `remove`.
   - `datastar-patch-signals` — deep-merge a JSON object into the client signals store. Setting a key to `null` removes it.

The same stream can emit any mix of those two events, then close. Or it can stay open indefinitely (server-pushed updates).

That's it. Everything else is sugar.

## The Tao of Datastar

The maintainers publish ["The Tao of Datastar"](https://data-star.dev/guide/the_tao_of_datastar) — opinions on how to build with the framework. These aren't mechanics; they're design taste, and they should shape *which* solution you propose before any code is written. Full prose in `references/tao.md`.

1. **State lives in the backend.** The backend is the source of truth. The frontend is user-exposed and untrusted.
2. **Start with the defaults.** Before changing any default option, stop and ask how you got there.
3. **Backend drives the frontend** by patching elements and signals — not the reverse.
4. **Signals sparingly.** Use signals for user interaction (toggles, form binding) and for sending new state *to* the backend. Don't mirror backend state in signals — fetch it.
5. **In morph we trust.** Send large/"fat" DOM chunks (up to and including `<html>`); morph diffs efficiently. Don't hand-tune surgical updates.
6. **SSE for responses.** Patch elements, patch signals, and execute scripts all ride one `text/event-stream`. There's no benefit to another content type for Datastar responses.
7. **Compress streams.** Brotli on morphed HTML streams routinely hits ~200:1.
8. **Use your backend templating language.** Keep HTML DRY there, not in JS.
9. **Plain `<a>` for navigation.** Page navigation hasn't changed in 30 years. Use the View Transition API for polish.
10. **Let the browser own history.** Each page is a resource. Manual history management is added complexity for nothing.
11. **CQRS.** One long-lived read stream for server-pushed updates, short-lived writes for actions. This is what makes real-time collab simple in Datastar.
12. **`data-indicator` for loading.** Under CQRS you'll often want to *manually* show the indicator on action and hide it when the patched DOM arrives.
13. **Don't do optimistic updates.** They deceive the user. Show a loading indicator and confirm from the backend.
14. **Accessibility is yours.** Datastar stays out of the way — semantic HTML, ARIA, keyboard, screen readers.

When the user's instinct conflicts with the Tao (e.g. "let's keep this state on the client to avoid a roundtrip", "let's optimistically update the cart"), don't silently override — flag the Tao position, then defer to the user's call.

## When to use this skill

Use this skill when:
- The project loads `datastar.js` (CDN URL: `https://cdn.jsdelivr.net/gh/starfederation/datastar@…/bundles/datastar.js`).
- You're writing HTML with `data-bind`, `data-on:*`, `data-signals`, `data-text`, `data-show`, `data-computed`, `data-effect`, `data-indicator`, `data-on-intersect`, `data-on-interval`, etc.
- You're writing a Go backend handler that emits SSE for a Datastar frontend (uses `datastar.NewSSE`, `PatchElements`, `MarshalAndPatchSignals`, `ReadSignals`).
- You need to translate an htmx or Alpine pattern into Datastar — the two are similar but **not** interchangeable.

Do **not** silently fall back to htmx attributes (`hx-get`, `hx-target`, `hx-swap`) or Alpine attributes (`x-data`, `x-on`, `x-bind`, `@click`) when the project is Datastar. They will not work.

## Workflow

For any non-trivial Datastar feature:

1. **Decide the signals.** What client state does this feature read or write? Name it (`$count`, `$query`, `$form.email`, `$_fetching`). Underscore-prefixed names like `$_loading` are the convention for "private" UI-local signals.
2. **Decide who initializes each signal.** `data-signals` on a parent element for static defaults; an initial server response patching signals for server-supplied defaults; `data-bind` for form inputs (auto-creates).
3. **Wire the trigger.** Pick the right `data-on:*` attribute and add timing modifiers (`__debounce.300ms`, `__throttle.1s`, `__once`) before adding the action.
4. **Write the backend handler.** `ReadSignals` → do work → open `NewSSE` → emit one or more `PatchElements` / `MarshalAndPatchSignals` calls → return. The SDK handles SSE framing.
5. **Pick the patch mode.** Default `outer` (morph by id) is right ~80% of the time. Use `inner` to replace contents but keep the wrapper. Use `append`/`prepend` for list growth. Use `remove` to delete.
6. **Verify the morph target exists.** If you emit `<div id="results">…</div>` with default mode, there must already be `<div id="results">` somewhere on the page, or the patch lands nowhere visible.

## Cheat sheet — the attributes you'll reach for first

| Need | Attribute |
|---|---|
| Two-way bind input to signal | `data-bind:fieldName` |
| Display a signal as text | `data-text="$name"` |
| Show/hide based on condition | `data-show="$count > 0"` |
| Toggle a class | `data-class:active="$selected"` |
| Set any attribute reactively | `data-attr:disabled="$loading"` |
| Initialize signals | `data-signals="{count: 0, user: {name: ''}}"` |
| Derived value | `data-computed:full="$first + ' ' + $last"` |
| Side effect on signal change | `data-effect="console.log($count)"` |
| Click handler that calls backend | `data-on:click="@post('/save')"` |
| Run on element load | `data-on:load="@get('/init')"` |
| Run when scrolled into view | `data-on-intersect="@get('/more')"` |
| Run on a timer | `data-on-interval__duration.1s="$count++"` |
| Track fetch in flight | `data-indicator:_fetching` |
| Debug — print signal store | `<pre data-json-signals></pre>` |

For the full attribute reference (including every modifier), see `references/attributes.md`.

## Cheat sheet — actions

```html
<!-- Send all signals as JSON body, stream SSE response -->
<button data-on:click="@post('/save')">Save</button>

<!-- Same, but debounced -->
<input data-bind:query
       data-on:input__debounce.300ms="@get('/search')" />

<!-- File uploads need form encoding -->
<input type="file" data-bind:upload
       data-on:change="@post('/upload', {contentType: 'form'})" />

<!-- Limit which signals get sent -->
<button data-on:click="@post('/checkout', {filterSignals: {include: /^cart/}})">
  Checkout
</button>
```

Full actions reference: `references/actions.md`.

## Cheat sheet — Go backend

```go
import "github.com/starfederation/datastar-go/datastar"

type Store struct {
    Query string `json:"query"`
    Count int    `json:"count"`
}

func handler(w http.ResponseWriter, r *http.Request) {
    var s Store
    if err := datastar.ReadSignals(r, &s); err != nil {
        http.Error(w, err.Error(), 400)
        return
    }

    sse := datastar.NewSSE(w, r)

    // Patch DOM: by default this morphs by id (`#results`)
    sse.PatchElements(fmt.Sprintf(
        `<ul id="results"><li>Found %d matches for %q</li></ul>`,
        len(matches(s.Query)), s.Query))

    // Patch signals: deep-merged into client store
    sse.MarshalAndPatchSignals(map[string]any{
        "count": s.Count + 1,
    })
}
```

Full Go SDK reference: `references/go-sdk.md`.

## Critical gotchas

These trip everyone up — internalize them before writing code:

1. **Modifier syntax is `__name`, not `.name`.** Alpine uses dots; Datastar uses double underscores. Modifier *arguments* (durations, thresholds) do use a dot.
   - Right: `data-on:click__debounce.300ms`
   - Wrong: `data-on:click.debounce.300ms`

2. **IDs are load-bearing.** Default patch mode is `outer`, which finds the existing element by `id`. Top-level elements in your patched HTML need stable ids that match the live DOM, or the morph silently lands nowhere useful. If you don't want id-matching, set an explicit `selector` and a different `mode`.

3. **Signals are sent on every request.** Every signal *except* underscore-prefixed ones (the default filter excludes `/(^_|\._)/`) is sent to every endpoint — convenient, but non-underscore state in the store still leaks everywhere. Filter with `{filterSignals: {include: /pattern/}}` for sensitive or large state; use the `$_local` convention for UI-only signals you never want sent.

4. **Implicit signal creation.** Referencing `$tpyo` in any expression silently creates a new signal with value `""`. There's no compile-time check. Use `<pre data-json-signals></pre>` while developing to see the live store.

5. **Two events only.** `datastar-patch-elements` and `datastar-patch-signals`. If you find yourself writing `datastar-merge-fragments` or similar, stop — that's pre-v1 and won't work.

6. **`data-on-load` is not a thing.** It's just `data-on:load`. The `load` is the DOM `load` event going through the generic `data-on` attribute. Same with `change`, `input`, `submit`, `click`, etc. Only `data-on-intersect`, `data-on-interval`, `data-on-signal-patch` are true custom-named attributes.

7. **Auto-camelCase.** `data-bind:user-name` creates `$userName`. Use the camelCase form in expressions. Override with `__case.kebab` / `__case.snake` modifiers if needed.

8. **Removal vs hiding.** `data-show="false"` hides via CSS but keeps the element. To actually remove from DOM, emit a patch with `mode: remove`.

9. **Form encoding default is JSON.** Fine for normal data; not fine for file uploads. Pass `{contentType: 'form'}` to the action when you need multipart.

10. **`data-ignore` is recursive.** Marks the entire subtree as Datastar-inert. If you only want to skip the element itself but keep descendants reactive, use `data-ignore__self`.

## When to consult the reference files

The reference files are loaded into context only when needed — don't read them all upfront. Reach for them when:

- **`references/attributes.md`** — you need to look up a less-common attribute (`data-replace-url`, `data-preserve-attr`, `data-ignore-morph`, `data-ref`), or you need the full modifier list for `data-on`.
- **`references/actions.md`** — you need exact options for `@get`/`@post` (filters, retry, content type, abort behavior), or you need `@peek`, `@setAll`, `@toggleAll`.
- **`references/sse-events.md`** — you're emitting raw SSE without an SDK, or debugging a wire-level problem.
- **`references/go-sdk.md`** — you're writing the Go backend and need method signatures, option types, or full handler examples.
- **`references/patterns.md`** — the user describes a recognizable pattern (active search, infinite scroll, optimistic update, live validation, lazy load, server-pushed real-time). The reference has battle-tested templates that are faster than re-deriving from primitives.
- **`references/tao.md`** — the user is making a design call where Datastar's opinions are load-bearing (where state should live, whether to optimistically update, how to do navigation/history, when CQRS earns its keep). The reference has the maintainers' full prose on each principle.

## Installation snippet

If the project doesn't yet load Datastar:

```html
<script type="module"
        src="https://cdn.jsdelivr.net/gh/starfederation/datastar@v1.0.1/bundles/datastar.js"></script>
```

For production, self-host the same file. There's no advertised first-party npm package — the canonical install is the CDN script (or downloaded bundle).

## Pro features (commercial license)

These attributes/actions require a Datastar Pro license and **won't work in the free core**: `data-persist`, `data-animate`, `data-custom-validity`, `data-match-media`, `data-on-raf`, `data-on-resize`, `data-scroll-into-view`, `data-view-transition`, `data-replace-url`, `data-query-string`, `@clipboard`, `@fit`, `@intl`. Don't suggest them as solutions to a free-core problem; if the user has Pro, they'll say so or the code will already reference these.
