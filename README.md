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
npx serve .        # http://localhost:3000 — the root page is the demo
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
| Attributes | `placeholder`, `readonly` |
| Property | `value` — the markdown source (getter + setter) |
| Events | `sac:input` (once per edit) and `sac:change` (focus leaves after an edit), `detail: { value }`, bubbling, not composed — the kit convention. Native `input` / `change` still fire for older hosts |
| Methods | `focus()` |
| Static | `registerBlock(def)` → unregister function · `unregisterBlock(name)` · `blocks` — see [Blocks](#blocks-registerblock) |
| Keyboard | Ctrl/Cmd+B/I/K · Enter continues lists · Tab soft-tabs · Backspace merges lines · in a table: Tab / Shift+Tab move between cells, Enter adds a row, Esc leaves the table |
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
codebase must keep.

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

## Vendoring

`kit/` is the vendored SACRVM APPKIT (version in `kit/VERSION`), used by the
demo page and as the token source. Per the kit's consuming rules it is copied
**verbatim and never edited here** — to upgrade, delete the folder and unzip
the next release.

## Licence

MIT — see `LICENSE`. Vendored: SACRVM APPKIT (MIT), marked (MIT),
DOMPurify (Apache-2.0/MPL-2.0).
