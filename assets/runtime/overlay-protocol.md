# IDT Overlay Protocol — Runtime

**Status:** Implemented (Phase 1, DOJ-3707)
**Contract source:** `openspec/changes/2026-05-base-overlay-pattern/design.md`
**Schema:** `assets/schemas/overlay-protocol.schema.json`

This document is the **executable specification** of the IDT Base + Overlay
discovery and invocation runtime. IDT is a markdown-driven Claude Code plugin —
there is no compiled runtime. The runtime IS Claude reading this file from a
SKILL.md hand-off and following the procedure step by step. The TypeScript-
shaped pseudocode is the contract; the prose below is what actually executes.

If you are an IDT skill (`new-course`, `course-audit`, `slides-preview`, or any
future authoring command) and you have just produced a Layer 1 + Layer 2 base
draft, jump to **§3 Invocation** and follow it in order.

---

## 1. Layer model (recap)

IDT separates three layers. The runtime treats them differently.

| Layer | Owner | Examples | Overlay rules |
|---|---|---|---|
| **L1 — Standard invariants** | IDT base, ships always | `au_id`, `activity_type`, all stable IDs, semver classification fields, Open Badges 3.0 / W3C VC stable fields | NEVER overridable. Validator (§5) aborts the run if an overlay mutates these. |
| **L2 — Pedagogy generic** | IDT base, ships always | SAM > ADDIE, Atomic Habits cue→reward, Irby coach/mentor/tutor distinction, Ship-First Design, Bloom's mapping, Kirkpatrick L1-L4 | Overlays MAY extend or annotate. Contradictions log a visible warning naming the overlay's `SKILL.md` path but do NOT abort. |
| **L3 — Editorial voice** | Consumer plugins (e.g. dojo-academy), opt-in | "Builder-First, AI-Native" voice, named frameworks like "Multi-Quading", `CONTEXT → CONCEPT → BUILD → SHIP → REFLECT` formula, deep-guide-challenge-brief rubric | Lives ONLY as overlays in consumer plugins. IDT base has zero L3 content. |

The base draft IDT produces is L1 + L2 only. L3 is layered on by overlays — or
not, when there are no overlays installed in the cwd.

---

## 2. Discovery

Discovery happens once per IDT command run, BEFORE the base draft is produced
(so the runtime knows whether overlays exist and can warn the user when none
are found in a context where they were expected — though "no overlays" is also
a valid voice-neutral path).

### 2.1 Inputs

- `cwd` — the absolute working directory where the IDT command was invoked
  (Claude Code provides this).
- `command` — the IDT command name without the `/idt:` prefix (e.g.
  `"new-course"`).

### 2.2 Procedure

Execute these steps in order. Each step lists the failure mode that applies if
something goes wrong; failure modes are restated in §6 in tabular form.

1. **Locate plugin manifests.** From `<cwd>`, search for
   `<cwd>/.claude-plugin/plugin.json`. Also follow any nested plugin sources
   declared by Claude Code (the IDT plugin itself is one such source — its
   built-in skills are not overlays even though they live in `skills/`; they
   are skipped because their frontmatter does not declare `overlay_target`).
   - If `<cwd>/.claude-plugin/plugin.json` does not exist: discovery completes
     with zero overlays. This is the normal "external consumer with no Dojo
     overlays installed" path. No warning.
   - If the file exists but cannot be parsed (JSON syntax error or schema
     violation): emit a visible warning naming the offending plugin path,
     **skip that plugin's overlays only**, and continue with whatever other
     plugins were found. Do NOT abort discovery globally — one malformed
     plugin must not silence the rest of the protocol.

2. **Scan each plugin's skills directory.** For every plugin discovered in
   step 1, list `<plugin-root>/skills/*/SKILL.md`. Parse each SKILL.md's YAML
   frontmatter. A skill is a candidate overlay iff its frontmatter contains a
   non-empty `overlay_target` array.

3. **Filter by command.** Keep only candidates whose `overlay_target` array
   includes the running `command`. Drop everything else silently — including
   skills whose `overlay_target` lists a command IDT does not implement
   (forward-compat for newer plugin versions; no warning).

