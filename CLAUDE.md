# CLAUDE.md

Guidance for Claude Code in this repository.

## What this is

One web component: `<sac-md-editor>`, an Obsidian-style live-preview markdown
editor. Add-on module for the SACRVM APPKIT — deliberately **not** in the core
kit (decided upstream; the kit's `MIGRATION.md` lists it as the planned
optional module). The whole product is `js/sac-md-editor.js` (the editor)
and `js/sac-md-secret.js` (the `:::secret` block registration), plus the demo
`index.html`; `kit/` is the vendored appkit (version in `kit/VERSION`),
**verbatim, never edited here** — to upgrade, delete the folder and unzip the
next release.

**Zero build, forever.** No bundler, no TypeScript, no node_modules, no test
framework build chain. Plain files + F5 is a hard ecosystem rule, not a
temporary state. Dev loop: `npx serve .`, open the root page, edit, reload.

## The one invariant — everything else is negotiable

**The canonical document is the concatenated `textContent` of the line divs.**
Every transformation preserves `textContent` exactly: marker characters
(`**`, `#`, backticks…) stay in the DOM and get *wrapped* in inline elements
for styling — never synthesized, never deleted. This is what makes the
save/load round-trip trivial and the active/inactive line flip cheap.

Any change that would make rendered output differ from `textContent` is wrong,
even if it looks right on screen. When reviewing a patch, check this first.

## Architecture tour (~3,000 lines, one file — on purpose, see roadmap)

- **Line model**: one `div` per source line inside the shadow root. The caret
  line ("active") shows flat raw text; all other lines render.
- **Inline rendering**: `marked.Lexer.lexInline` tokenizes; we walk the token
  tree ourselves and emit wrapped spans (`renderInline`). We do NOT use
  marked's HTML renderer.
- **Inline HTML policy**: escape-don't-parse, with DOMPurify as the second
  line of defence. User-typed `<script>` is text, always.
- **Cross-line state**: `classifyLine(src, state, setext)` is the ONE place
  that decides what a line is; `advanceState` is the one place that moves
  state downward. State = `inFence` (the opening ``` / ~~~ run, since only the
  same character at least as long closes), `inSecret`, `prev` (what the line
  above was: setext underlines, indented code and link definitions depend on
  it) and `listCtx` (an indented line inside a list is continuation, not
  code). Fence-awareness is load-bearing: `:::secret` inside a code fence is
  text, not a boundary.
- **The sweep**: every edit ends in `_syncBlockState()`, because one keystroke
  can change lines far away - type ``` and everything below becomes code;
  type `===` and the paragraph ABOVE becomes a heading; add `[id]: url` and
  references anywhere resolve. It walks top-down (state, types, link
  definitions), then bottom-up (setext levels), then re-renders exactly the
  lines whose key (`_lineKey`: state + setext + definitions for lines with
  `]`) changed. Plain typing re-renders nothing. Loading a value uses the same
  sweep on detached lines, then inserts them in one go.
- **Block registry**: the editor knows no block names. `SacMdEditor.registerBlock(def)`
  (static, page-wide; reach it via `customElements.get("sac-md-editor")`)
  adds a bounded block: `open`/`close` RegExps or a general `match(src)`
  returning "open" / "close" / "line" (a one-line marker block). Blocks live
  in `BLOCKS`, are classified in `classifyLine` outside fences only, do not
  nest, and render inline markdown in their body. Lines get
  `.block-open|close|body|line` + `.block-<name>`; `masked` blurs bodies,
  `toggle` puts the eye (contenteditable=false, one-line SVG so no stray
  text nodes break the round-trip) on the opening line and flips
  `.block-revealed` per block, session-only. `label` / `color` travel as
  inline custom properties on the line; a definition's `css` is injected
  into every editor's shadow root (the only way a host can style inside
  it). Registering re-derives every live editor (`refreshBlocks`), and a
  detached editor catches up on connect (`blockGen`). A throwing host
  `match` counts as no match.
- **`:::secret`** is the registration in `js/sac-md-secret.js` - the demo loads
  it, hosts that want it load it after the editor.
- **Delimiters** (in the secret registration): read `:{2,3}`, write `:::`. The two-colon form is what
  content written before 2026-08 carries; the three-colon form reads as a
  Pandoc-style fenced div. Both stay accepted — a bulk data migration was
  rejected upstream because a naive rewrite cannot see code fences and would
  silently corrupt content documenting the syntax.

## Security stance — load-bearing, do not soften

The blur is a courtesy against shoulder-surfing, **not** a security boundary.
Plaintext stays in the DOM and in `value`. Never advertise masking as
security, never add a "the editor protects your secrets" line to any doc, and
never build features that assume the blur hides data from code. A host that
needs real secrecy enforces it server-side — the upstream app strips these
blocks on every machine-facing path (search index, embeddings, agent wire)
and that enforcement lives there, not here.

## Origin and the Fishbowl relationship

