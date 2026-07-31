# DAB Entry Formatter

A tool that gives Dictionary App Builder (DAB) users readable single-entry formatting from a
FieldWorks XHTML export, without hand-writing CSS. See `DESIGN.md` for what it does and how.

`dab-entry-formatter.py` writes out `dab-entry-formatter.html`, a self-contained browser tool
(open it directly — no server, no install) that reads a DAB project's appDef + FLEx export, lets
the user tune a small set of layout parameters against a live preview, and saves the result
straight back into the project.

This file holds the facts about DAB/FLEx that the tool's logic depends on — the reason it's built
the way it is. `prototype/HISTORY.md` (not published) has the fuller story of how they were
learned.

## Vocabulary

- **DAB** — Dictionary App Builder (SIL). Builds Android dictionary apps.
- **FLEx** — FieldWorks Language Explorer. Exports a dictionary as `.xhtml` + `.css`.
- **appDef** — DAB's project file, `<name>.appDef`, XML.
- **Imported Styles** — DAB's parsed model of the exported CSS. In the appDef: the `<styles>`
  element with **no** `type` attribute.
- **Modifications** — user overrides applied only when a single entry is displayed. In the
  appDef: `<styles type="single-entry">`. Gated by
  `<feature name="modify-single-entry-styles" value="true"/>`.

## The constraint that governs everything

**A Modification row is applied only if its selector string appears verbatim in the Imported
Styles list.** Rows with any other selector are accepted by the dialog, saved to the appDef,
redisplayed on reopen — and silently discarded when the app is built. No warning anywhere.
(See `prototype/HISTORY.md` for the evidence this was established from.)

Corollaries:

- **Modifications override imported declarations regardless of CSS specificity.** A one-class
  selector beats a four-class imported rule. DAB appears to merge them into the existing rule
  rather than append a later one. To emulate DAB in a local browser test, add `!important` to
  the modification declarations — otherwise specificity gives the wrong answer.
- **Prefer bare single-class names.** The Imported list holds both full CSS selectors and a bare
  name per style. Bare names are honoured, and because FLEx reuses classes across contexts, one
  row often covers main entries *and* subentries — `.examplescontent`, `.sharedgrammaticalinfo`,
  `.sensecontent`, `.sensenumber` all do.
- **Always validate generated selectors against the Imported list and refuse to write on a
  miss.** `dab-entry-formatter.py`'s generated tool does this both live (in the Tune preview) and
  again before every Copy/Save.

## The style model

The whole layout reduces to three numbers plus per-field "own line" choices:

| | |
|---|---|
| hang = 0.6em | a block's wrapped lines sit 0.6em in from its own first line |
| step = 0.6em | each level of subordination steps in 0.6em |
| number box = 1.2em | the sense number hangs out to the left of its sense |

Resulting left edges (first line / wrapped):

```
0     / 0.6    headword, part of speech, variant back-refs
0              sense number
  1.2 / 1.8    definition
    1.8 / 2.4  example + translation, scientific name, semantic domains
0.6            subentry headword, subentry part of speech, subentry sense number
  1.8 / 2.4    subentry definition
    2.4 / 3.0  subentry example + translation
```

Geometry composes from the containing block, so a bare-class rule produces correctly *different*
positions in each context: `.sensecontent { margin-left: 1.8em }` puts the definition at 1.2em in
a main entry and 1.8em in a subentry, because the subentry is itself indented 0.6em.

These defaults, and the field/selector grouping they apply to, are `FIELD_DEFS` and
`DEFAULT_GLOBALS`/`DEFAULT_SENSE_NUMBER` in `dab-entry-formatter.py`.

## Traps

- **`text-indent` is inherited.** FLEx puts `text-indent: -12pt; margin-left: 21pt` on `.entry`
  for the print hanging indent, and it leaks into every block DAB creates. Reset it explicitly on
  every block you introduce.
- **Roles cannot be inferred from class-name stems.** Bare names collapse to fewer stems than
  there are roles, which looks like a clean generalisation and is not one: a project can have a
  subentry headword, a minor-complex-form headword, and a cross-reference headword (which must
  stay inline) all sharing the same stem with different numeric suffixes. Assign roles from
  position in the document tree, not from the name, and show the user the result.
- **The `-N` suffixes are configuration-dependent.** Never hardcode them; derive from the export.
- **Attribute order/presence on entry `<div>`s is configuration-dependent too.** Some FLEx export
  configurations insert extra attributes — e.g. `nodeId="..."` — between `class` and `id` on
  every top-level entry div. Match `class=` and `id=` independently wherever they fall in the tag,
  never assume a fixed attribute sequence (found via a second fixture, a Maba project export).
- **Example separators live in CSS `content:`, not in the text.** FLEx puts the separator in
  `.examplescontents:before` and `.examplescontent + .examplescontent:before`. Print
  configurations use `' | '`; a screen configuration may use `'\0A'` (a literal newline, rendered
  because `.entry` carries `white-space: pre-wrap`). Blanking those with `content: ''` works for
  either, so the same rules serve both a print and a screen export.
- **A block inside an inline span still breaks the line.** No rule is needed on the
  `.examplescontents` wrapper; making `.examplescontent` a block is enough.
- **Scope: FLEx XHTML exports only.** LIFT-based DAB projects have entirely different markup and
  none of these class names.

## Verifying a rule set

Don't trust "the CSS looks right" — measure it:

1. Build an HTML page: the project's exported CSS + the generated `<styles type="single-entry">`
   rules (with `!important` added, per the specificity corollary above) + one sample entry per
   structural type.
2. Render it and **measure** the left edge of each line with `Range.getClientRects()`, grouping
   rects by rounded `top` to get one x per line.
3. Assert against the intended geometry table above.

Chrome is available for headless rendering:

```bash
"/c/Program Files/Google/Chrome/Application/chrome.exe" --headless=new --disable-gpu --no-sandbox \
  --hide-scrollbars --force-device-scale-factor=2 --window-size=1040,900 \
  "--screenshot=C:\path\out.png" "file:///C:/path/in.html"
```

`file://` URLs work for rendering; use `python -m http.server` instead if the page loads sibling
files. Chrome may be denied write access to some sandboxed/temp directories — give it an output
path under the project directory if a screenshot write fails silently.

## Editing an appDef

`dab-entry-formatter.py`'s generated tool writes directly into the **live** appDef — a deliberate
departure from "never touch the live file," made at the project owner's explicit request. The one
safety net: the first Save for a project backs up its pre-tuning appDef once, as
`<name> (backup before dab-entry-formatter).appDef`, and never overwrites that backup again.

- Replace only the `<styles type="single-entry">` block; byte-splice so everything else stays
  untouched. Assert the prefix and suffix are byte-identical afterward.
- Preserve the file's existing encoding and line endings (detect, don't assume — e.g. Baraïn's
  file is UTF-8 without BOM, CRLF throughout).
- Escaping inside the `name` attribute: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`,
  `'` → `&apos;`, `"` → `&quot;`.
- Row format: `<style name="SELECTOR" category="text">` with
  `<style-decl property="P" value="V"/>` children. An empty content value is
  `value="&apos;&apos;"`.
- Leave the `<stylesheets>` checksums and the imported `<styles>` block alone — DAB recomputes
  them when it reopens the project.

## Build conventions

- Python 3, **standard library only** — end users are field linguists on Windows without pip.
- The browser is the GUI. Never re-implement CSS layout in a widget toolkit; the preview must be
  rendered by the same engine class as the app's WebView.
- Ship a PyInstaller single-file `.exe` alongside the `.py`.
