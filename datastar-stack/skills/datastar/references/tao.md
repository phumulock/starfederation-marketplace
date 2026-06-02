# The Tao of Datastar

Source: <https://data-star.dev/guide/the_tao_of_datastar>

> Datastar is just a tool. The Tao of Datastar, or "the Datastar way" as it is often referred to, is a set of opinions from the core team on how to best use Datastar to build maintainable, scalable, high-performance web apps.

The Tao is design taste, not API rules. The framework will let you violate every one of these — they exist because the maintainers have seen what happens when you do. Use this reference when you're making a design call (where state lives, whether to optimistically update, how navigation works, whether to manage history) and want the original reasoning, not just the bullet.

---

## 1. State in the Right Place

> Most state should live in the backend. Since the frontend is exposed to the user, the backend should be the source of truth for your application state.

**Implication:** When you catch yourself adding a signal that mirrors a database row or any other server-known fact, that signal is in the wrong place. The signal store is for *ephemeral UI state* and *pending input on its way to the backend* — not for caching what the backend already knows.

## 2. Start with the Defaults

> The default configuration options are the recommended settings for the majority of applications. Start with the defaults, and before you ever get tempted to change them, stop and ask yourself, well... how did I get here?

**Implication:** Default patch mode (`outer` morph by id), default content type (JSON for non-GET), default signal-filter behavior (send everything) — these are calibrated for the common case. Reaching for an option to override a default is a small alarm bell. Ask whether the design upstream of the override is the actual problem.

## 3. Patch Elements & Signals

> Since the backend is the source of truth, it should drive the frontend by patching (adding, updating and removing) HTML elements and signals.

**Implication:** The flow is **action → backend → SSE patches → UI**. Not **frontend computes new state → frontend updates UI → backend gets notified**. If you find yourself mutating signals to reflect what *should* be the result of a backend action, you're inverting the model.

## 4. Use Signals Sparingly

> Overusing signals typically indicates trying to manage state on the frontend. Favor fetching current state from the backend rather than pre-loading and assuming frontend state is current. A good rule of thumb is to only use signals for user interactions (e.g. toggling element visibility) and for sending new state to the backend (e.g. by binding signals to form input elements).

**Implication:** Two legitimate uses for a signal:
1. **User interaction** — `$menuOpen`, `$tabSelected`, `$_showAdvanced`. Local UI state that has no backend meaning.
2. **Outbound binding** — `data-bind:email`, `data-bind:query`. Input on its way to the server.

Anything else (a signal that caches a list of items, a signal that mirrors server config) is suspect. Patch the DOM instead.

## 5. In Morph We Trust

> Morphing ensures that only modified parts of the DOM are updated, preserving state and improving performance. This allows you to send down large chunks of the DOM tree (all the way up to the html tag), sometimes known as "fat morph", rather than trying to manage fine-grained updates yourself.

**Implication:** Don't pre-optimize by computing the minimal DOM patch on the backend. Render a "fat" chunk — even the whole page — and let morph figure out what changed. It preserves focus, input values, and scroll position; it's smarter than your hand-tuned diff will be; and it's almost always fast enough.

## 6. SSE Responses

> SSE responses allow you to send 0 to n events, in which you can patch elements, patch signals, and execute scripts. Since event streams are just HTTP responses with some special formatting that SDKs can handle for you, there's no real benefit to using a content type other than text/event-stream.

**Implication:** Even when you only want to emit *one* patch, use SSE. Don't reach for a JSON endpoint that the frontend then has to interpret. The whole point is that one transport handles everything from a single-event response to an indefinitely open server-push stream — uniformly.

## 7. Compression

> Since SSE responses stream events from the backend and morphing allows sending large chunks of DOM, compressing the response is a natural choice. Compression ratios of 200:1 are not uncommon when compressing streams using Brotli.

**Implication:** The "fat morph" model (principle 5) only stays cheap if the wire is compressed. Brotli on `text/event-stream` is the default Tao stack. If you're seeing bandwidth concerns from morphed HTML, the answer is almost always "enable compression," not "stop sending fat morphs."

## 8. Backend Templating

> Since your backend generates your HTML, you can and should use your templating language to keep things DRY (Don't Repeat Yourself).

**Implication:** Components/partials/macros in your backend template language are the unit of reuse. Don't push that responsibility into JS-side rendering. In Go, that's `html/template`, `templ`, `gomponents`, etc.

## 9. Page Navigation

> Page navigation hasn't changed in 30 years. Use the anchor element (`<a>`) to navigate to a new page, or a redirect if redirecting from the backend. For smooth page transitions, use the View Transition API.

**Implication:** No SPA-style router. No `data-on:click="@get('/page')"` for what should be a link. A real `<a href>` gives you middle-click, right-click "open in new tab," keyboard focus, screen-reader semantics, and browser-native back/forward — for free. For the polish that SPAs claim as their reason to exist, use the View Transition API on top of normal navigation.

## 10. Browser History

> Browsers automatically keep a history of pages visited. As soon as you start trying to manage browser history yourself, you are adding complexity. Each page is a resource. Use anchor tags and let the browser do what it is good at.

**Implication:** Resist the urge to call `history.pushState` to "feel like an SPA." Every page that has a back-button-meaningful state should be its own URL with its own server-rendered response. The browser already knows how to do this.

## 11. CQRS

> CQRS, in which commands (writes) and requests (reads) are segregated, makes it possible to have a single long-lived request to receive updates from the backend (reads), while making multiple short-lived requests to the backend (writes). It is a powerful pattern that makes real-time collaboration simple using Datastar.

**Implication:** The canonical real-time Datastar shape is: open one long-lived SSE stream on page load (`data-on:load="@get('/stream')"` or similar) that the server keeps open to push updates from any source; do all user actions as separate short POSTs that mutate state and return immediately. The reads stream catches up the user (and every other connected user) without coupling reads to writes.

## 12. Loading Indicators

> Loading indicators inform the user that an action is in progress. Use the data-indicator attribute to show loading indicators on elements that trigger backend requests. When using CQRS, it is generally better to manually show a loading indicator when backend requests are made, and allow it to be hidden when the DOM is updated from the backend.

**Implication:** For normal request/response, `data-indicator:_loading` handles itself. Under CQRS, the *write* request returns immediately — but the *visible result* doesn't appear until the *read* stream pushes the corresponding patch. The auto-toggle would clear too early. Manually set a signal on click, and have the server-pushed patch clear it.

## 13. Optimistic Updates

> Optimistic updates (also known as optimistic UI) are when the UI updates immediately as if an operation succeeded, before the backend actually confirms it. It is a strategy used to makes web apps feel snappier, when it in fact deceives the user. Rather than deceive the user, use loading indicators to show the user that the action is in progress, and only confirm success from the backend.

**Implication:** This is the most opinionated principle in the Tao and worth flagging explicitly when the user asks for optimistic UI. The Datastar position is: latency is real; honesty is better than the illusion of speed. If a request feels slow enough that an optimistic update is tempting, address the latency or set the user's expectation with a clear indicator — don't lie about the outcome.

## 14. Accessibility

> The web should be accessible to everyone. Datastar stays out of your way and leaves accessibility to you. Use semantic HTML, apply ARIA where it makes sense, and ensure your app works well with keyboards and screen readers.

**Implication:** The framework will not add `aria-*` attributes for you, will not manage focus on patched content for you, and will not announce SSE-driven updates to screen readers for you. When emitting patches that change page structure, *you* are responsible for keeping focus sensible (often: render `autofocus` or `data-attr:tabindex` on the newly-patched element) and for `aria-live` regions where appropriate.
