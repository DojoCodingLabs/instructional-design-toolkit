---
name: workbook-generate
description: >
  Interactive course workbook design system — turns a course's text classes into one
  standalone, accessible, interactive HTML explainer (a navigable document chunked by
  module, Articulate-Rise lesson model; stepped mode for screen-recording).
  Defines the curated component kit, the content-shape -> component mapping rules,
  the accessibility contract, and the consumer-spec (instruction-bundle-spec.yaml)
  consumption + graceful-degrade rules. Consumer plugins own the brand tokens and the
  component vocabulary via their `workbooks` spec section; this skill stays voice-neutral.
triggers:
  - workbook
  - interactive workbook
  - course workbook
  - generate workbook
  - rise
  - interactive explainer
---

# Interactive Workbook Design Knowledge Base

This skill encodes the design system for **interactive course workbooks** — a
single, standalone, self-contained HTML artifact that renders a whole course's
**text classes** (`text-*.md`) as an interactive explainer the learner scrolls
or steps through.

It is the student-facing sibling of `course-visualize` (which produces an
instructor-facing analytics view of the same course). Where `slides-generate`
turns a **video brief** into a deck for live teaching, `workbook-generate`
turns the **reading classes** into a self-study / screen-share artifact.

The philosophy is borrowed from interactive-HTML explainers: a concept should
recover its natural shape. A process becomes a navigable flow, a comparison
becomes a side-by-side, code becomes annotated-and-copyable, and "do you
follow?" becomes a reveal + a check question. Interaction is **felt**, not
described — but it is delivered through a **curated, tested component kit**, not
freeform per-concept JavaScript.

## What this skill is NOT

- It is **not** a slide system (`slide-design` / `slides-generate` own that).
- It is **not** a persistence layer. V1 holds all learner answers **in memory
  only** — refresh resets. No `localStorage`, no `postMessage`, no parent-frame
  communication, no backend. (Those are deliberately deferred.)
- It does **not** ship brand tokens. Palette / typography / voice come from the
  consumer's `instruction-bundle-spec.yaml`. This knowledge base is
  voice-neutral and must stay that way (the "would this help an instructor in
  Argentina shipping a Python course on Moodle?" test).

## The artifact (per-course by default)

One `workbook-{course-slug}.html` covering **all modules** of a course, with:

- a **module-chunked navigable flow** (default, `scroll_behavior: doc`): one
  module ("lesson") shows at a time — the Articulate-Rise model, not one endless
  scroll. A **stepped** flow (one screen at a time, for screen-recording) is the
  alternative. **No scroll-snap** — plain smooth scroll within a lesson.
- a **table of contents** that switches the active module — a **sticky sidebar
  on desktop**, a **collapsible "Contents" menu on mobile** (a slim sticky bar +
  toggle, never a wall of links pushing content down);
- a **progress indicator** (module-based) and, at each module's end, an **arc
  indicator** ("Module N of M") + a **forward-hook** Next control;
- **interactive blocks** chosen to match each piece of content's shape (see the
  mapping rules below).

### Light-touch storytelling (structure here, voice in the overlay)

The base ships the *slots* that give a course narrative momentum, voice-neutral:
the **arc indicator** (where you are), and a **forward-hook** on the Next control
— a primary action line (`Next: {title} →`) plus an **optional teaser line**
that is a **voice slot**. Leave the teaser empty in the base draft (it hides via
`:empty`, so a generic consumer just sees `Next: {title} →`); the consumer's
voice overlay fills it with momentum copy (the L3 "momentum endings" concern).
Richer beats (transition/closure cards, connective lesson intros) are a
deliberate future addition, not in the base today.

An optional `--scope module` narrows generation to a single module
(`workbook-{module-slug}.html` under that module).

## The curated component kit

Each component is a documented, theme-variable-driven, accessible-by-construction
block. **This catalog is owned by IDT base** — the canonical, voice-neutral
vocabulary of what an interactive workbook can contain. The base template ships
the HTML/CSS/JS for all of them; the command composes a workbook by selecting
and filling them. A consumer MAY narrow, extend, or set options on this catalog
via an **optional** `workbooks.components` override in their spec; absent that
override, the full catalog is available with sensible defaults. The library
itself never lives in the consumer spec — only brand theming and thin
preferences do.

> **What interactivity is FOR here.** The artifact's job is to **display rich
> information**. Per the source thesis, interactivity earns its place in exactly
> two ways: (a) **felt understanding** — drag/click and *see* the effect
> (knobs, diagrams, simulations); or (b) **closing the loop** — hand the learner
> something to take back to Claude or a mentor (**export / copy-as-prompt**).
> It does NOT capture answers. Because V1 persists nothing, a widget that
> records a learner's input and does nothing with it (an ungraded quiz to
> nowhere, a reflection box that evaporates on refresh) is an anti-pattern.
> Self-check belongs in **predict-and-reveal** (immediate, no capture); taking
> work forward belongs in **export**.

