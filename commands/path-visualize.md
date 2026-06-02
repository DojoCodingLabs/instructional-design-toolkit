---
description: "Genera syllabus.html standalone para un path — header de metadata + per-milestone course list + per-course modules + per-module artifact-coverage table (✅/❌ por text/video/slides/quiz/challenge). Tool de QA de authoring, NO student-facing."
argument-hint: "[path-slug | absolute-path-to-path-dir]"
---

# /path-visualize — Path Syllabus Viewer (HTML standalone)

Dispatcher que invoca el skill `instructional-design-toolkit:path-visualize`.

Lee la estructura de un path (slug o ruta absoluta a un directorio de path) y
emite un único HTML auto-contenido al lado del `path-overview.md` del path:

```
<path-dir>/syllabus.html
```

El HTML es un visor de **cobertura estructural cross-course** — distinto del
sibling `course-visualize` (curvas pedagógicas de UN curso). Audiencia: path
owner + admin haciendo authoring QA, NO student-facing y NO instructor in-class
(para eso existen `workbook` y `slides-generate` respectivamente).

---

## Phase 1 — Resolve input

1. **Resolvé el argumento.** `$ARGUMENTS` puede ser:
   - Un slug (e.g. `agentic-ai`) — resolvelo a
     `<consumer-root>/content/paths/<slug>/`.
   - Una ruta absoluta a un directorio de path que contiene un
     `path-overview.md`.
   Si está vacío, pedile al usuario el slug o la ruta. Si el directorio o el
   `path-overview.md` no existen, abortá con un mensaje claro nombrando el
   path probado.
2. **Detectá el consumer root.** Caminá desde `cwd` hacia arriba buscando un
   `.claude-plugin/plugin.json` o un `content/paths/` sibling. Ese root es el
   ancla para resolver slugs de cursos en Phase 2 (debe contener
   `content/courses/<courseSlug>/`).
3. **Idempotencia.** Si `<path-dir>/syllabus.html` ya existe, sobrescribilo
   sin preguntar — es un archivo derivado, regenerable. Reportá la
   sobrescritura al usuario.

---

## Phase 2 — Read structure

1. **Parseá el frontmatter de `path-overview.md`.** Campos esperados:
   `slug`, `title`, `status`, `level`, `emblem`, `badge`, `faculty`,
   `description`, `cert_name`, `skills_demonstrated[]`, `earning_criteria[]`,
   `tech_pills[]`, `reward_points`, `milestones[]`. Cada `milestone` tiene
   `title`, `description`, `courses[]` (array de slugs ordenados).
2. **Tolerá variantes.** Si el path no tiene `milestones[]` (e.g. un path
   plano con `courses[]` al root del frontmatter), degradalo a un único
   milestone virtual con `title: "Courses"` y `courses` = ese array. Si el
   path tampoco tiene `courses[]`, emití warning visible en el HTML y un
   render con cero courses (no abortes).
3. **Para cada course slug:**
   - Resolvé `<consumer-root>/content/courses/<courseSlug>/`.
   - Si no existe, emití warning visible en el HTML para ese course
     (marcalo `MISSING` en la tabla) y continuá con el siguiente.
   - Leé `course-overview.md` si existe — extraé `title` para mostrar nombre
     legible. Si no, usá el slug.
   - Enumerá subdirectorios `module-*/` ordenados lexicográficamente
     (`module-01-*`, `module-02-*`, …).
   - Para cada módulo, leé `classes/` y clasificá cada archivo por prefijo:
     - `text-*.md` → bucket `text`
     - `video-*.md` → bucket `video`
     - `slides-*.html` → bucket `slides`
     - `quiz-*.md` → bucket `quiz`
     - `challenge-*.md` → bucket `challenge`
     Un bucket cuenta como `✅` si tiene ≥1 archivo, `❌` si está vacío.
4. **Agregá totales.** Computá el %-by-type cross-path:
   `coverage_pct[type] = modules_with_type / total_modules`. Mostralo en el
   header.
5. **Read-only.** No mutes ningún archivo de `content/`. Esto es un viewer.

---

## Phase 3 — Resolve design tokens (overlay-aware)

Buscá `<consumer-root>/instruction-bundle-spec.yaml`.

- **Si existe:** leé las secciones top-level `design` (palette, typography,
  voice) y `path_visualizer` (tree layout, `card_pattern` con `artifact_grid`
  order + `status_glyphs`). Usalas para componer el CSS y el JSON de
  configuración del HTML.
- **Si NO existe:** usá tokens **neutrales** + emití un comentario HTML
  visible (`<!-- WARNING: instruction-bundle-spec.yaml not found at <path>;
  rendering with neutral defaults -->`) **y** un banner amarillo en la UI
  (`<div class="warning">…</div>`) explicando que el visual está en modo
  default y mencionando el path esperado. NO crashees. NO hardcodees "Dojo".

### Neutral defaults (sin YAML)

