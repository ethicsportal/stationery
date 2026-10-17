# AGENTS.md

What a contributor to Stationery, human or not, has to know and cannot read off the code. The docs site under `docs/` has an `AGENTS.md` of its own for writing its pages.

## What the gem is

Stationery writes PDFs from Ruby classes: a document is a class, `view_template` describes it, and the engine measures, places and paginates. It is pure Ruby, with no runtime dependency and no native extension. That is the point of the gem, and no change may give it up.

| Where | What |
|---|---|
| `lib/stationery/` | The gem: elements, layout, text, fonts, images, the PDF writer, tagging, the testing helpers |
| `spec/stationery/`, `spec/integration/` | Unit specs and whole documents; `spec/rails/` runs under `gemfiles/rails.gemfile` |
| `examples/` | One runnable file per example, each with a `preview` |
| `benchmark/` | Benchmarks, the profiler, the memory report and the metrics gate |
| `docs/` | The docs site, a Rails application of its own with its own bundle |

## The canvas

A layout node, the SVG renderer and a `canvas { |c| … }` block call only what `Stationery::Canvas::Interface` names, in top-left coordinates and points: never `canvas.page`, `canvas.num`, a path's `to_s`, an operator or the resources, which are the PDF canvas's own. The paginator, the page templates and `Structure` ask the render's canvases (`PDF::Canvases`) for the canvas of a page, and `Document#paint_on` is what every output runs. `spec/stationery/document_paint_on_spec.rb` paints every example on a canvas that is not the PDF canvas, so a node that reaches for PDF fails there.

## How a change is made

1. **A branch off fresh `main`**, one pull request per issue or per part of one. Nothing is pushed to `main`.
2. **A spec first**, seen failing, then the change.
3. **Before the pull request is opened**, all three pass:

   ```sh
   bundle exec rspec                                  # judged by its exit status, not by its summary line
   bundle exec rubocop lib spec examples Rakefile     # docs/ has its own bundle and is linted by CI
   bundle exec rake metrics
   ```

4. **A changelog bullet** under `## Unreleased` in `CHANGELOG.md`, and the README and docs pages that state what the change touches are updated with it. Several docs pages embed sections of the README, so the README is what counts for them.
5. **Merged when CI is green**, with `main` merged into the branch first when `main` has moved (merged, never rebased, never force-pushed). A pull request stacked on another is pointed at `main` before its parent merges.

## The metrics gate

`bundle exec rake metrics` renders fixed documents and compares their allocations, pages and bytes with `benchmark/baseline.json`. CI holds allocations and not time, because wall time on a shared machine is noise.

- **A document that does not use a feature is byte for byte what it was**, and allocates what it did. A new feature costs nothing where it is not used.
- The baseline is recorded again (`bundle exec rake metrics:update`, with the Ruby the baseline names) only for a change made on purpose, and the commit says why.
- A version bump lengthens the version string in the file, so the release preparation records the baseline again.

## What a standard asks for is checked, not reasoned about

| Claim | Validator | Task |
|---|---|---|
| PDF/A, PDF/UA | veraPDF: a local `verapdf` (`brew install verapdf`), else Docker | `bundle exec rake verify:conformance` |
| Factur-X | Mustang, through Docker | `bundle exec rake verify:factur_x` |
| Signatures | `openssl`, and `pdfsig` when installed | `bundle exec rake verify:signature` |
| File structure | `qpdf --check` | `bundle exec rake verify:readers` |
| Opens and works in the major viewers | qpdf, Poppler, MuPDF, PDFium, pdf.js, PDFKit | `bundle exec rake verify:readers` |

A new example that is tagged joins the `conformance` renders in the `Rakefile`, which `verify:readers` checks too. An engine message is added to `Stationery::Verify::Allowlist` only when its cause is found in the engine's source and is the check's environment, not the file; a file that lacks what a viewer needs is a bug to fix. A rule is taken from the validator's profile, and a pull request says which rule it is.

## What is drawn is looked at

