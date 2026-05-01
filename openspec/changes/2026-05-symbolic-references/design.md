# Symbolic references over hardcoded URLs — Design

**Status:** Proposed
**Decided by:** Daniel Bejarano + Juan C. Guerrero (per Slack `1777650182.967539` taxonomy)

## Problem

Course content today links between lessons using one of three patterns:

1. **Relative `.md` paths** — e.g. `[Context Engineering](../../../agentic-coding/module-04-context-engineering/module-overview.md)`. Works on the filesystem; breaks when the platform renders Markdown because `.md` files are not addressable URLs in the runtime.
2. **No cross-references** — most existing courses simply don't link between lessons. Audit (sibling `audit-checklist.md`) confirms this is the dominant case in dojo-academy.
3. **Hypothetical hardcoded URLs** — none today (audit found 0). But as content grows, authors will reach for `[Lesson 3](/app/courses/<slug>?class=<slug>)`. That couples content to whatever route schema DojoOS happens to use that week — a recipe for the kind of breakage seen in [DOJ-3714](https://linear.app/dojo-coding/issue/DOJ-3714).

We need a convention before authors learn the wrong habit.

## Decision

Course content uses **symbolic references** in cross-link positions. The hosting platform implements a **resolver** that converts symbolic refs to live URLs at render time.

### Symbolic ref syntax

```
[<display text>][<type>:<slug-path>]
```

Or inline:

```
[<type>:<slug-path>]
```

| Type prefix | Slug-path shape | Example |
|---|---|---|
| `course:` | `<courseSlug>` | `[course:dojo-mindset]` |
| `module:` | `<courseSlug>/<moduleSlug>` | `[module:dojo-mindset/module-01-the-last-skill]` |
| `lesson:` | `<courseSlug>/<moduleSlug>/<classSlug>` | `[lesson:dojo-mindset/module-01-the-last-skill/text-01-the-last-skill]` |
| `lesson:` (workbook) | `<courseSlug>/docs/<chapterSlug>/<lessonSlug>` | `[lesson:vibe-coding-blueprint/docs/ch-01-mindset/lesson-01-vibe-coder]` |
| `video-lesson:` | `<courseSlug>/<moduleSlug>/<classSlug>` | `[video-lesson:dojo-mindset/module-01/video-01-intro]` |
| `assessment:` | `<courseSlug>/<moduleSlug>/<classSlug>` | `[assessment:dojo-mindset/module-01/quiz-01]` |
| `simulation:` | `<courseSlug>/<moduleSlug>/<classSlug>` | `[simulation:dojo-mindset/module-01/challenge-01]` |
| `final-assessment:` | `<courseSlug>/<moduleSlug>/<classSlug>` | `[final-assessment:dojo-mindset/module-final/exam-01]` |

The slug-path uses the canonical `au_id` format from DOJ-3705 (xAPI compliance). The two systems share the same identifier — `au_id` is the authoritative key, and symbolic refs are just one way to spell it.

### Resolver contract

Each platform that renders course content ships a function with this signature:

```typescript
function resolveSymbolicRef(ref: SymbolicRef, ctx: ResolveContext): string
```

Where:

```typescript
type SymbolicRef = {
  type: 'course' | 'module' | 'lesson' | 'video-lesson' | 'assessment' | 'simulation' | 'final-assessment'
  slugPath: string  // e.g. "dojo-mindset/module-01-the-last-skill/text-01-the-last-skill"
}

type ResolveContext = {
  // Optional path context — used by Pathways to emit /app/pathways/<pathSlug>/...
  pathSlug?: string
  // Locale (cmi5 translations share au_id; resolver may inject locale param)
  locale?: 'en' | 'es'
}
```

For DojoOS Pathways the resolver is roughly:

```typescript
function resolveSymbolicRef(ref: SymbolicRef, ctx: ResolveContext): string {
  const [courseSlug, ...rest] = ref.slugPath.split('/')
  switch (ref.type) {
    case 'course':
      return ctx.pathSlug
        ? `/app/pathways/${ctx.pathSlug}/courses/${courseSlug}`
        : `/app/courses/${courseSlug}`
    case 'lesson':
    case 'assessment':
    case 'simulation':
    case 'video-lesson':
    case 'final-assessment': {
      const classSlug = rest[rest.length - 1]
      const base = ctx.pathSlug
        ? `/app/pathways/${ctx.pathSlug}/courses/${courseSlug}`
        : `/app/courses/${courseSlug}`
      return `${base}?class=${classSlug}`
    }
    case 'module':
      // Modules don't have their own URL — fall back to the course home + module anchor
      return ctx.pathSlug
        ? `/app/pathways/${ctx.pathSlug}/courses/${courseSlug}#${rest[0]}`
        : `/app/courses/${courseSlug}#${rest[0]}`
  }
}
```

A Moodle resolver would emit `@@PLUGINFILE@@/...`-style refs. A Ralph LRS exporter would emit IRIs. The resolver is the only platform-coupled code; everything upstream stays portable.

### Where the convention is enforced

- **IDT commands** (`new-course`, `write-text-class`, `write-lesson`, etc.) emit symbolic refs by default. The base prompt forbids hardcoded URLs.
- **DojoOS Pathways** ships `src/lib/symbolicRefs.ts` (or similar) implementing the resolver, and the lesson renderer invokes it on every parsed Markdown link.
- **Linter** (future enhancement) — a script that walks course markdown and flags any link target that looks like a route schema (`/app/`, `https://*.dojocoding.io/app/`).

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| **Hardcoded URLs** — author writes `/app/courses/<slug>?class=<slug>` directly | Couples every line of content to the current DojoOS route schema. Any schema change (e.g. fixing [DOJ-3714](https://linear.app/dojo-coding/issue/DOJ-3714)) becomes an N-file content rewrite, where N = number of cross-refs. The cost grows with content volume; today N is small, in a year it's >1,000. |
| **URL templates with placeholders** — author writes `{baseUrl}/courses/{courseSlug}?class={classSlug}` | Decouples the host but still bakes the route shape into content. If DojoOS moves from `?class=` to `/classes/<slug>`, every template needs a content edit. Symbolic refs don't expose the route shape at all. |
| **Frontmatter-only refs** — store cross-refs as a `related: ["lesson:..."]` array in frontmatter, not in body | Loses inline display text — readers can't see "see [Context Engineering][module:agentic-coding/...]" in the prose. Inline refs are how teachers naturally weave references into explanations. |
| **A new file extension** (`.dmd`?) | Premature standardization. Markdown with link-text-as-symbolic-ref reuses every existing tool (preview, grep, diff) and doesn't fork the ecosystem. |

## Trade-offs of this design

**Costs:**

- **Resolver per platform.** DojoOS today; Moodle / Ralph / Cornerstone / etc. each ship their own. Most are 30-50 lines of code.
- **Refs not clickable in raw Markdown preview.** GitHub renders `[course:dojo-mindset]` as plain text, not a link. Acceptable: the rendered platform is where the reader actually clicks.
- **One more concept for course authors to learn.** Mitigated by IDT enforcement — authors using `new-course` don't write refs by hand.

**Benefits:**

- **Schema changes don't touch content.** [DOJ-3714](https://linear.app/dojo-coding/issue/DOJ-3714) (path-aware routing) is a one-file resolver fix in DojoOS, not a content sweep.
- **Content portability.** Same symbolic refs work on Moodle, Ralph LRS, Cornerstone — each platform's resolver does the translation.
- **`au_id` and symbolic refs share the same identifier.** Single source of truth — no risk of drift between the cmi5 invariant and the cross-reference convention.

## How this depends on / interacts with other changes

| Sibling | Relationship |
|---|---|
| [DOJ-3705](https://linear.app/dojo-coding/issue/DOJ-3705) (xAPI / au_id template compliance) | This change reuses au_id as the slug-path. DOJ-3705 must ship first (or at least concurrently) so the IDs exist. |
| [DOJ-3714](https://linear.app/dojo-coding/issue/DOJ-3714) (path-aware routing in dojo-os) | Independent fix on the platform side. Once symbolic refs are in place, DOJ-3714's fix only changes the resolver — content stays put. |
| [DOJ-3707](https://linear.app/dojo-coding/issue/DOJ-3707) (skill-overlay discovery in IDT) | Editorial overlays may want to add platform-specific anchors (`#section`) post-resolution. Overlay protocol must allow that. |
| [DOJ-3712](https://linear.app/dojo-coding/issue/DOJ-3712) (CLAUDE.md docs in dojo-academy + IDT) | Documents this convention for human authors. |
