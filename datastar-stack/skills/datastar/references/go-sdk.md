# Datastar Go SDK reference

The reference backend SDK. Repo: `github.com/starfederation/datastar-go`. Requires Go 1.24+.

This file documents the public API surface and shows complete, realistic handler examples.

## Install

```bash
go get github.com/starfederation/datastar-go
```

```go
import "github.com/starfederation/datastar-go/datastar"
```

## Public surface

### `datastar.ReadSignals(r *http.Request, dst any) error`

Reads the current client signals into a Go struct (or `map[string]any`).

- For `GET` and `DELETE` requests, signals are parsed from the `datastar` query parameter.
- For `POST`/`PUT`/`PATCH`, signals are parsed from the JSON request body.

```go
type Store struct {
    Query string `json:"query"`
    Page  int    `json:"page"`
}

var s Store
if err := datastar.ReadSignals(r, &s); err != nil {
    http.Error(w, err.Error(), http.StatusBadRequest)
    return
}
```

The `json:"…"` tags must match the **client-side signal names exactly** (after Datastar's auto-camelCase conversion). If the client has `$userName`, the field tag is `json:"userName"`.

### `datastar.NewSSE(w http.ResponseWriter, r *http.Request, opts ...SSEOption) *ServerSentEventGenerator`

Opens the SSE response. Sets `Content-Type`, `Cache-Control`, and `Connection` headers; flushes the response writer. Returns a `*ServerSentEventGenerator` handle for the rest of the handler. (Examples below use the receiver variable name `sse`.)

```go
sse := datastar.NewSSE(w, r)
```

Call this **once** per handler. Subsequent calls on the same `w` will conflict with the streaming response.

> **All emitter methods return `error`.** `PatchElements`, `MarshalAndPatchSignals`, `PatchSignals`, `RemoveElement`, `ExecuteScript`, and `Redirect` each return an `error` (a write/flush failure, usually a disconnected client) and accept trailing variadic options. The examples below discard the error for brevity; in production, check it — once a write fails the stream is dead and you should stop emitting.

### `(*ServerSentEventGenerator).PatchElements(elements string, opts ...PatchElementOption) error`

Emits a `datastar-patch-elements` event. Default mode is `outer` (morph by id of the top-level element in `elements`).

```go
sse.PatchElements(`<div id="results"><ul><li>match</li></ul></div>`)
```

Common options:

| Option | Effect |
|---|---|
| `datastar.WithSelector("#list")` | CSS selector target. Required for `inner`, `replace`, `append`, `prepend`, `before`, `after`, `remove`. |
| `datastar.WithMode(datastar.ElementPatchModeInner)` | Replace innerHTML of selector. |
| `datastar.WithMode(datastar.ElementPatchModeAppend)` | Append into selector. |
| `datastar.WithMode(datastar.ElementPatchModeRemove)` | Delete element at selector. |
| `datastar.WithUseViewTransitions(true)` | Wrap the morph in a View Transition. |
| `datastar.WithNamespace(datastar.NamespaceSVG)` | For SVG/MathML inserts (`NamespaceHTML`, `NamespaceSVG`, `NamespaceMathML`). |

Other `ElementPatchMode` constants: `ElementPatchModeOuter` (default), `ElementPatchModePrepend`, `ElementPatchModeBefore`, `ElementPatchModeAfter`, `ElementPatchModeReplace`. There are also mode sugar options that skip the `WithMode(...)` wrapper — `WithModeInner()`, `WithModeAppend()`, `WithModeRemove()`, etc.

### `(*ServerSentEventGenerator).RemoveElement(selector string, opts ...PatchElementOption) error`

Shorthand for a `mode: remove` patch:

```go
sse.RemoveElement("#notification")
```

Equivalent to `sse.PatchElements("", datastar.WithSelector("#notification"), datastar.WithMode(datastar.ElementPatchModeRemove))`.

### `(*ServerSentEventGenerator).MarshalAndPatchSignals(signals any, opts ...PatchSignalsOption) error`

JSON-encodes `signals` and emits a `datastar-patch-signals` event.

```go
sse.MarshalAndPatchSignals(map[string]any{
    "count":      newCount,
    "user.email": "a@b.com",
    "_fetching":  false,
})
```

Or with a struct:

```go
sse.MarshalAndPatchSignals(struct {
    Count int `json:"count"`
    User  struct {
        Email string `json:"email"`
    } `json:"user"`
}{Count: 42, User: ...})
```

To remove a signal, encode `nil`:

```go
sse.MarshalAndPatchSignals(map[string]any{
    "_tempFlag": nil,
})
```

If you already have raw JSON bytes, use `PatchSignals(signals []byte, opts ...PatchSignalsOption) error` directly. There's also `MarshalAndPatchSignalsIfMissing` for "don't clobber existing values" (the SDK side of the wire `onlyIfMissing` flag).

### `(*ServerSentEventGenerator).ExecuteScript(script string, opts ...ExecuteScriptOption) error`

Convenience for injecting a `<script>` into `<body>` via `PatchElements`. The browser executes it on insertion.

```go
sse.ExecuteScript(`alert('Saved!')`)
```

### `(*ServerSentEventGenerator).Redirect(path string, opts ...ExecuteScriptOption) error`

Triggers a client-side navigation. Implemented as a script execution.

```go
sse.Redirect("/dashboard")
```

---

## Complete handler examples

### Active search

```go
type SearchStore struct {
    Query string `json:"query"`
}

func searchHandler(w http.ResponseWriter, r *http.Request) {
    var s SearchStore
    if err := datastar.ReadSignals(r, &s); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    results := findMatches(s.Query) // your domain code

    sse := datastar.NewSSE(w, r)

    var b strings.Builder
    b.WriteString(`<ul id="results">`)
    for _, r := range results {
        fmt.Fprintf(&b, `<li>%s</li>`, html.EscapeString(r))
    }
    b.WriteString(`</ul>`)

    sse.PatchElements(b.String()) // mode=outer, morphs #results
}
```

Frontend:

```html
<input data-bind:query
       data-on:input__debounce.300ms="@get('/search')" />
<ul id="results"></ul>
```

### Form submit with loading state and validation errors

```go
type SaveStore struct {
    Email string `json:"email"`
    Name  string `json:"name"`
}

func saveHandler(w http.ResponseWriter, r *http.Request) {
    var s SaveStore
    if err := datastar.ReadSignals(r, &s); err != nil {
        http.Error(w, err.Error(), 400)
        return
    }

    sse := datastar.NewSSE(w, r)

    // Validate
    errs := map[string]string{}
    if !strings.Contains(s.Email, "@") {
        errs["email"] = "Invalid email"
    }
    if s.Name == "" {
        errs["name"] = "Required"
    }

    if len(errs) > 0 {
        sse.MarshalAndPatchSignals(map[string]any{
            "errors":    errs,
            "_fetching": false,
        })
        return
    }

    // Save
    if err := db.Save(s); err != nil {
        sse.MarshalAndPatchSignals(map[string]any{
            "errors":    map[string]string{"_form": err.Error()},
            "_fetching": false,
        })
        return
    }

    sse.MarshalAndPatchSignals(map[string]any{
        "errors":    nil,         // clear any prior errors
        "_fetching": false,
    })
    sse.PatchElements(`<div id="status">Saved.</div>`)
}
```

Frontend:

```html
<form id="form"
      data-signals="{errors: {}}"
      data-on:submit__prevent="@post('/save')">
    <input data-bind:name />
    <p data-show="$errors.name" data-text="$errors.name" style="color:red"></p>

    <input data-bind:email />
    <p data-show="$errors.email" data-text="$errors.email" style="color:red"></p>

    <button data-indicator:_fetching
            data-attr:disabled="$_fetching"
            type="submit">Save</button>
</form>
<div id="status"></div>
```

### Infinite scroll

```go
type FeedStore struct {
    Page int `json:"page"`
}

func feedHandler(w http.ResponseWriter, r *http.Request) {
    var s FeedStore
    _ = datastar.ReadSignals(r, &s)
    if s.Page < 1 {
        s.Page = 1
    }

    items := loadFeedPage(s.Page) // returns []Item
    nextPage := s.Page + 1
    hasMore := len(items) > 0

    sse := datastar.NewSSE(w, r)

    // Append new items
    var b strings.Builder
    for _, it := range items {
        fmt.Fprintf(&b, `<article><h3>%s</h3></article>`, html.EscapeString(it.Title))
    }
    sse.PatchElements(b.String(),
        datastar.WithSelector("#feed"),
        datastar.WithMode(datastar.ElementPatchModeAppend))

    // Replace the sentinel with one pointing at the next page (or remove it)
    if hasMore {
        sse.PatchElements(fmt.Sprintf(
            `<div id="sentinel" data-on-intersect__once="@get('/feed?page=%d')">Loading…</div>`,
            nextPage))
    } else {
        sse.RemoveElement("#sentinel")
    }
}
```

Frontend (initial render):

```html
<div id="feed">…</div>
<div id="sentinel" data-on-intersect__once="@get('/feed?page=2')">Loading…</div>
```

### Real-time push (server-driven updates)

```go
func liveHandler(w http.ResponseWriter, r *http.Request) {
    sse := datastar.NewSSE(w, r)

    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-r.Context().Done():
            return
        case <-ticker.C:
            stats := currentStats() // your domain code
            sse.MarshalAndPatchSignals(map[string]any{
                "online": stats.OnlineUsers,
                "lag":    stats.LagMs,
            })
        }
    }
}
```

Frontend:

```html
<div data-on:load="@get('/live', {openWhenHidden: true})"></div>
<p>Online: <span data-text="$online"></span></p>
<p>Lag: <span data-text="$lag"></span> ms</p>
```

`openWhenHidden: true` keeps the stream alive when the tab is backgrounded — critical for true real-time UIs.

### Optimistic UI with rollback on error

```go
type IncrementStore struct {
    Count int `json:"count"`
}

func incHandler(w http.ResponseWriter, r *http.Request) {
    var s IncrementStore
    _ = datastar.ReadSignals(r, &s)

    sse := datastar.NewSSE(w, r)

    newCount, err := db.Increment()
    if err != nil {
        // Roll back the optimistic update
        sse.MarshalAndPatchSignals(map[string]any{
            "count": s.Count - 1,
            "error": err.Error(),
        })
        return
    }

    // Authoritative value (may match the optimistic one)
    sse.MarshalAndPatchSignals(map[string]any{
        "count": newCount,
    })
}
```

Frontend:

```html
<button data-on:click="$count++; @post('/inc')">+</button>
<p data-text="$count"></p>
```

The `$count++` runs synchronously and updates the UI immediately; the `@post` runs after. The server response then patches the authoritative value.

---

## Streaming patterns

For long-running responses (search-with-progress, AI streaming, etc.), call `PatchElements` / `MarshalAndPatchSignals` repeatedly during the request. Each call writes a separate SSE event and flushes. The client sees them as they arrive.

```go
sse := datastar.NewSSE(w, r)

sse.PatchElements(`<div id="status">Loading…</div>`)

for chunk := range streamFromUpstream() {
    sse.PatchElements(fmt.Sprintf(
        `<span class="chunk">%s</span>`,
        html.EscapeString(chunk)),
        datastar.WithSelector("#output"),
        datastar.WithMode(datastar.ElementPatchModeAppend))
}

sse.PatchElements(`<div id="status">Done.</div>`)
```

The handler returns when the stream is exhausted, which closes the connection.

---

## HTML escaping

Datastar does not escape HTML in `PatchElements`. Whatever you pass goes into the DOM verbatim. **Always** escape user-controlled content with `html.EscapeString` (or use a templating package that escapes by default — `html/template`, `templ`, etc.).

```go
// UNSAFE
sse.PatchElements(fmt.Sprintf(`<p>Hello %s</p>`, s.Name))

// Safe
sse.PatchElements(fmt.Sprintf(`<p>Hello %s</p>`, html.EscapeString(s.Name)))
```

The same applies to signal values that get echoed back to other clients — escape on the way in or on the way out, but always before they reach `PatchElements`.

---

## Tips for routing

Datastar makes no assumptions about your router. With `net/http`:

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /search", searchHandler)
mux.HandleFunc("POST /save", saveHandler)
mux.HandleFunc("GET /feed", feedHandler)
mux.HandleFunc("GET /live", liveHandler)
http.ListenAndServe(":8080", mux)
```

With `chi`, `gin`, `echo`, `gorilla/mux` — same idea. The handler signature is `func(http.ResponseWriter, *http.Request)`.

For dev-time autoreload, pair with `air`, `wgo`, or `templ` (if you're using `templ` for HTML rendering, it has a `--watch` mode that plays nicely with Datastar's SSE reconnection).