A change to layout is rendered to a picture (`pdftoppm -png -r 72 file.pdf out`) and looked at before it is merged, and the pull request says what was seen. `to_pdf(debug: true)` draws the rectangles of the layout.

## The changelog

- One bullet per change, under `## Unreleased`, written for someone who uses the gem: what it does, how it is written, what it leaves alone, and the classes it lives in.
- A change in behaviour is named as one. A release gathers them in one bullet, **Behaviour changes**, at the end of its section.
- When both sides of a merge changed a bullet, `main`'s is kept. After a merge the bullets are read again: a union of both sides can keep the wrong copy.

## Labels

Every pull request carries exactly one `type` label and at least one `area`
label from `.github/labels.yml`, never a `status` label. `/flow:plan` labels the
issue, `/flow:lfg` copies the issue's `type` and `area` labels onto the PR (never
`plan` or another status label). Without an issue, the type comes from the
change's conventional-commit prefix and the areas from
`bin/labels infer $(git diff --name-only origin/main...HEAD)`. Labels change in
the manifest and reach GitHub with `bin/labels sync`, never through the UI.
Rules: `.github/LABELS.md`. `bin/labels` + `.github/LABELS.md` are the shared
labels kit (canonical copy in docs-kit): never edit them in place.

## Releases

Releases are cut by the maintainer with `bin/release` (`bin/release --dry-run` shows what would ship), never with `rake release` by hand and never by an agent without being asked. A release is prepared by a pull request that turns `## Unreleased` into the section of the version and records the metrics baseline. `bin/release`, `rakelib/release.rake` and the shared jobs of `.github/workflows/release.yml` are the zoolutions release kit, synced verbatim from docs-kit (`RELEASE_KIT.md` there): never edit them here; only the `test` and `publish-verify-image` jobs are this repo's.

Acrobat cannot be driven in CI, so before a release pull request merges the maintainer opens a fixed set from `bundle exec rake verify:readers:render` (`tmp/readers/`) in Acrobat Reader on macOS **and** Windows: `invoice`, `form`, `report`, `e_invoice`, `accessible_report`, `conforming_signed_form` and `encrypted_invoice` (password `reader`). On each: it opens without a warning bar; the form fills and prints; links and bookmarks jump; the attachment opens; the signature panel reads the signature; the tag tree shows in Acrobat's Accessibility panel (Pro) or its reading order. The result is a checklist in the release pull request's body. No claim of Adobe compatibility is made beyond what this step saw.

## Code and specs

- Pure Ruby, stdlib only. What the gem cannot do is a warning (`Warnings::...`), not approximated and not raised where a document can still render. No `rescue` that swallows, no `&.` over a nil you have not explained. No shared mutable state at class level (`rubocop-thread_safety`).
- Unit specs in `spec/stationery/`, whole documents in `spec/integration/`, Rails in `spec/rails/` (`BUNDLE_GEMFILE=gemfiles/rails.gemfile bundle exec rake spec:rails`). Assert through `lib/stationery/testing/` on what was laid out and drawn, not on a PDF's bytes; byte comparisons pin the clock. Every branch of a public DSL option is covered (80% minimum).
- A new example reads nothing from `spec/` except the font fixtures in `spec/fixtures/fonts/` (`examples/invoice.rb` reads them), has a `preview`, and is painted on a non-PDF canvas by `spec/stationery/document_paint_on_spec.rb`. Look at what is drawn: `bundle exec stationery render <file>.rb --png`.

## Performance

Measure before and after on the same machine, or make no claim; `/ruby-gem:perf` does it with `main` in a worktree. `bundle exec rake bench` is time against Prawn (not in CI: wall time on a shared machine is noise; within the ± is "within noise", and say whether a number is one operation or a whole render). `bundle exec rake metrics` is what CI holds. Also `PROFILE=1 bundle exec ruby -Ilib benchmark/profile.rb` (StackProf) and `bundle exec rake memory` (long documents). Hot paths, where a change comes with numbers: text wrapping and measurement (`text/`, `fonts/`), layout and pagination (`layout/`), tables, images, the writer and its compression (`pdf/`). Never optimise a path the profile does not show hot, and never trade correctness or conformance for a win the numbers do not show.

