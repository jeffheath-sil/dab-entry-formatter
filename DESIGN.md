# DAB Entry Formatter — design

## Problem

A FieldWorks XHTML export carries print formatting: compact, condensed, one paragraph per entry
with a hanging indent. On a phone that's hard to read. DAB can override styles for the
single-entry screen, but doing so today means hand-writing CSS against markup you can't see,
through a dialog that silently discards any row whose selector it doesn't recognise (see
`CLAUDE.md`). Getting one dictionary to look right by hand took several days and multiple
build-and-check rounds just to find out what had been ignored.

Almost none of that effort is dictionary-specific. The same small rule set, parameterised, serves
most FLEx XHTML exports.

## Solution

`dab-entry-formatter.py` writes a single self-contained file, `dab-entry-formatter.html`. Open it
in Chrome or Edge:

1. **Choose DAB project folder** — reads the `<name>.appDef` and the FLEx export
   (`<name>_data/lexicon/*.xhtml`) directly from disk via the browser's File System Access API.
   (Firefox/Safari lack that API; the tool still loads there via a plain folder picker, but can
   only offer a "Copy .appDef styles" clipboard button instead of one-click Save.)
2. **Tune** — a fixed set of layout parameters (indent step, line height, per-field "own line" /
   indent / wrap-indent / space-before, sense-number outdent) drives a live preview of real
   entries from the project, rendered with the same CSS the app itself will apply. Every
   generated selector is checked against the project's Imported Styles list live, and flagged if
   DAB would silently discard it.
3. **Save** — writes the generated `<styles type="single-entry">` block straight into the live
   appDef (byte-spliced; everything else untouched) and turns on
   `modify-single-entry-styles` if needed. The first save for a project backs up its pre-tuning
   appDef once (see `CLAUDE.md`).

No CSS, no selectors, no build-and-check loop.

## Why the browser, not a GUI toolkit

Tkinter/Qt can't lay out CSS, so a preview drawn by either would be a second layout implementation
that disagrees with the app. The app's WebView is Chromium; the user's own browser is close
enough to trust, and it costs nothing to install.

## Scope

- FLEx XHTML exports only — a LIFT-based project is detected and rejected with a message, not a
  crash.
- Single-entry view only, not the dictionary list/browse screen.
- Python 3, standard library only (field linguists on Windows, no pip); a PyInstaller `.exe` is
  built alongside for anyone without Python.

## Current limitation: fixed field list, not general role assignment

The set of tunable fields (`FIELD_DEFS` in `dab-entry-formatter.py`) is one project's rule set
generalised by hand, not derived from *this* project's export. Pointed at a different project, it
styles whichever of those classes actually exist and has nothing to say about the rest — it
doesn't yet infer roles from a class's position in the document tree (see `CLAUDE.md`'s "Roles
cannot be inferred from class-name stems" trap).

## Next

- **Second fixture.** Run against a project with a different FLEx configuration before trusting
  that the field list generalises at all. Started: a Maba project export surfaced and fixed an
  entry-parsing bug (see `CLAUDE.md`'s attribute-order trap) — the field list itself hasn't been
  checked against it yet.
- **General role assignment.** Replace the fixed field list with inference from ancestor-chain
  position, with a panel for the user to correct misreads.
- **Verify stage.** Headless-render the tuned entries and assert measured line positions against
  the geometry the parameters imply — same idea as the manual method in `CLAUDE.md`, automated
  and surfaced to the user before they save.
- **Presets.** Serialize the tuning parameters to a small JSON file so a layout can be shared
  between projects or users.
- **Upstream fix.** If SIL makes Modifications apply regardless of the Imported Styles list, the
  selector-validation logic becomes unnecessary and the tool gets simpler.
