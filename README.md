# sac-md-editor

An Obsidian-style **live-preview markdown editor** as a single zero-dependency
web component — an add-on module for the
[SACRVM APPKIT](https://github.com/SACRVM/sacrvm-appkit), usable on any page
that provides `marked` and `DOMPurify` as globals.

**No mode switch.** The line your caret sits on shows raw markdown source;
every other line shows rendered markdown with the syntax markers dimmed.
**No build step.** One classic deferred script, plain files, F5.

## Try it

```bash
python3 -m http.server 8765   # http://localhost:8765 — the root page is the API page with a live demo
                              # http://localhost:8765/test/ — the round-trip tests
```

## Use it

```html
<link rel="stylesheet" href="kit/css/ui.css">          <!-- or your own tokens -->
<script defer src="kit/js/vendor/marked.min.js"></script>
<script defer src="kit/js/vendor/purify.min.js"></script>
<script defer src="js/sac-md-editor.js"></script>
<script defer src="js/sac-md-secret.js"></script>  <!-- optional: :::secret blocks -->

<sac-md-editor placeholder="Write…"></sac-md-editor>
```

| Surface | |
|---|---|
| Attributes | `placeholder`, `readonly`, `bare` (no frame; the writing surface fills the host's height — for an editor that is the whole pane), `links="host"` (relative hrefs are links, for the host to open — see [Links](#links)) |
| Property | `value` — the markdown source (getter + setter). Lossless: what you set reads back unchanged; only `\r\n` and a lone `\r` read back as `\n` |
| Events | `sac:input` (once per edit) and `sac:change` (focus leaves after an edit), `detail: { value }`, bubbling, not composed — the kit convention. Native `input` / `change` still fire for older hosts. `sac:link` (a link is followed), `detail: { href, text, raw, line }`, bubbling, composed, cancelable — see [Links](#links). `history-change` (`detail: { canUndo, canRedo }`, bubbling, composed) — drive your own Undo / Redo buttons from it |
| Methods | `focus()` · `undo()` / `redo()` (a step is a word, a pause in typing, or one structural edit) · `canUndo()` / `canRedo()` |
| Static | `registerBlock(def)` → unregister function · `unregisterBlock(name)` · `blocks` — see [Blocks](#blocks-registerblock) · `registerInline(def)` → unregister function · `unregisterInline(name)` · `inlines` · `linkSchemes` — see [Links](#links) · `strikethrough` — see [Strikethrough](#strikethrough) |
| Keyboard | Ctrl/Cmd+B/I/K · Enter continues lists · Backspace merges lines · Alt+Enter follows the link at the caret · Tab / Shift+Tab indent / outdent list items and code (over a selection: every line); anywhere else Tab moves the focus on, and Esc then Tab leaves from anywhere · in a table: Tab / Shift+Tab move between cells, Enter adds a row, Esc leaves the table; the Table button in a table opens a grid dialog (add / delete / move columns and rows, alignment) · the toolbar is one tab stop, arrow keys move in it |
| Accessibility | The surface is a multi-line textbox named by the host's `aria-label` (else "Markdown text"), placeholder as `aria-placeholder`; in read-only a masked block's eye is a focusable toggle button. A task's box ticks on a click or tap while editing |
| Language | Follows the page language through the kit's `sac.t` / `sac.lang`, relabelled live; ships German, keys `md-editor.*` |

The editor's own strings (toolbar, reveal toggle, link prompt, the
`:::secret` badge) switch with the page language at runtime, without
re-rendering the document. It registers a German table itself; other
languages go in through `sac.i18n.add(lang, { "md-editor.bold": … })`.
Without the kit's `globals.js` it stays English. `placeholder` is your
string, so translate it on your side.

Styling is Shadow-DOM-scoped but driven by the kit's seed tokens
(`--fg`, `--field`, `--accent`, `--text`, `--border`, `--on-accent`, …), so
the editor rethemes with the page in light and dark. It works without the kit
too — every token read has a dark fallback, or define the properties yourself.

## The one invariant

The canonical document is the **concatenated `textContent` of the line divs**.
Every transformation preserves `textContent` exactly — marker characters stay
in the DOM and get wrapped, never synthesized or deleted. That is what makes
the save/load round-trip trivial, and it is the property every change to this
codebase must keep. `test/index.html` pins it over a corpus of tricky lines;
serve the repo and open `/test/`.

## Blocks (`registerBlock`)

The editor has no block names of its own. A page registers them once, and
every editor on it picks them up, live:

```js
const Editor = customElements.get("sac-md-editor");
Editor.registerBlock({
    name:   "note",
    open:   /^:::note(\s|$)/,
    close:  /^:::end(\s|$)/,
    label:  "Note",              // pill on the opening line (or a function)
    color:  "#3b82f6",           // card tint
    masked: false,               // true: body blurred until revealed
    toggle: false,               // true: eye button on the opening line
    css:    ".line.block-note.block-body { font-style: italic; }",
});
```

Instead of `open` / `close`, `match(src)` may return `"open"`, `"close"` or
`"line"` - the last makes a single line a block of its own, for marker forms.
Blocks are recognised outside code fences only, do not nest, and render
inline markdown in their body. The definition's `css` is injected into the
editor's shadow root, the one way to style a block from outside.

### `:::secret`

`js/sac-md-secret.js` registers the one block the demo ships: lines between
`:::secret` and `:::end` (two or three colons accepted on read) render
blurred, with an eye toggle on the opening line for session-only reveal. Load
it after the editor.

> **Security note:** the blur is a courtesy against shoulder-surfing, not a
> security boundary. The plaintext remains in the DOM and in whatever you save
> from `value`. If secrets must stay out of an index, an agent or a wire, the
> host application has to enforce that server-side. Do not build on the blur.

## Links

The editor follows web links (`http(s)`, `mailto`, `#`) itself: a click in
read-only, Ctrl/Cmd+click while editing — the caret line included, where the
link under the pointer is looked up in the source — Alt+Enter with the caret
in a link, or the "Open" button that shows under the caret there (the way in
on touch). Every follow fires a
cancelable `sac:link` first; `preventDefault()` and open it yourself.

A host's own link forms come in two ways:

```js
const Editor = customElements.get("sac-md-editor");
Editor.linkSchemes = ["note:", "wiki:"];   // page-wide: [x](note:01J…) is a link
// <sac-md-editor links="host">            // per editor: [x](../a.md#why) is one too

Editor.registerInline({                    // the inline twin of registerBlock
    name:   "wikilink",
    match:  /\[\[([^\[\]\n|]+?)(?:\|([^\[\]\n]+?))?\]\]/,
    render: (m) => ({ text: m[2] || m[1], href: `wiki:${m[1]}`, title: m[1] }),
});

editor.addEventListener("sac:link", (e) => {
    const { href, text, raw, line } = e.detail;
    if (!/^https?:/.test(href)) { e.preventDefault(); openRecord(href); }
});
```

A host link (an allowed scheme, or a relative href with `links="host"`) goes
nowhere unless the host opens it. `javascript:`, `data:`, `vbscript:`,
`file:` and `blob:` are refused as schemes. Any href the editor does not allow
still looks like a link but has no target.

A registered inline keeps the invariant: the shown text becomes the anchor,
the rest of the match its hidden markers; shown text that is not in the source
(a title for an id) is drawn over the hidden source. Inlines are left alone in
code spans, code blocks and inside a link's text; the anchor carries
`.inline-<name>`, and the definition's `css` is injected like a block's.

## Strikethrough

GFM strikes `~x~` and `~~x~~` alike, so prose that uses a lone `~` for
"about" — `~5 min (~10 at most)` — gets half a sentence struck through. One
switch, page-wide:

```js
customElements.get("sac-md-editor").strikethrough = "double";   // ~~x~~ only; "gfm" is the default
```

Either way the tildes stay in the source; only the rendering changes.

## Vendoring

Releases are tags (`v1.1.0`, ...) with notes on the
[releases page](https://github.com/SACRVM/sac-md-editor/releases): copy
`js/sac-md-editor.js` (and `js/sac-md-secret.js` if you use it) from the tag
you want. A minor version changes behaviour you or your writers will notice;
a major one breaks the API above.

`kit/` is the vendored SACRVM APPKIT (version in `kit/VERSION`), used by the
demo page and as the token source. Per the kit's consuming rules it is copied
**verbatim and never edited here** — to upgrade, delete the folder and unzip
the next release.

## Licence

MIT — see `LICENSE`. Vendored: SACRVM APPKIT (MIT), marked (MIT),
DOMPurify (Apache-2.0/MPL-2.0).
