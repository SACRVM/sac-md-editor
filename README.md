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

<sac-md-editor placeholder="Write…"></sac-md-editor>
```

| Surface | |
|---|---|
| Attributes | `placeholder`, `readonly` |
| Property | `value` — the markdown source (getter + setter) |
| Events | `input` (every edit), `change` (focus leaves after an edit) |
| Methods | `focus()` |
| Keyboard | Ctrl/Cmd+B/I/K · Enter continues lists · Tab soft-tabs · Backspace merges lines |

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

## Masked blocks (`:::secret`)

Lines between `:::secret` and `:::end` (two or three colons accepted on read)
render blurred, with an eye toggle on the boundary line for session-only
reveal.

> **Security note:** the blur is a courtesy against shoulder-surfing, not a
> security boundary. The plaintext remains in the DOM and in whatever you save
> from `value`. If secrets must stay out of an index, an agent or a wire, the
> host application has to enforce that server-side. Do not build on the blur.

Planned: a small **block-type registry**
(`registerBlock({ match, masked, toggle, className })`) so hosts define their
own bounded blocks — spoiler, collapse, callout — and `:::secret` stops being
special-cased. Until then it is the single built-in.

## Vendoring

`kit/` is the vendored SACRVM APPKIT (version in `kit/VERSION`), used by the
demo page and as the token source. Per the kit's consuming rules it is copied
**verbatim and never edited here** — to upgrade, delete the folder and unzip
the next release.

## Licence

MIT — see `LICENSE`. Vendored: SACRVM APPKIT (MIT), marked (MIT),
DOMPurify (Apache-2.0/MPL-2.0).