- Palette: `#111` foreground, `#fafafa` background, `#666` muted, `#22c55e`
  success (✅), `#ef4444` danger (❌), `#f59e0b` warning.
- Typography: Inter (Google Fonts CDN — única dep externa permitida) con
  fallback `system-ui, -apple-system, sans-serif`.
- Voice: copy en EN-only neutral ("Path Syllabus", "Coverage", "Modules
  missing X"). Sin nombres de marca.

### Defaults aplicados al `card_pattern` cuando no hay YAML

- `artifact_grid` order: `[text, video, slides, quiz, challenge]`.
- `status_glyphs`: `{ present: "✅", missing: "❌" }`.

---

## Phase 4 — Emit `syllabus.html`

Escribí un único archivo HTML standalone en `<path-dir>/syllabus.html`. Reglas
duras:

- **Self-contained.** Todo el CSS y JS inline. NO `<link href="cdn...">`
  excepto Google Fonts para Inter (convención aceptada por el target).
- **Tamaño ≤500KB.** Imágenes opcionales solo si el path declara `badge` y el
  archivo existe localmente — referencialo por path relativo (`./badge.png`),
  no embebido en base64.
- **Mobile-responsive.** Mínimo 375px sin scroll horizontal. Optimizá 768px+
  con grid de cards más amplio.
- **Accesibilidad básica.** `<table>` con `<th scope="col">` para artifact
  columns; `<details>` para per-course expand (keyboard-nativo); `aria-label`
  en filter chips e icon buttons.

### Estructura del documento

1. **`<head>`** — título `"<path.title> — Syllabus"`, meta viewport, Google
   Fonts link para Inter, `<style>` inline con los tokens resueltos en
   Phase 3.
2. **Header sticky** — `<h1>` con `emblem + title`, sub-line con `faculty +
   level + cert_name`, badge image si existe. Aggregate coverage strip:
   ```
   Text 87% · Video 62% · Slides 41% · Quiz 94% · Challenge 58%
   ```
   Bonus: cada métrica como mini progress bar (CSS-only, sin SVG library).
3. **Search box + filter chips** — `<input type="search" placeholder="Search
   courses / modules (Ctrl+K)">`. Chips: "Missing text", "Missing video",
   "Missing slides", "Missing quiz", "Missing challenge". Click toggles
   `[hidden]` en rows que no matchean.
4. **Per-milestone section** — `<section>` con `<h2>milestone.title</h2>`
   + `<p>milestone.description</p>` + container de course cards.
5. **Per-course `<details>`** — `<summary>` con course title + count de
   módulos + mini-coverage del course. Al expandir, `<table>` con una row
   por módulo:

   ```
   | Module | Text | Video | Slides | Quiz | Challenge |
   |--------|------|-------|--------|------|-----------|
   | 01 — The Builder Mindset | ✅ (3) | ✅ (5) | ✅ (5) | ✅ (1) | ✅ (1) |
   | 02 — Scope and Map       | ✅ (4) | ❌    | ❌    | ✅ (1) | ❌    |
   ```

   El `(N)` muestra el count de archivos en ese bucket — útil para detectar
   módulos con cobertura desbalanceada (e.g. 8 texts vs 0 videos).
6. **Footer** — generador stamp (`generated by /idt:path-visualize at
   <ISO date>`), warning banner si Phase 3 cayó en defaults.

### Search + filter (JS inline, vanilla)

- `Ctrl+K` / `Cmd+K` focusea el search box (prevent default).
- Search filtra por substring case-insensitive sobre `data-search` atribute
  de cada row (course title + module name concatenados).
- Filter chips son toggleable; múltiples chips activos = AND lógico ("rows
  que matchean TODOS los missing filters seleccionados").
- Sin lag a 100+ módulos — usá `requestAnimationFrame` para el filter loop
  si el path tiene >50 módulos totales.

### Sortable

- Click en `<th>` de la columna `Module` ordena alfabético/reverso.
- Click en `<th>` de un artifact-type ordena por presencia (presents first /
  missing first toggle).

---

## Phase 5 — Validate output

Antes de reportar éxito:

1. **File written.** Confirmá que `<path-dir>/syllabus.html` existe y pesa
   ≤500KB.
2. **HTML well-formed.** No tags abiertos sin cerrar; `<details>` siempre
   tiene `<summary>`; tablas tienen `<thead>` + `<tbody>`.
3. **Voice neutrality (base).** El cuerpo del HTML generado por la base
   skill NO debe contener la cadena `"Dojo"` ni nombres de marca específica.
   Esos vienen del overlay del consumer (vía
   `instruction-bundle-spec.yaml`). Si el YAML del consumer aporta voice,
   esa voice puede contener nombres de marca — es Layer 3 legítima.
4. **Aggregate coverage suma.** `coverage_pct[type]` ∈ [0, 100] y consistente
   con el conteo render-by-render (no recomputes con datos distintos).

Si alguna falla, emití warning + abortá la escritura (preservá la versión
previa del `syllabus.html` si existía).

---

## Phase 6 — Report

Al usuario, retorná:

- Path absoluto del `syllabus.html` generado.
- Conteo total: milestones, courses, módulos, archivos de classes.
- Coverage %-by-type.
- Lista de courses `MISSING` (slugs referenciados en `milestones[].courses[]`
  que no resolvieron a un directorio en `content/courses/`).
- Lista de módulos con coverage < 60% (potenciales gaps de authoring).
- Warning si Phase 3 cayó en defaults (`instruction-bundle-spec.yaml`
  ausente).

Sugerencia next-step: `/idt:course-visualize <course-slug>` para zoom-in en
un course con coverage baja.

---

## Overlay invocation (post-base-draft)

Después de producir el base draft (HTML neutral L1+L2 con coverage rollup),
seguí `${CLAUDE_PLUGIN_ROOT}/assets/runtime/overlay-protocol.md` para
descubrir y aplicar overlays del consumer. El runtime camina
`<cwd>/.claude-plugin/plugin.json`, busca skills que declaren
`overlay_target: ["path-visualize"]` en su frontmatter, los ordena por
`overlay_priority` y los aplica en orden.

Para este comando, esperá (cuando un consumer está instalado):

- **Overlays estructurales (prioridad ~50)** — pueden agregar columnas extra
  a la tabla de coverage (e.g. "rubric", "feedback embed"), enriquecer el
  header con métricas custom (e.g. cmi5 AU count por module), ajustar el
  threshold de "coverage baja" del 60% default.
- **Overlays de voz / editorial (prioridad ~100)** — pueden inyectar la
  paleta de marca, el typography stack del consumer, copy localizado de los
  filter chips, footer branding. Es Layer 3 — overlays consumer-side son la
  fuente legítima de identidad visual.

### Invariantes Layer 1 inmutables

Los overlays NO pueden mutar:

- El path slug del frontmatter (`path.slug`).
- Los course slugs referenciados en `milestones[].courses[]`.
- El stamp de generación (timestamp ISO + comando que lo emitió).
- Los buckets de artifact-type (`text`, `video`, `slides`, `quiz`,
  `challenge`) — son contrato con la disciplina cmi5/xAPI authoring.

Cualquier intento de mutación aborta la corrida con error apuntando al
`SKILL.md` del overlay ofensor (consistente con §5 del overlay-protocol).

### Contradicciones Layer 2 → warning visible, no abort

Ejemplos: un overlay declara una columna extra pero no aporta el accessor
para popularla; un overlay declara un threshold de coverage pero da un valor
fuera de [0, 100]. El runtime emite warning visible (consistente con §6.1
del overlay-protocol) pero no aborta — el output base sigue siendo válido.

### Token de plugin no resoluble

Si `${CLAUDE_PLUGIN_ROOT}` aparece como literal en lugar de expandirse a un
path real (caso de invocación CLI directa fuera del contexto de plugin
install), el comando DEBE:

1. Detectar la falla de resolución (read fail con "no such file or
   directory").
2. Emitir un **warning visible** en la respuesta nombrando el token y el
   path esperado.
3. Saltar la invocación de overlays graciosamente y entregar el base draft
   (`syllabus.html` neutral) sin transforms del consumer.
4. NO crashear. NO producir el base draft silenciosamente como si no hubiera
   overlays instalados — eso borra la distinción entre "no hay overlays" y
   "el subsistema de overlays no es alcanzable".

§6.1 de `assets/runtime/overlay-protocol.md` es autoritativo; este bloque es
solo un puntero.

---

## Notas de diseño

- **WHY standalone HTML, no SPA.** El usuario target abre el archivo
  localmente para debug de authoring. Single-file portability + offline =
  cero fricción. Una SPA forzaría servir desde dev server.
- **WHY path-level, no extender course-visualize.** `course-visualize`
  renderiza curvas pedagógicas (Bloom's, complexity ramp, ship milestones)
  de UN course. `path-visualize` renderiza una tabla de coverage estructural
  cross-course. Outputs divergen >70%; dos verbos, dos comandos
  (`feedback_dedicated_variant_over_premature_dry`).
- **WHY no Pathways embed.** Audiencia interna (path owner + admin), no
  estudiante. Embed forzaría iframe + auth + breaking API surface para un
  uso de debug local.
- **WHY tolerar variantes de path-overview.** Audit pre-implement: solo
  `agentic-ai` tiene `milestones[]` poblado; los otros 4 paths pueden
  diverger. La skill no debe asumir shape rígida.
- **L1 invariantes** (cmi5/xAPI `au_id`, `activity_type`, stable IDs) NO se
  tocan — el viewer es read-only sobre `content/`. Un overlay que pretenda
  mutarlos aborta vía Phase 4 validator + §5 del overlay-protocol.