## Workflow

- **Verify:** `bundle exec rspec` (exit status) · `bundle exec rubocop lib spec examples Rakefile` · `bundle exec rake metrics` · tagging, PDF/A or PDF/UA touched: `bundle exec rake verify:conformance` · `docs/`: `cd docs && bundle exec rubocop && bundle exec rspec`
- **Specs:** fast loop `bundle exec rspec <file>`; full suite `bundle exec rspec`; Rails `BUNDLE_GEMFILE=gemfiles/rails.gemfile bundle exec rake spec:rails`; two worktrees at once: yes
- **CI:** required `Lint`, `Ruby 3.4`, `Ruby 4.0`, `Docs site`, both `conformance / ...`; `Metrics gate`, `PDF/A and PDF/UA (veraPDF)`, `Readers (macOS, PDFKit)`, `Rails` are not required but must be green; full suite on the PR; failure report none
- **Review bot:** cubic (threads must be resolved)
- **Merge:** squash — `gh pr merge <PR> --squash --delete-branch`
- **PR contract:** `## Summary` (`Closes #N`), `## Test plan`; labels per `## Labels`; layout change: say what the rendered picture showed; conformance claim: name the veraPDF rule
- **Conflicts:** `CHANGELOG.md` — both sides' bullets under `## Unreleased`, a bullet both changed takes main's, re-read after; `lib/stationery/version.rb` — main's; `docs/Gemfile.lock` — main's with the `stationery (X.Y.Z)` pin set to `version.rb` under `PATH` and `CHECKSUMS`, never `bundle lock`; `benchmark/baseline.json` — main's, then `bundle exec rake metrics`
- **Worktree setup:** `bundle install` (`cd docs && bundle install` for docs work); undo none
- **Release:** `bin/release <v>`, the maintainer's, never unasked; changelog `CHANGELOG.md`, heading `## X.Y.Z (YYYY-MM-DD)` under `## Unreleased`; prep `bundle exec rake metrics:update`; lockfiles `docs/Gemfile.lock`; docs `deploy-docs.yml` on release; human: the Acrobat Reader checklist in `## Releases`
- **Main health:** the CI status of main's head
- **Upstream:** libraries we own `zoolutions/*`; scrub none
- **Bench:** `bundle exec rake bench`; `bundle exec rake metrics` (CI holds `benchmark/baseline.json`); hot paths in `## Performance`

## Out of scope by design

- A CSS layout engine: `html` reads a subset and reports the rest as `Warnings::UnsupportedCss`.
- Shaping and bidi inside the gem: the `shaper` hook hands text to an application's shaper.
- Native code, and any runtime dependency.
- Lossy WebP.

## Models and agents

Sessions run on `opus` with `fable` as the advisor (`.claude/settings.json`); consult it before choosing an approach, a public API or option name, a dependency, anything irreversible, and when a failure repeats. Every spawned agent names its `model:`; one that does not runs on `sonnet` (`CLAUDE_CODE_SUBAGENT_MODEL`). The workflow commands come from the `flow` and `ruby-gem` plugins (`/flow:lfg`, `/flow:plan`, `/flow:finish-prs`, `/ruby-gem:perf`, ...); this repository keeps `/architect` and `/security` in `.claude/commands/`. The docs site keeps its own: `docs/AGENTS.md` and the `write-docs-page` skill in `docs/.claude/skills/`.

## What has gone wrong before

- Rubocop run on `docs/` from the repository root fails on a missing plugin. Lint `lib spec examples Rakefile` and leave `docs/` to CI.
- The docs image is built from the repository root with `docs/Dockerfile.dockerignore`, which leaves `spec/` out: an example may not read a file from `spec/`.
- Two renders compared byte for byte need the clock pinned, or they differ by their dates when a second turns between them.
- GitHub closes a pull request whose base branch is deleted. A pull request stacked on another is pointed at `main` before its parent is merged.
- An overflow fixture of many paragraphs splits across pages instead of overflowing. Use `box(break_inside: :avoid)` for one that must overflow.
