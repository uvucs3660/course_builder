# AGENTS.md

This file provides guidance to Codex (Codex.ai/code) when working with code in this repository.

## What this is

The course publishing pipeline: it compiles `content/<course>/<year>/` (YAML + Markdown + assets) into the class's published artifacts. Two publish targets share that content tree:

- **Canvas** — the Dart CLI (`bin/build_canvas_zip.dart`) emits a Common Cartridge `.imscc` zip importable into UVU's Canvas.
- **Web** — `bin/build_lectures.dart` generates lecture deck content (slides, TTS scripts) into `content/<course>/<year>/lectures/slides/_lectures` (gitignored), and `lecture_site/` (a TypeScript/Netlify app with its own `README.md`) builds and serves it at https://cs3540-lectures.netlify.app, with instructor narration recording and per-student listening progress on top. `tool/publish_lectures.sh` runs generate → build → deploy in one command.

It is one subproject inside the `uvu/` instructor workspace — see the workspace `AGENTS.md` two levels up for the broader context (semester lifecycle, FERPA rules, what each sibling tool does). Remote: `uvucs3660/course_builder` (renamed 2026-09-02 from `summer_2026`; GitHub redirects the old URL).

Two live content sets: `content/cs3660/2026/` and `content/cs3540/2026/`. Spec at `docs/specs/2026-05-06-cs3660-2026-redesign-design.md`; implementation plan at `docs/plans/2026-05-06-cs3660-2026-canvas-zip.md`.

## Commands

```bash
dart pub get                                                           # once after clone
dart run bin/build_canvas_zip.dart content/cs3660/2026 dist/cs-3660-001-summer-2026.imscc
dart run bin/build_canvas_zip.dart content/cs3540/2026 dist/cs-3540-001-fall-2026.imscc
dart test                                                              # full suite
dart test test/loaders/course_loader_test.dart                         # one file
dart test -n 'Canvas import invariants'                                # one group / name regex
dart analyze                                                           # uses analysis_options.yaml (lints/recommended)
dart run bin/build_lectures.dart                                       # regenerate lecture decks (defaults: cs3540 2026)
tool/publish_lectures.sh                                               # decks + lecture_site build + Netlify deploy
```

`lecture_site/` has its own toolchain (`npm test`, `npx tsc --noEmit`, `npm run build`, `npm run deploy` — see `lecture_site/README.md`). Its build reads `CONTENT_DIR`, defaulting to this repo's `content/cs3540/2026/lectures/slides/_lectures`.

`dist/` is gitignored — the built `.imscc` is a release artifact, not source.

## Pipeline (the big picture)

```
content/<year>/course.yaml + */*.md + */*.yaml
        │
        ▼
lib/src/loaders/                    YAML + frontmatter + markdown → in-memory models
        │
        ▼
lib/src/models/Course               (assignments, wikiPages, modules, rubrics, quizzes, webResources)
        │
        ▼
lib/src/emitters/                   each emits one Canvas XML/HTML file
        │
        ▼
lib/src/packager.dart               assembles the zip with the exact directory layout
                                    Canvas's importer expects
        │
        ▼
dist/<name>.imscc                   uploaded to Canvas via Settings → Import Course Content
```

Producer-first ordering matters: when you add a new content kind, plumb it loader → model → emitter → packager → manifest in that order. Skipping the manifest step is the most common mistake — Canvas only imports what `imsmanifest.xml` declares.

## Canvas Common Cartridge: non-obvious invariants

The format is reverse-engineered from `../cs-3660-001-_-2025-summer-full-term-export/` (the Summer 2025 export). When in doubt, open that directory and compare. The regression suite at `test/canvas_import_invariants_test.dart` documents the specific bugs that have already been hit.

- **Identifiers come from `imsId('kind:slug')` (`lib/src/ims_id.dart`).** MD5 of the namespaced slug, prefixed with `g`, lowercase hex — matches Canvas's own `g…` directory names. Callers MUST namespace (`assignment:foo` vs `page:foo`) or two different things collide on the same id.
- **Wiki pages need three `<meta>` tags in the rendered HTML head** — `identifier`, `editing_roles`, `workflow_state` (and `front_page` on the front page). `renderMarkdownToCanvasHtml` adds them when `canvasIdentifier` is passed. Without them, module references silently resolve to nothing.
- **Assignments do NOT need those meta tags** in the body HTML. Their metadata lives in `assignment_settings.xml` next to the body. The body's `<file href>` in the manifest must point at `g<id>/<slug>.html`, not `g<id>/g<id>.html`.
- **Internal cross-links use `$WIKI_REFERENCE$/pages/<imsId>` and `$CANVAS_OBJECT_REFERENCE$/assignments/<imsId>`** — never the slug. Canvas resolves these tokens against manifest identifiers during import.
- **Resource type in the manifest is `imscc_xmlv1p1`**, not `imscc_xsd`. The latter is a different IMS variant Canvas's importer doesn't handle.
- **Rubrics import as a free-floating library, not attached to assignments.** The loader inlines a rendered rubric table into each assignment body (`_appendRubricTable` in `course_loader.dart`) so criteria are visible before the instructor manually attaches the rubric in the Canvas UI. The association element in `assignment_settings.xml` is `<rubric_identifierref>`, not `rubric_id`.
- **Authors write relative markdown links; the loader rewrites them.** Content cross-links as `[text](other-page.md)` so links work on GitHub and in IDE preview. After every page is loaded, `rewriteRelativeMarkdownLinks` resolves each bare `<name>.md` href against all page slugs (falling back to `cheatsheet-<name>`) and replaces it with the `$WIKI_REFERENCE$` token. Unresolvable targets are left as-is — a visibly relative link beats a plausible identifier that goes nowhere.
- **Quizzes ship duplicated:** `g<id>/assessment_qti.xml` AND `non_cc_assessments/<id>.xml.qti`. The 2025 export shows this duplication; Canvas reads either, but the manifest declares both resources.
- **SVGs with `<style>` blocks must travel via `web_resources/`, not inline.** Canvas sanitizes inline SVG and strips the styles. Files served from `web_resources/` come back as raw bytes. The `cheatsheets_dir` glob loader collects every non-markdown sidecar and rewrites `<img src="…">` to `$IMS-CC-FILEBASE$/<cheatsheets_dir>/<rel>`.