### Baseline (always available)

| Component | Purpose |
|---|---|
| **step / section** | A unit of the flow. Each `H2` of a reading is typically one step. |
| **progress bar** | Course-level and per-module completion %. Updates as the learner advances. |
| **completion event** | Fired (not persisted) when the learner reaches the end. A hook point for V2. |

### Progressive disclosure

| Component | Purpose |
|---|---|
| **accordion** | Collapsible detail blocks. Scaffolds complexity — TL;DR open, depth on demand. |
| **tabs** | Parallel variants on one surface (e.g. "Python / JS / Go", "before / after"). |
| **reveal** | Hide content until the learner asks for it. |
| **predict-and-reveal** | A "think about this first" prompt + a hidden **model answer / expert take**. The self-check pattern — pedagogy via retrieval + immediate feedback, with **no capture and no grading**. Replaces ungraded quizzes/input boxes. |

### Loop-closer (the "stay in the loop" move)

| Component | Purpose |
|---|---|
| **export / copy-as-prompt** | A button that copies a ready-made prompt (or the learner's notes) to the clipboard so they can paste it into Claude or send it to a mentor. The only way interaction leaves a no-persistence artifact — turns passive display into two-way. Clipboard has a `file://` fallback. |

### Inline SVG (restores the spatial dimension)

| Component | Purpose |
|---|---|
| **flow diagram** | A process / pipeline as clickable, ordered steps. |
| **comparison diagram** | Two or more options laid out spatially for contrast. |
| **chart** | A small bar / line chart. Hand-rendered SVG with **precomputed coordinates** (no runtime scale fn) and the **single salient datum recolored** to the accent (`.is-peak`) while the rest stay neutral — the highlight is itself a teaching move. No library. |
| **annotated figure** | An SVG figure with callout labels. |

### Annotated code

| Component | Purpose |
|---|---|
| **code block** | Syntax-styled code with a copy button (with a `file://` clipboard fallback) and optional margin annotations / line callouts. High value for a coding curriculum. |
| **code diff** | A before -> after variant of the code block: removed lines (`.diff-del`, struck through) and added lines (`.diff-add`) banded inline. Teaches *the change*, not just the result — the core move of teaching code. |

### Parametric knob (the reliable "felt" interaction)

| Component | Purpose |
|---|---|
| **knob** | A slider whose value writes a CSS custom property (`--knob`) that a preview element consumes and a readout mirrors. Lets a learner *drag and watch the effect* — the felt interactivity of the reference site's simulations, delivered through a fixed, tested component rather than bespoke per-concept JS. In-memory only. |

### Course navigation

| Component | Purpose |
|---|---|
| **module chapter** | A titled chapter grouping a module's reading steps. |
| **table of contents** | Jump-to navigation across modules; reflects progress. |

### Stretch tier (only when a concept truly needs it — add deliberately, test it)

| Component | Purpose |
|---|---|
| **live simulation** | A measured simulation — state array -> recompute -> diff-vs-previous -> re-render, with a readout that prints the lesson's key metric live (e.g. "25% of keys moved"). The most powerful pattern in the reference, but the most expensive to generate/QA — reach for the **knob** first. |
| **state visualizer** | A small interactive state machine / structure (e.g. add/remove nodes) for a concept best understood by manipulation. |

These are the most fragile to generate and the most expensive to QA. Prefer a
baseline or disclosure component unless manipulation is the point of the lesson.

## Semantic color language

Color carries consistent meaning across every component (and is always paired
with text or an icon — never color alone, per the accessibility contract):

- **accent** (`--wb-accent`) = the *current / focal / interactive* thing — the
  active step, a selected tab, the salient datum on a chart, an interactive
  control, a key term.
- **good** (`--wb-good`) = *success / done / favourable* — a completed step, the
  favourable side of a comparison, added lines in a diff.
- **bad** (`--wb-bad`) = *removed / unfavourable* — the weak side of a
  comparison, removed lines in a diff.

This mirrors the reference site's clay / olive / rust roles. Consumers retheme
the hexes via the spec's `design.palette`; the *roles* stay fixed so the visual
grammar is stable across a course (stable grammar = lower cognitive load).

## Content-shape -> component mapping (the pedagogical core)

Read the reading's prose and promote it into the component whose shape fits.
There is no rigid formula; the content determines the form. Heuristics:

- **Course** -> module **chapters** (ordered by module directory), with a
  **table of contents** and course-level **progress**.
- **Each `text-*.md`** -> a run of steps inside its module chapter.
- **Each `H2`** -> a **step / section**.
- **A table** -> a **comparison diagram**, **tabs**, or a **chart** (pick by
  whether the table contrasts options, shows variants, or holds numbers).
- **An ordered "step 1 / 2 / 3" list or a described process** -> a **flow
  diagram** or a stepper.
- **A fenced code block** -> an **annotated code block**; if it shows an edit
  (old vs new), use the **code diff** variant.
- **A small dataset with a standout value** -> a **chart**, recoloring the peak.
- **A tunable relationship** ("as X grows, Y...") -> a **knob** so the learner
  drags X and watches Y; reach for a **live simulation** only if measuring the
  outcome is the lesson.
- **A quotable / load-bearing sentence** -> a **statement / callout**.
- **A digression or optional depth** -> an **accordion**.
- **Parallel alternatives** (languages, approaches) -> **tabs**.
- **End of each major section** -> an optional **predict-and-reveal** self-check
  (a "think about it first" question + a revealed model answer) — not a graded
  quiz. Sibling `quiz-*.md` files become predict-and-reveal checks.
- **Something the learner would want to take further** (a parameter they tuned,
  a reflection, a topic to go deeper on) -> an **export / copy-as-prompt** so it
  leaves the page as a prompt for Claude or a mentor.
- **Course intro** -> a **hero** step; **course end** -> a **recap** + the
  **completion event**.

Density guidance: not every paragraph needs a widget. Interactivity should
earn its place — one well-chosen interactive block per concept beats five
decorative ones. Long readings get chaptered and made collapsible so the
single-file artifact stays navigable.

## Consuming the consumer spec (`instruction-bundle-spec.yaml`)

The workbook's brand + structural vocabulary comes from the consumer's
`instruction-bundle-spec.yaml` (see the command for resolution order). This
skill reads:

- **`design`** (top level) -> `palette`, `typography`, `voice`, `spacing`,
  `components`. Inlined into the generated HTML as CSS custom properties.
- **`workbooks.scroll_behavior`** -> `doc` (navigable, module-chunked; **default**)
  or `stepped` (one screen at a time, for recording). No scroll-snap.
- **`workbooks.navigation`** -> TOC layout (sidebar desktop / collapsible mobile),
  arc indicator, and the forward-hook on the Next control (teaser = voice slot).
- **`workbooks.animation`** -> `on_enter` transition + `reduced_motion` policy.
- **`workbooks.step_patterns`** -> the available step layouts (hero, content,
  quiz, branch, progress, ...).
- **`workbooks.components`** (OPTIONAL) -> a consumer **override layer** on top
  of IDT's built-in catalog (the vocabulary above, owned by IDT base). When
  present, it may enable/disable components, set options, or opt into the
  stretch tier. **When absent, the full IDT catalog is available with sensible
  defaults** — the workbook is fully rich, not plainer. The consumer spec never
  *defines* the library; it only themes and tunes it.

### Validate before generating

Fail fast (with a clear, path-pointing error) when required keys are missing:
`design.palette.accent`, `design.typography.family.body`,
`workbooks.step_patterns`. Do not emit a half-themed artifact.

### Graceful degrade (never crash)

If `instruction-bundle-spec.yaml` cannot be resolved from any source, emit the
workbook anyway with **neutral defaults** (system-ui typography, a single
monochrome accent, a generic stepped layout) and a **visible WARNING** — both
to the user (stderr / chat) and as an HTML comment at the top of the generated
file. Never silently produce a neutral artifact as if the spec said so, and
never crash.

## Standalone HTML contract

- **No CDN runtime dependencies.** Corporate / school networks block them.
  Fonts are the one allowed exception — a Google Fonts `<link>` with a
  `system-ui` fallback in the font stack.
- **No build step, no framework.** Inline `<style>` and inline vanilla
  `<script>`. The file opens in any browser by double-clicking it.
- **Design CSS inlined at generation time** from the `design` palette/type.
- **Self-contained assets.** Any image is inlined (e.g. base64) or referenced
  relative to the file; nothing fetched at runtime.

## Accessibility contract (not optional)

- Full keyboard operation: Tab order follows reading order; **Enter / arrow
  keys advance** steps; all interactive controls are focusable.
- Landmark roles: each step is a `role="region"` with an `aria-label`; the
  nav layer is a `<nav>`; progress uses `role="progressbar"` with
  `aria-valuenow`.
- `prefers-reduced-motion: reduce` disables enter animations and smooth-scroll.
- Color contrast meets WCAG AA for text on the chosen background (verify the
  consumer's palette; the neutral defaults already pass).
- State conveyed by text + icon, not color alone (e.g. diff add/del, comparison).

## Anti-patterns

- Hardcoding any brand token, copy, or "Dojo"-ism in this skill or the base
  template. Brand lives in the consumer spec.
- Hardcoding the component catalog or step patterns here instead of reading
  them from the spec.
- Freeform per-concept JavaScript. Compose from the kit; extend the kit
  deliberately (and test it) when a concept truly needs something new.
- Decorative interactivity. Every widget must teach something the prose alone
  could not convey as well.
- **Capture-to-nowhere widgets.** Do not add inputs that record a learner's
  answer and do nothing with it (ungraded quiz, reflection box). With no
  persistence they are dead ends. Use **predict-and-reveal** for self-check and
  **export** to take work forward.
- Persisting state in V1. The completion event is a fire-only hook; nothing
  else is stored.