4. **Order the remaining overlays.** Sort by:
   - `overlay_priority` ascending (lower number → applied first).
     - If `overlay_priority` is missing, default per the table below.
     - If `overlay_priority` is non-integer or out of [0, 1000]: emit a
       visible warning naming the overlay and treat it as the default for its
       declared kind (or 75 if the kind cannot be inferred).
   - Tie-breaker: alphabetical by `SKILL.md` path, **case-insensitive,
     forward-slash-normalized**. This makes ordering deterministic across
     macOS/Linux/Windows and across plugin discovery implementations.

   **Default `overlay_priority` tiers** (apply unless the overlay author has a
   documented reason to deviate):

   | Tier | Default value | When to use | Examples |
   |---|---|---|---|
   | Structural | `50` | Reshapes the document scaffold early — adds sections, marks load-bearing fields, enforces structural rules. | `dojo-academy/skills/content-standards` (CONTEXT→CONCEPT→BUILD→SHIP→REFLECT formula application, "text classes carry the course" load-bearing rule) |
   | Generic | `75` | Annotates / extends without changing voice. Locale enrichment, accessibility annotations, link-checking, alt-text injection. | locale variants, partner-co-branded link decoration |
   | Voice / editorial | `100` | Runs late, on the structured draft. Voice transforms ("the learner will" → "you will ship"), named-framework introductions, momentum endings. | `dojo-academy/skills/academy-philosophy` |

5. **Return the ordered list of overlays.** Pass it to §3 Invocation.

---

## 3. Invocation

Once the base draft is produced and discovery has returned an ordered overlay
list, run the overlays sequentially. Each overlay receives the prior overlay's
output (or the base draft, for the first overlay). The contract is in
`assets/schemas/overlay-protocol.schema.json` (`OverlayInput`, `OverlayOutput`,
`OverlaySkillFrontmatter`).

### 3.1 Pseudocode

The pseudocode below is the contract shape. It is reproduced from `design.md`
without modification — implementers (Claude executing this prose) MUST match
it byte for byte at the type boundary.

```typescript
type OverlayInput = {
  command: string                // e.g. "new-course"
  baseDraft: object              // cmi5/xAPI-shaped object (or command-shaped)
  context: { cwd: string, repo: string, locale?: 'en' | 'es' }
}

type OverlayOutput = {
  draft: object                  // mutated/annotated draft
  warnings?: string[]            // surfaced in user-facing Claude response
}

function applyOverlay(input: OverlayInput): OverlayOutput
```

### 3.2 Loop

For each overlay in the ordered list, in order:

1. Validate Layer 1 invariants on the **input** `baseDraft` (§5). If they fail
   here, the failure was caused by a previous overlay — abort with the path of
   the **previous** overlay.
2. Build an `OverlayInput`:
   - `command` — the running command name.
   - `baseDraft` — the prior step's output (the L1+L2 draft for the first
     overlay; the previous overlay's `draft` for subsequent ones). Treat as
     deeply read-only.
   - `context.cwd` — the discovery cwd.
   - `context.repo` — the conventional repo name (e.g. `"dojo-academy"`); used
     in warnings.
   - `context.locale` — populated if the running command knows the language
     (e.g. `course.meta.language` on `new-course`); omitted otherwise.
3. Hand off to the overlay skill — i.e. read its `SKILL.md` and execute the
   transforms it describes, providing the `OverlayInput` as input. The overlay
   returns an `OverlayOutput`.
4. **If the overlay throws / aborts / fails to return a well-formed
   OverlayOutput:** emit a visible warning in the user-facing Claude response
   naming the overlay's `SKILL.md` path, then **continue with the prior
   draft** as if this overlay had been a no-op. Do NOT silently skip — the
   warning is the difference between "no overlays installed" and "overlays
   installed but broken", and silent skip would erase that distinction.
5. **If the overlay returns successfully:**
   a. Validate Layer 1 invariants on the returned `draft` (§5). If they fail,
      ABORT the entire run with a clear error pointing at this overlay's
      `SKILL.md` path. Do not proceed to subsequent overlays. Do not write.
   b. Validate Layer 2 (§5.3). If it fails, log a visible warning naming this
      overlay's `SKILL.md` path but PROCEED. Layer 2 is defended opinion, not
      contract.
   c. Append the overlay's `warnings` array (if any) to the run-wide warning
      buffer.
   d. Set the prior draft to the returned `draft` and continue to the next
      overlay.

After the loop completes:

6. Run the Layer 1 validator one final time on the last overlay's `draft`
   (defense in depth — the per-overlay validation in step 5a should already
   have caught any violation, but a final check here protects against bugs in
   the loop itself).
7. Write the final draft via the normal IDT path (the writer agent — e.g.
   `cmi5-metadata-writer` for `new-course`).
8. Surface the run-wide warning buffer to the user in the Claude response —
   one warning per line, each prefixed by the overlay's `SKILL.md` path.

### 3.3 Empty overlay list

If discovery returned zero overlays, skip §3.2 entirely. Write the base draft
directly. Do NOT emit a warning — voice-neutral output is a valid path, not a
failure.