Extracted 2026-08-28 from `SACRVM/the-fishbowl`
(`src/Fishbowl.Data/Resources/js/components/fb-md-editor.js`) after four
months of daily production use there. The port renamed the tag
(`fb-md-editor` → `sac-md-editor`) and internal classes; behaviour is
unchanged. **Fishbowl vendors this repo** since 2026-09-25
(`src/Fishbowl.Data/Resources/js/vendor/sac-md-editor/`, pinned commit in the
`VERSION` file there, first pin 27b9e2e) - it is a consumer, so coordinate
breaking changes with it. So is `SACRVM/markdown-pad` (`vendor/`).

What deliberately did NOT move here: Fishbowl's secret *semantics* — the
server-side `SecretStripper`, the client-side encryption round-trip
(`:::secret#N:::end` markers + `content_secret`), and the invariant tests.
Those are host concerns. This component only does the visual layer.

## API surface (public contract — breaking it needs a version bump)

Attributes `placeholder`, `readonly` · property `value` · events `input`,
`change` (native, composed) · method `focus()` · static
`registerBlock(def)` → unregister fn, `unregisterBlock(name)`, `blocks`
(definition fields: name, open/close or match, masked, toggle, label,
className, color, css - see the header).
Line classes `.block-*` are internal; a host styles its blocks through the
definition's `css`. Keyboard: Ctrl/Cmd+B/I/K,
Enter list continuation, empty-item list exit, Backspace line merge, Tab
soft-tab, and in a table Tab / Shift+Tab cell navigation and Enter new row
(leaving a table re-aligns its pipes - a source edit that fires `input`).
i18n keys `md-editor.*` (hosts may add languages under them, so
renaming a key is breaking too); strings follow `sac.lang` live via
`_relabel()`, which only touches attributes, toolbar text and a CSS custom
property, never line `textContent`.

Known divergence from kit v2 conventions: the kit's value components fire
`sac:change`/`sac:input` with `detail: { value }`; this editor still fires
native `input`/`change` from its Fishbowl days. Aligning is a breaking change
— do it deliberately, with the consumer, not as a drive-by.

## Roadmap (ordered)

1. ~~**Block-type registry**~~ — done 2026-09-27 (see Architecture).
   `:::secret` is a registration (`js/sac-md-secret.js`); Fishbowl has to load
   it (or register its own, e.g. the encrypted single-line
   `:::secret#N:::end` form via `match` returning "line") when it next
   re-vendors past 27b9e2e - that is the breaking part.
2. **Kit event alignment** — `sac:change`/`sac:input` per the kit convention,
   coordinated with Fishbowl's switch to vendoring this repo.
3. **Split the file** — upstream's own review called 2,092 lines in one file a
   wart. Only after 1 and 2; splitting first would make both harder to review.

## Conventions

- English and ASCII in source and comments.
- Header comments ARE the API documentation (kit convention) — keep the
  top-of-file block accurate; the README quotes it.
- Manual verification: run the demo, exercise caret movement across rendered
  lines, list continuation, a code fence containing `:::secret` (must stay
  text), the reveal toggle, tables (Tab / Enter / leaving re-aligns), and a
  copy/paste round-trip. There is no automated
  suite yet; if one lands, it must not drag in a build step.

## Firepit inbox

At the start of a session, read any pending messages in `.firepit/inbox/*.md` — cross-project notes Firepit routes here. Act on each, then mark it done with the `firepit_inbox_complete` MCP tool, passing the message's filename as the `id`.

## Firepit knowledge

Before researching something that may already be known, query the knowledge base with the `firepit_knowledge_search` MCP tool (scope `both` covers this project plus the global base). Save durable findings with `firepit_knowledge_add` — written in English, per the indexing convention. **This is a public repo, so `.firepit/knowledge` is a pointer file into the private central store** — the docs live there, never in this tree.

## Firepit pinned knowledge

@.firepit/knowledge-pinned.md

The import above auto-loads the knowledge docs marked `pin: true` in their frontmatter — always-on rules that apply every session without a search. Firepit regenerates the file from the pinned docs (it is gitignored here, per the public-repo convention); don't edit it directly. Pin/unpin via the pinned flag on `firepit_knowledge_add` / `firepit_knowledge_update`, and keep the pinned set small — everything else stays reachable through `firepit_knowledge_search`.

## Firepit artifacts

When you produce a file the user will want to open — a report, screenshot, diagram, generated image, log excerpt, build output, or an executable you built for them to run — pin it with the `firepit_artifact_add` MCP tool so it appears in the project's paperclip pane. Do this as you produce it, not at the end of the session; a path buried in scrollback is a path the user has to hunt for. Pinning only links the file — it stays where it is, and `firepit_artifact_remove` never deletes it. Check `firepit_artifact_list` first so you update an existing entry instead of piling up near-duplicates, and unpin what has gone stale.

## Firepit conventions

<!-- claude-firepit-fragments -->

@../.firepit/projects/claude.md
@../.firepit/projects/claude-github-public.md

The two imports above are shared files in the Firepit central repo — edit them there and every project follows. They carry policy; the tools themselves are described by Firepit's MCP server at the handshake, so nothing is duplicated between the two.
