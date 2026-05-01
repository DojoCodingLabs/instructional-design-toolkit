# Symbolic references over hardcoded URLs — Proposal

**Linear:** [DOJ-3713](https://linear.app/dojo-coding/issue/DOJ-3713)
**Status:** Proposed (pending governance answer to Slack thread `1777650182.967539`)
**Owners:** Daniel Bejarano (CEO), Juan C. Guerrero (Security PO)

## What we will build

A convention for cross-references in course content that decouples authoring from rendering. Course markdown will use **symbolic refs** of the form `[course:<slug>]`, `[lesson:<courseSlug>/<moduleSlug>/<classSlug>]`, etc. The hosting platform (DojoOS today, optionally Moodle / Ralph LRS / SCORM Cloud tomorrow) ships a **resolver** that translates symbolic refs into the platform's live route schema at render time.

## For whom

| Audience | What they get |
|---|---|
| Course authors (humans + IDT-driven agents) | A stable, route-agnostic way to link from one lesson to another. They never have to know whether DojoOS uses `/app/courses/<slug>` or `/app/pathways/<pathSlug>/courses/<slug>` — the platform handles it. |
| DojoOS Pathways team | Freedom to evolve the route schema (e.g. fix DOJ-3714 path-aware routing) without rewriting any course content. |
| External LMS hosts (Freedom Academy on Moodle, Cornerstone, Ralph LRS, etc.) | Cross-references that survive an export from dojo-academy because the `[course:<slug>]` syntax doesn't bake in DojoOS routing. |
| `instructional-design-toolkit` itself | A way to fulfill its multi-tenant promise — the toolkit can emit symbolic refs without coupling any of its outputs to a specific platform. |

## Why now

Two pressures surfaced this on 2026-05-01:

1. **DOJ-3714 reproducer on staging** — clicking "Continue" on a course inside a Path drops the `pathSlug` from the URL. The symptoms (broken breadcrumbs, lost analytics attribution, agent context blindness) are routing concerns, but they expose the deeper question: *what does course content reference, the schema or the slug?* If content references the schema, every schema change is a content rewrite. If content references the slug, the platform owns the schema.

2. **dojo-academy ↔ IDT separation of concerns** (DOJ-3705 through DOJ-3712) — IDT is being established as the multi-tenant authoring toolkit and dojo-academy as the content monorepo. The IDT README explicitly markets it as platform-agnostic. The first command we migrate that emits cross-references will need to know what convention to follow. **This change is the answer.**

## What success looks like

- IDT commands (`new-course`, `write-text-class`, `write-lesson`, etc.) emit symbolic refs instead of relative `.md` paths.
- DojoOS frontend ships a resolver utility that maps each symbolic ref to a live URL using the current route schema.
- Audit checklist (sibling file `audit-checklist.md`) shows all dojo-academy refs that need migration. Conversion happens in a follow-up PR.
- Documentation in `dojo-academy/CLAUDE.md` and IDT `CLAUDE.md` instructs course authors to use symbolic refs only.

## Why not alternatives

See `design.md` for the full alternatives table.

## Out of scope

- Implementation of the resolver in dojo-os (sibling — uses `tasks.md` checklist)
- Migration of existing dojo-academy content (sibling — uses `audit-checklist.md`)
- The path routing bug in dojo-os ([DOJ-3714](https://linear.app/dojo-coding/issue/DOJ-3714)) — this proposal documents the authoring-side convention; that issue is the platform-side rendering fix. The two are complementary.