---

## 4. Layer 1 immutable fields (authoritative list)

These fields MUST NOT change across overlay invocations. The validator in §5
checks every one. This list is the source of truth — when in doubt, treat a
field as immutable.

### 4.1 Stable identifiers

Mutating any of these breaks every learner's xAPI history.

| Field path (course.schema.json) | Why immutable |
|---|---|
| `meta.id` | Course identifier. Anchors all xAPI statements. |
| `modules[].id` | Module identifier. Used in xAPI activity tree. |
| `modules[].au_id` | cmi5 Assignable Unit identifier. Required by spec to be stable. |
| `modules[].lessons[].id` | Lesson identifier. |
| `modules[].lessons[].au_id` | Optional cmi5 sub-AU identifier — immutable when present. |
| `modules[].classes[].id` | Class (video / quiz / challenge) identifier. |
| `capstone.id` | Capstone identifier. Persists across course revisions. |
| `capstone.assessment_criteria[].id` | Each criterion has a stable id. |

### 4.2 cmi5 / xAPI structural fields

| Field path | Why immutable |
|---|---|
| `modules[].au_id` | (also listed above — primary cmi5 invariant) |
| `modules[].classes_cmi5.activityType` | xAPI activityType IRI — must remain consistent across revisions. |

### 4.3 Semver classification

These fields drive the MAJOR / MINOR / PATCH classification used by
`/idt:course-revise`. Overlays MUST NOT influence release classification.

| Field path | Why immutable |
|---|---|
| `meta.version` | Current semver. |
| `meta.version_timeline[].type` | Each historical entry's `major | minor | patch` classification. |
| `meta.version_timeline[].version` | Each historical entry's version string. |
| `meta.version_timeline[].date` | Each historical entry's date. |

### 4.4 Open Badges 3.0 / W3C Verifiable Credentials

The v1.0 `course.schema.json` does not yet emit credentials, but the contract
reserves all credential stable fields as Layer 1 immutable. When credential
fields land in a future schema version, they inherit this rule automatically.

| Field path (reserved) | Notes |
|---|---|
| `credentials[].id` | Reserved — credential identifier when emitted. |
| `credentials[].issuer.id` | Reserved — issuer DID/URL. |
| `credentials[].credentialSubject.id` | Reserved — recipient identifier. |
| `credentials[].issuanceDate` | Reserved — append-only timestamp. |

---

## 5. Layer 1 invariant validator

Run this validator **before** the first overlay, **after every overlay
return**, and once **after the loop completes**. The cost is negligible
(string comparisons against the input snapshot) and the protection is
load-bearing.

### 5.1 Procedure

1. Take a deep snapshot of the input draft BEFORE handing it to the overlay
   (or, for the pre-loop pass, BEFORE the loop starts). Store every Layer 1
   field path from §4 with its value.
2. After the overlay returns, walk the `draft` and re-collect every Layer 1
   field path with its current value.
3. For each field path:
   - If the field exists in the snapshot but is missing or null in the new
     draft → **violation** (overlay deleted an immutable field).
   - If the field's value changed (string-equal comparison; deep equality for
     arrays/objects) → **violation**.
   - If the field is newly present in the new draft but was absent in the
     snapshot → **NOT a violation** (overlays may add fields the base did not
     populate, e.g. a credential block).
4. If any violation is recorded, ABORT the run with an error of the form:

   ```
   ERROR: Overlay <skill-path> violates Layer 1 invariant.
   Field: <field-path>
   Before: <snapshot-value>
   After:  <new-value>
   See: assets/runtime/overlay-protocol.md §4 for the immutable-fields list.
   The run has been aborted; no files were written.
   ```

   Do NOT continue to subsequent overlays. Do NOT write the draft. Surface the
   error in the user-facing Claude response.

### 5.2 Snapshotting tip

For the markdown-driven runtime, "snapshot" means: extract the Layer 1 field
values into a flat lookup table (e.g. JSON Pointer paths → values) before the
overlay runs, and compare the same paths after. This is cheaper than full
deep-equality of the whole draft and matches the intent of the contract
(only L1 paths are immutable; the rest of the draft is fair game).

### 5.3 Layer 2 contradiction check (advisory, non-blocking)

After Layer 1 passes, the runtime SHOULD also check for obvious Layer 2
contradictions. This is a best-effort lint, not a contract. Examples:

- Overlay output sets `analysis.blooms_progression` to a non-monotonic array
  when the base draft was monotonic — Bloom's progression climb violation.
- Overlay output removes `capstone.deliverable` or sets it to "reflexioná
  sobre…" — Ship-First Design violation.