## Content directory conventions

`content/<course>/<year>/` is the input — one directory per course per year. `bin/build_canvas_zip.dart <content-dir> <output-file>` takes it as an argument; there is no hardcoded year in `lib/`. Tests **do** hardcode paths, so grep for `content/cs` before moving anything.

`course.yaml` is the index; almost everything else is glob-loaded by directory:

- **`pages/`** — explicit list in `course.yaml#pages`. The page listed as `front_page: true` becomes the course landing page.
- **`cheatsheets_dir`** (currently `cheatsheets/`) — every `*.md` becomes a wiki page with slug `cheatsheet-<basename>`; title is the first H1. Auto-emits a `Cheat Sheet Library` module. Non-markdown files (SVGs, images) under the dir ship as web resources.
- **`lectures_dir`** (currently `lectures/`) — every `*.md` must have YAML frontmatter declaring `week`, `youtube_id`, `companion_sheets`, `reflection_assignment`, `vernacular_tags`. The loader prepends an auto-generated banner (cheat sheet links, reflection link, vernacular tags, YouTube embed or "not yet recorded" placeholder) and emits a `Lecture Spine` module sorted by week.
- **`quizzes_dir`** (currently `quizzes/`) — every `*.yaml` is one quiz. Each question must have **exactly one** `correct: true` choice; `loadQuiz` throws if not. Each quiz auto-creates a paired remediation assignment (slug `<quiz>-remediation`, due 48 h after, 0 points, `online_url` submission, uses `quiz-remediation` rubric) — this is the "C-3 workflow" referenced in the spec.
- **`images_dir`** (cs3540: `images/`) — every non-markdown file under it ships as a web resource; page bodies reference `$IMS-CC-FILEBASE$/<images_dir>/<name>`.
- **`schedule:`** (optional block in `course.yaml`) — the loader *generates* the course schedule page (`lib/src/schedule.dart`) from the declared `weeks` (topic, meeting days, act headings, holiday notes) merged with the real assignment due dates, and links each week to its lecture page. Never hand-write a schedule page — it drifts from `course.yaml` invisibly. Due dates in `course.yaml` are UTC; the schedule renders them in Mountain Time (`toMountain`, DST-aware).
- **Assignment body dirs** — markdown bodies referenced explicitly by `course.yaml#assignments`: cs3660 uses `onboarding/`, `reflections/`, `sprints/`, `cc-artifacts/`; cs3540 uses `onboarding/`, `assignments/`, `devlog/`.
- **`rubrics/*.yaml`** — listed explicitly in `course.yaml#rubrics`.

### Rubric slugs are globally unique across courses

The 2h2.us grader's `RubricService.load(slug)` reads `${slug}.yaml` from **one flat directory** and throws if the file's `slug:` field differs from the filename stem. fivex's `mod_node/scripts/refresh-rubrics.mjs` copies every course's rubrics into that flat directory and **exits non-zero on a filename collision** — without that guard, two courses shipping `pass-fail.yaml` would silently clobber each other.

Prefix per course. CS 3540 uses `cs3540-`; CS 3660's predate the convention and stay bare.

## Adding a new year

`README.md` covers the steps. The trap: dates in `course.yaml` (`start_at`, `end_at`, every `due_at`) must move with the new term, and the zybook code in lecture/page bodies may also need refreshing. After copying, run the build and inspect the resulting zip's `course_settings/module_meta.xml` to confirm dates flowed through.

## When something fails to import in Canvas

1. Re-run `dart test` — `canvas_import_invariants_test.dart` catches the known classes of breakage.
2. Diff the offending file against the same path in `../cs-3660-001-_-2025-summer-full-term-export/`. Canvas accepts what's in that export.
3. Check `imsmanifest.xml` declares the resource. Canvas silently ignores files not in the manifest.
4. For broken cross-links, grep the rendered HTML for slug strings — if a `$WIKI_REFERENCE$/pages/<X>` uses a slug instead of an `imsId(…)`, Canvas can't resolve it.
