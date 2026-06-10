---
title: "Building bookreader-tui: a terminal EPUB reader, in phases, in a week"
published: false
description: "A retrospective on building bookreader-tui — a Textual TUI for reading EPUBs and keeping a personal library — from a weekend hack to v1.0.0 on PyPI. The architecture, the phases, the bug that ate a morning, and the rename that ate a day."
tags: python, textual, tui, ebook
series: "Building TUIs in anger"
canonical_url: https://github.com/prajwalmahajan101/BookReader
---

> A retrospective on [`bookreader-tui`](https://github.com/prajwalmahajan101/BookReader) — a terminal EPUB reader and personal library in Python + [Textual](https://textual.textualize.io/), built in named phases and shipped to PyPI as v1.0.0.

## TL;DR — and the bug you should remember

**Inline graphics are widgets, not render strings.** If you're using a render-time string protocol (kitty / iTerm2 / sixel) inside a TUI framework that owns its widget tree, the only way to make images survive layout, scroll, and re-render is to mount them as actual widgets. A `Static` whose `render()` returns escape sequences will work in scroll-mode and silently fail in any view that re-renders out of band — pagination, modal layering, hot-swapped layouts.

That bug ate a morning. The fix was a rewrite of the two-page widget from "one `Static` per spread" to "`Horizontal[Vertical, Vertical]` of mounted `Static` + `Image` per page." That uncovered the second bug — `DuplicateIds` when the next spread mounted before the previous one finished tearing down — which the async-`remove_children()` ordering fixes, with a regression test pinning the order.

The numbers: ~5 phases, ~50 tests, mypy `--strict`, ruff format + check, CI on Python 3.11 and 3.12, layered `core` → `epub` → `library` → `ui`. v0.1.0 → v1.0.0 in roughly a week of evenings. PyPI distribution name `bookreader-tui` because `bookreader` was taken (about that, later).

The order:

```
core (config, paths, errors) → epub (parse, render)
   → library (repo + service over SQLite)
   → ui (screens, widgets)
   → polish (themes, two-page, bookmarks, inline images)
   → release (PyPI, docs site, README rewrite)
```

The point of the post is the order, the boundaries between layers, and the small number of decisions each layer earned.

## What bookreader-tui is, and isn't

A terminal app for reading EPUBs and keeping a small personal library. Textual draws the UI; SQLite stores the library; everything runs in your shell. No cloud, no daemon, no Qt, no Electron — your books live in `~/.local/share/bookreader/` and nothing leaves the machine.

What it isn't: a Calibre replacement. There's no DRM handling, no format conversion, no metadata fetching from third-party catalogues, no e-reader sync. If you need a fleet manager for a Kindle, this isn't it. If you want to open an EPUB in your terminal, resume exactly where you left off, and have your collection a `:` away — that's the whole pitch.

## The four-layer rule, in one paragraph

```
src/bookreader/
├── core/      # config, paths, logging, exceptions. No I/O.
├── epub/      # parse + render EPUBs. Pure. No UI, no DB.
├── library/   # persistence + business logic.
│              #   repository/  → SQLite, raw rows
│              #   service/     → orchestration, the UI's only door
└── ui/        # Textual screens + widgets. Talks to services.
```

The rule the UI obeys: **never import a repository directly.** UI talks to a service, the service talks to the repository, the repository talks to SQLite. Three reasons that boundary earns its keep:

1. **Tests are honest.** `epub/` is pure-Python — unit tests need a fixture EPUB, nothing else. `library/repository/` is unit-testable against a temp-file SQLite. `library/service/` gets integration tests — real SQLite, no mocks, no skipping the layer that matters. The UI keeps a handful of smoke tests, but the interesting assertions live below it.
2. **Errors have a home.** Every layer raises a subclass of `BookReaderError`. The repository wraps `sqlite3.OperationalError` in `RepositoryError` at its boundary so the UI's central handler can route them. The UI catches `BookReaderError`, *not* `Exception` — programming bugs (`KeyError`, `AttributeError`) bubble, and you find them. This was a Phase-2 cleanup, after a `bare except: Exception` painted a real `AttributeError` as "Add failed" and ate twenty minutes of debugging.
3. **You can rewrite a layer without burning the project.** I rewrote `PagedView` twice. The reader stayed unchanged because rendering an EPUB chapter is a pure function in `epub/`, and the widget composing the spread is the only thing that had to know about pagination.

mypy `--strict` is the gate that holds this together. The day you let a single `Any` in at a layer boundary is the day the discipline stops paying.

## Phases, and why phases work

Every change landed on a `feature/phaseN_*` branch, in conventional commits, atomic per logical change. `main` was releasable throughout — every merge was something you could `pipx install` that evening.

| Phase | Theme | The one thing it earned |
|---|---|---|
| **1** | Reader core | Read an EPUB, render text, remember position. JSON state file. |
| **1.5** | Two-page mode | A `PagedView` that balances a spread across two columns. |
| **2** | SQLite library | Repository + service, library screen, collections. |
| **2.1 / 2.2** | Theme + width | Three themes, `BOOKREADER_READING_WIDTH`, command palette. |
| **3** | Polish wave 2 | Bookmarks with notes, multi-session continuity by EPUB `dc:identifier`, inline images. |
| **4** | Library curation | `C` collections overview, `W` wishlist, reader-side library actions. |
| **5.0** | Pre-flight | Image auto-detect, `I` runtime toggle, theme picker scoping, paged-mode image fix. |
| **0.4 → 1.0** | Release | PyPI publish, Apache-2.0, mkdocs-material docs site, README rewrite. |

The reason phases work isn't ceremony — it's that **each phase has exactly one user-visible thing it owes you**, and a clear acceptance bar. Phase 2 isn't done when "the library code is there." Phase 2 is done when "I can press a key, see my books, open one, and the position from before resumes." That bar forces the integration tests up front instead of after.

The cost of phases: you have to be honest about what's *not* in scope this round. Phase 1 doesn't have a library — it has a single JSON file. Phase 3.4 doesn't have collections — those are Phase 4. If you can't write that sentence, you don't have a phase, you have a wishlist.

## The EPUB layer: rendering without an HTML engine

An EPUB is a zip of XHTML, CSS, images, and a `content.opf` manifest. The library options are `ebooklib` (parse the zip and manifest) plus `BeautifulSoup` + `lxml` (parse each chapter's XHTML). What `epub/` *doesn't* do: layout. Layout is the UI's problem.

The contract `epub/` exposes to the UI is small:

- A `Book` — title, author, identifier (`dc:identifier`, the load-bearing key for resume continuity), TOC tree.
- A `Chapter` — id, title, an ordered list of *render blocks*: paragraphs, headings, lists, blockquotes, code, images (by manifest path).

Render blocks are the boundary. They're plain dataclasses; they have no Textual types in them; you can serialize them; you can test them. The UI knows how to turn one into a `Static` (text blocks) or an `Image` widget (image blocks). The EPUB layer never imports Textual.

The decision the layer earned: **don't try to honour publisher CSS.** EPUBs are full of styling that assumes a real browser-shaped layout engine — page floats, fixed-position figures, font-size cascades. A TUI doesn't have any of that, and faking it produces uncanny-valley reading. Pick a clean column width (`BOOKREADER_READING_WIDTH`, default 110, override 60–200), one set of in-app themes, and let the typography breathe. The reader looks the same across every EPUB you open. That's a feature.

## The library layer: SQLite is the right answer

Schema lives in `library/repository/migrations/`. Forward-only, numbered, idempotent. Sqlite via stdlib `sqlite3`, no ORM. The repository methods are typed, single-purpose, raw SQL. The service composes them.

Three things the layer learned the hard way:

**Identifier, not path, is the primary key for resume.** A book at `~/books/dune.epub` and the same book at `/mnt/usb/dune.epub` are the same book. The state table is keyed on `dc:identifier`; the path is a secondary lookup. Move a file, change a directory layout, plug in a different USB — your position survives.

**The wishlist needs its own table.** A wishlist entry is a book you don't have yet — title, optional author, intent to read later. Trying to model it as a `Book` row with nullable everything turned the schema into a swamp. Two tables, one foreign-key column for the eventual attachment when the EPUB arrives.

**Sessions are append-only.** Per-book reading time is the sum of session durations; the session table is never updated, only inserted into. If you need to recompute time-read, you replay. This is the same instinct as a WAL: don't mutate what you can derive.

## The UI: Textual, screens, and three keys that mattered

Textual gives you screens (full-frame views), widgets (composable trees), CSS-like styling, and a key-binding system. The app has roughly the screens you'd expect — library, reader, modal forms for ratings / bookmarks / wishlist — and a command palette via `Ctrl+P`.

Three key bindings are worth calling out because they shaped the rest of the UX:

- **`c` / `C`.** Lowercase toggles completion on the current book. Uppercase opens the *Collections* overview. Symmetric, discoverable, doesn't collide with `q` / `Q`.
- **`I`.** Runtime toggle for inline images. Critical because graphics-protocol detection isn't always right, and "the images are broken" is sometimes "the terminal doesn't speak kitty graphics today."
- **`/`.** Filter in the library list. Not in 1.0 — but reserved in the binding table so it doesn't accidentally collide with the `:` command bar that's coming.

The thing Textual makes easy that surprised me: themed CSS. Three themes (`dark`, `light`, `sepia`) are three short stylesheets, each variable-driven, each switchable at runtime. Once the colour tokens are in variables, "ship a new theme" is "write a new file."

The thing Textual makes painful: layout for inline graphics. Which is the next section.

## The image bug, in full

This is the one from the TL;DR. The setup, in three steps:

**Step 1 — scroll mode worked.** `ChapterView` renders one block per child widget: `Static` for text, `Image` (from [`textual-image`](https://pypi.org/project/textual-image/)) for images. Scroll naturally, images render inline, layout reflows.

**Step 2 — paged mode rendered text and lost images.** `PagedView` had a different render strategy: one `Static` per spread, whose `render()` method returned the concatenated chapter text with image escape sequences spliced in. This worked for kitty graphics escape codes in *isolation* — you could `cat` them into a terminal and see images. Inside a `Static.render()`, they… kind of worked, briefly, and then disappeared on the next scroll.

**Step 3 — the realisation.** Graphics protocols like kitty's aren't text. They place pixels at a coordinate the terminal tracks separately from the text grid, and they expect the surrounding app to manage the lifecycle of those placements — clear them on redraw, re-emit on scroll, scope them to a region. `textual-image` does exactly that, *as long as it owns a widget node in the tree*. Hand-splicing escape codes into a `Static.render()` cuts the widget out of the lifecycle, so the placement leaks until Textual paints over it.

The fix:

```python
class PagedView(Vertical):
    async def mount_spread(self, left: Page, right: Page) -> None:
        await self.remove_children()        # async, must await
        spread = Horizontal(
            Vertical(*self._widgets(left),  id="page-left"),
            Vertical(*self._widgets(right), id="page-right"),
        )
        await self.mount(spread)

    def _widgets(self, page: Page) -> list[Widget]:
        out: list[Widget] = []
        for block in page.blocks:
            if isinstance(block, TextBlock):
                out.append(Static(block.markup))
            elif isinstance(block, ImageBlock):
                out.append(Image(block.path, width="auto"))
        return out
```

Two things to notice. First, **`await self.remove_children()` before `mount`** — synchronous removal lets the new spread's widget IDs collide with the old spread's still-tearing-down widgets, and Textual raises `DuplicateIds`. There's a regression test that mounts spread A, mounts spread B, and asserts no exception — the kind of test that looks trivial and pins a real bug.

Second, **`width: "auto"`** on the inline `Image`. In a fixed-height column, `height: 1fr` collapses the image to zero on the first frame; `height: auto` with `max-width: 100%` is the combination that respects the natural EPUB image dimensions while clamping to the column width. Counterintuitive — documented now in the troubleshooting page.

The general rule that fell out of this: **if a widget's content has a lifecycle the framework manages — graphics placements, focus, hit-testing — give it a widget node. Don't try to be clever inside `render()`.**

## The `BOOKREADER_IMAGES_ENABLED` morning

The image bug above wasn't the *first* bug in inline images. The first bug was me — I'd forgotten to set `BOOKREADER_IMAGES_ENABLED=1`. I spent a morning convinced the protocol was broken end-to-end, then watched it work the moment the env var landed.

The post-mortem produced a useful design decision: **auto-detect, env-var to override.** The 5.0c phase inspects `TERM`, `LC_TERMINAL`, `WEZTERM_EXECUTABLE`, and the `KITTY_*` family to enable images automatically in graphics-capable terminals. Explicit `BOOKREADER_IMAGES_ENABLED` still wins when set. And `I` toggles at runtime, so if the auto-detect is wrong for your setup, you can correct it without restarting.

The rule, generalised: **a feature flag that's only set by users who already know the feature exists is a feature off by default.** Auto-detect for the common case, env var for the override, runtime toggle for the apology.

## Silent except, and the day it ate an `AttributeError`

Phase 2 shipped with a modal form for adding a book. Form validation lived in the screen's submit handler, wrapped in:

```python
try:
    self.service.add_book(...)
except Exception:
    self.app.notify("Add failed", severity="error")
```

That `Exception` is doing exactly what its name says: catching everything. Including the `AttributeError` from a typo I'd introduced. Including the `KeyError` from a missing column lookup. Including, on one memorable evening, a `TypeError` from passing a `Path` where a `str` was expected.

The cleanup (`feature/cleanup_error_safety`):

```python
try:
    self.service.add_book(...)
except BookReaderError as exc:
    self.app.notify(str(exc), severity="error")
```

And a `RepositoryError` introduced at the SQLite boundary so the service can `raise RepositoryError(...) from e` without leaking `sqlite3.OperationalError` into the UI catch list.

Two tests pin this down: one asserts that a `BookReaderError` raised in a service becomes a notification, and one asserts that a deliberately-injected `KeyError` *propagates* out of the modal so the test fails loudly. The second test is the load-bearing one. **A swallowed bug is the same as a missing test.**

## The PyPI rename: one decision I'd unmake

The Python package is `bookreader`. The console script is `bookreader`. The repo is `BookReader`. Every internal name agrees — except the PyPI *distribution*, which is `bookreader-tui`.

The reason: `bookreader` was already a registered project on PyPI. Renaming the *package* would have rippled through every import, every doc, every screenshot caption, every key-binding documentation that says "in `bookreader.ui`, …" — so the package kept its name, the script kept its name, and the distribution took the `-tui` suffix.

The cost is permanent: every install instruction has to say "install `bookreader-tui`, the command is `bookreader`." That's an extra sentence in every tutorial, every reply to a "how do I install this?" question, every README forever.

If you ever build something you intend to publish on PyPI: **check the distribution name first, before you write the first line of code.** It's a `curl https://pypi.org/pypi/<name>/json` away. I didn't. I should have.

## Release engineering: PyPI trusted publishing + mkdocs

The release pipeline is the smallest thing that works:

- A tag of the form `v*` on `main` triggers `.github/workflows/release.yml`.
- The workflow builds the sdist + wheel, publishes via PyPI **trusted publishing** (OIDC, no long-lived tokens in CI), and cuts a GitHub Release with the relevant `CHANGELOG.md` section as the body.
- A separate `gh-pages.yml` workflow builds the mkdocs-material site and deploys to GitHub Pages on every push to `main` that touches `docs/site/`, `mkdocs.yml`, or the README.

Trusted publishing is the part worth highlighting. The old model is "store a long-lived PyPI API token in your CI secrets and pray it doesn't leak." The new model is "configure PyPI to trust this specific repo's GitHub Actions environment, and CI swaps a short-lived OIDC token at publish time." Zero secrets to rotate, zero secrets to leak. Every Python project that publishes from CI should be using it.

## The 1.0.0 polish pass

The code surface in 1.0.0 is identical to 0.4.0. What 1.0.0 added is everything *around* the code:

- A README rewrite for first-time users: `pipx` / `uv tool` / `pip-in-venv` install paths with a PEP-668 note, a terminal × graphics-protocol compatibility matrix, a quick start, troubleshooting, and a comparison-vs-alternatives block.
- The mkdocs-material site, with the docs split per concern (install, images, keys, configuration, troubleshooting, development, ADRs, changelog).
- Real SVG screenshots captured via a Textual SVG pilot, used in the README, on PyPI, and on the docs site.
- `SECURITY.md` (vulnerability reporting via GitHub Private Vulnerability Reporting) and `CONTRIBUTING.md` (commit / branching / architectural conventions, distilled).
- GitHub repo metadata — description, homepage, topics. Surprisingly large impact on discoverability for a five-minute change.

The hardest part of 1.0.0 was *not* slipping new features in. There's always one more polish item that feels like it belongs. The trick is to write the changelog before the work — if it doesn't fit the "polish pass" section, it goes in the next phase.

## ADRs as crystallization, not ceremony

There are four ADRs in `docs/adr/`:

1. **0001 — Stack choice.** Why Textual, why `ebooklib`, why SQLite, why `pydantic-settings`.
2. **0002 — Library screen.** The repo + service split, the UI's contract.
3. **0003 — Textual theme system.** Variable-driven themes, why three is enough, how the picker is scoped.
4. **0004 — Phase 3 data model.** Bookmarks, sessions, identifier-keyed resume.

ADRs aren't documentation, they're decision *receipts*. The point isn't to predict; it's to record the constraints that were active when the decision was made, so the next person — including future me — can tell whether the constraint still holds. An ADR that just says "we chose X" is worse than nothing. An ADR that says "we chose X because Y was load-bearing at the time" is gold when Y changes and you have to reconsider.

The four above all earned their place by being asked-about later.

## What I'd do differently

- **Name-check on PyPI on day zero.** Already covered. Still smarts.
- **Auto-detect-first for every terminal capability flag.** Not just images — colour depth, mouse support, unicode width. Env vars should be overrides, not the primary path.
- **Service-layer integration tests from Phase 1.** I had repository unit tests early, but the service-layer integration tests didn't land until Phase 2.1. A few of the boundary bugs that ate time would have been caught earlier with even a smoke-level integration test.
- **Write the screenshot-capture script in Phase 1.** Not at the 1.0.0 polish pass. Screenshots evolve with the UI; capturing them is a chore that gets put off; having the script means a release-time screenshot refresh is a one-liner.

## What's next

Roughly, in order:

1. **0.4.x / 1.0.x docs polish.** Animated `vhs` hero for the README (already scripted at `docs/demo.tape`, blocked on CI vhs dependencies), expanded troubleshooting from the field reports already arriving.
2. **Phase 6 — search and filter.** Full-text chapter search across the library, `/` filter in the library list (already drafted as Wave 5 of the cleanup plan), and the `:` command bar.
3. **Read-only export.** A portable format so the SQLite library + reading state can move between machines manually. Cloud sync is explicitly out of scope; cross-machine portability is not the same problem.

## The one-line version

**Pick the boundaries, ship the phases, trust the framework's widget tree.** The bugs that ate the most time were all the same shape — assuming I could be clever inside a layer that owned its own lifecycle. The phases I'm proudest of are the ones where I let each layer do its job and pushed all the polish to the layer above.

If you try [`bookreader-tui`](https://pypi.org/project/bookreader-tui/) and it breaks, please [open an issue](https://github.com/prajwalmahajan101/BookReader/issues) — terminal + Python version attached. The fastest path to 1.0.1 is the bug report I haven't seen yet.