- Overlay output removes a module's `feedback_form` after the base provided
  one — Kirkpatrick L1 evaluation framework violation.

When a Layer 2 contradiction is detected: log a visible warning of the form
`"WARN: Overlay <skill-path> may contradict Layer 2 (<which framework>)"` and
**continue**. Do NOT abort.

---

## 6. Failure mode (full table)

Authoritative reference — matches `design.md` row-for-row. Each situation
below has a single, deterministic behavior.

| Situation | Behavior |
|---|---|
| Overlay throws an error (or returns a malformed `OverlayOutput`) | Emit a **visible warning in the user-facing Claude response** naming the failed overlay and its `SKILL.md` path. Continue with the prior draft. Visibility matters: silent skip would make "no overlays installed" indistinguishable from "overlays installed but broken". |
| Overlay output violates Layer 1 invariant (mutates `au_id`, deletes `meta.id`, changes `meta.version_timeline[].type`, etc.) | ABORT the run with a clear error pointing at the overlay's `SKILL.md` path and naming the offending field. Do not write. |
| Overlay output contradicts Layer 2 (e.g. flattens Bloom's progression, removes `capstone.deliverable`) | Log a visible warning naming the overlay. PROCEED. Layer 2 is defended opinion, not contract. |
| `<cwd>/.claude-plugin/plugin.json` is missing | Discovery returns zero overlays. No warning. (Voice-neutral output is a valid path.) |
| `<cwd>/.claude-plugin/plugin.json` is malformed (JSON parse error or schema validation failure) | Emit a visible warning naming the offending plugin path. **Skip that plugin's overlays only** — other plugins still discovered. Continue with whatever overlays were found. |
| A plugin declares `overlay_target` for a command IDT does not implement | Silently ignore the registration. No warning. (Forward-compat for newer plugin versions.) |
| No overlays found for the running command | Emit the base draft directly. Voice-neutral, cmi5-compliant. No warning. |

---

## 7. Determinism guarantees

The runtime guarantees these properties — if they ever break, that is a bug
worth a Linear issue:

1. **Deterministic ordering.** Two runs of the same command from the same cwd,
   against the same plugin set, produce overlays in identical order. The
   `(overlay_priority, alphabetical-path)` key is total.
2. **Idempotent base path.** A run from a directory with no overlays produces
   byte-identical output to the same command run with discovery disabled (a
   useful fallback for debugging).
3. **L1 atomicity.** Either every Layer 1 invariant survives the entire
   pipeline, or the run aborts before writing. There is no intermediate state
   on disk where overlays partially mutated immutable fields.
4. **No silent overlay failures.** Every failed overlay produces a visible
   warning in the user-facing response. Operators always know when an overlay
   fired and produced nothing useful.

---

## 8. How an IDT skill invokes this protocol

Inside an authoring skill (e.g. `skills/new-course/SKILL.md`), the overlay
section instructs Claude to:

1. Produce the Layer 1 + Layer 2 base draft per the skill's normal flow
   (cmi5 metadata, Bloom's mapping, Kirkpatrick forms, etc.).
2. Read this file (`assets/runtime/overlay-protocol.md`).
3. Execute §2 Discovery using `<cwd>` and the command name (e.g. `new-course`).
4. If discovery returned overlays: execute §3 Invocation in order, with the
   §5 validator running between each overlay.
5. Write the final draft via the skill's normal writer path.
6. Surface the warning buffer to the user.

For Phase 1, three skills are wired:

- `skills/new-course/SKILL.md`
- `skills/course-audit/SKILL.md`
- `skills/slides-preview/SKILL.md`

Future authoring commands (DOJ-3708 migrations: `write-text-class`,
`write-lesson`, `write-module`, `write-quiz`, `write-challenge`,
`write-video-script`, etc.) will follow the same pattern.

---

## 9. Out of scope (for Phase 1)

- Phase 2 (DOJ-3710): converting `dojo-academy/skills/{academy-philosophy,
  content-standards}` to overlays that comply with this protocol.
- Phase 3 (DOJ-3711): removing the duplicate `audit-course` and `plan-course`
  commands from dojo-academy.
- Phase 4: capability spec at `openspec/specs/overlay-protocol/spec.md`,
  authored after Phases 1–3 ship.
- Cross-overlay dependencies (overlay A requires overlay B). The contract
  treats overlays as independent. Author each overlay to be safe under any
  ordering at its tier.
- Overlay versioning. The current contract is implicit-v1. A future version
  bump will land alongside breaking changes to `OverlayInput`/`OverlayOutput`.
