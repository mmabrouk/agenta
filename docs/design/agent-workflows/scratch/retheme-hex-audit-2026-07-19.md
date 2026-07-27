# Retheme hex audit — navy/zinc to warm brand palette

Scope: everything under `web/`, grepped case-insensitively for the old palette's raw hex
and `rgba(5, 23, 41 …)` literals, excluding the token source-of-truth files that the
retheme will edit directly (`palette.ts`, `legacy-shim.ts`, `theme-variables.css`,
`tokens/*.json`, `theme/antd-overrides.generated.ts`, `scripts/generate-tailwind-tokens.ts`)
and build output (`node_modules`, `.next`, `dist`, `build`).

Values searched: `#1c2c3d`, `#051729`, `#f5f7fa`, `#eaeff5`, `#d6dee6`, `#bdc7d1`,
`#758391`, `#586673`, `#394857`, `rgba(5, 23, 41 …)` (both spacing variants), and the
dark-mode/accent values `#141414`, `#424242`, `#303030`, `#d1d151`, `#c2d54a`.

## 1. Summary

- **255 matching lines** across **50 files** outside the token files.
- **49 lines need a code edit** (TOKENIZE) before the retheme will reach them.
- **~72 lines are already var-backed** (`var(--ag-c-XXXXX, #xxxxxx)` Tailwind-arbitrary
  fallback syntax) or are JSDoc comments — no risk, LEAVE as-is, listed for completeness.
- **132 raw hits** are decorative path strokes/fills inside two near-identical, apparently
  **unused** SVG assets (`onboard-page-grids.svg`, OSS + EE) — LEAVE / verify-and-drop.
- **~10 lines** are members of fixed, hash-indexed decorative palettes (avatar
  initials-color pairs, template icon colors, an avatar gradient list) where the old-navy
  hex is coincidental, not a theme role — LEAVE.
- **1 real bug found in passing**: `web/oss/tailwind.config.ts`'s Tremor "brand" color
  block calls antd's `theme.getDesignToken()` with no seed, which returns antd's **stock
  default blue** (`colorPrimary: #1677ff`), not Agenta's navy — so today, before any
  retheme, Tremor's brand colors already don't match the app. It won't pick up the new
  brand color either; needs a code fix, not just a palette edit. See §3.

| Recommendation | Count (lines) |
|---|---|
| TOKENIZE | 49 |
| LEAVE (already var-backed, fallback hex is inert) | 51 |
| LEAVE (JSDoc comment only) | 20 |
| LEAVE (decorative/hash-indexed palette, coincidental hex) | 6 |
| LEAVE (comment referencing already-tokenized code elsewhere) | 1 |
| LEAVE (orphaned SVG asset, 132 raw occurrences) | 2 files |

## 2. Per-hit table

Grouped by file. "Var-backed (LEAVE)" rows use the `var(--ag-c-X, #fallback)` idiom — the
CSS custom property is always defined globally by `theme-variables.css`, so the literal
hex fallback never actually paints; it's inert until/unless someone deletes the var
definition, which is not something this retheme does.

### SVG assets (decorative, likely orphaned)

| File | Lines | Excerpt | Kind | Recommendation |
|---|---|---|---|---|
| `web/ee/public/assets/onboard-page-grids.svg` | 7–72 (66 hits: 63×`#051729` stroke/fill, 3×`#758391` fill) | `<path ... stroke="#051729" .../>` | SVG asset, EE | **LEAVE** — grepping `web/oss/src` and `web/ee/src` for `onboard-page-grid` finds **no importer**. This appears to be an orphaned public asset (dead weight, not wired into any page). Recommend confirming with design whether it's still needed; if kept, it must be manually re-exported in the new brand color since a static SVG in `public/` cannot consume CSS vars. |
| `web/oss/public/assets/onboard-page-grids.svg` | 7–72 (66 hits, identical pattern) | same | SVG asset, OSS | **LEAVE** — same as above, byte-identical duplicate of the EE asset. |

### React components — inline style / JS color constant (needs edit)

| File:Line | Excerpt | Kind | Recommendation |
|---|---|---|---|
| `web/oss/src/components/DeploymentsDashboard/components/DeploymentCard/index.tsx:30` | `borderColor: selectedEnv === env.name ? "#1C2C3D" : undefined,` | inline style, OSS | **TOKENIZE** → `var(--ag-colorPrimary)` (selected-card emphasis border) |
| `web/oss/src/components/EvalRunDetails/components/views/ConfigurationView/index.tsx:392` | `? "#1c2c3d"` (comparison swatch color, base run) | inline style ternary, OSS | **TOKENIZE** → `var(--ag-colorPrimary)` |
| `web/oss/src/components/Layout/assets/styles.ts:10` | `background: themeMode === "dark" ? "#141414" : "#ffffff",` | react-jss, OSS | **TOKENIZE** → `theme.colorBgContainer` (the JSS `theme` param already carries `colorBorderSecondary`/`colorTextTertiary` used a few lines down — this line should use the same object instead of a manual ternary) |
| `web/oss/src/components/Layout/assets/styles.ts:46` | `backgroundColor: "#1c2c3d",` (top banner) | react-jss, OSS | **TOKENIZE** → `theme.colorPrimary` |
| `web/oss/src/components/Onboarding/OnboardingCard.tsx:763` | `dark:!text-[#141414]` (dark-mode text on the bright primary button) | Tailwind arbitrary, dark variant, OSS | **TOKENIZE, with a gap** → this literal duplicates `componentsDark.Button.primaryColor` (`palette.ts`), which is an antd `components` override and is **not** emitted as a CSS var today. Recommend adding a semantic var (e.g. `--ag-colorPrimaryContrast`) to the generator as part of the retheme rather than hand-picking a new literal here. |
| `web/oss/src/components/pages/agent-home/assets/templates.ts:153` | `color: "#1c2c3d",` (PR reviewer template) | data config, OSS | **LEAVE** — one of ~30 `AGENT_TEMPLATES` entries, each hand-assigned its own decorative accent hex (`#7c2d12`, `#0d9488`, `#be123c`, …), used to color the template's initials avatar. Only 2 of 30 happen to equal the old brand navy; it reads as "one more crayon" in a rainbow list, not a themed role. Safe to leave; will just look slightly duplicative until someone diversifies it. |
| `web/oss/src/components/pages/agent-home/assets/templates.ts:973` | `color: "#1c2c3d",` (Docs Q&A template) | data config, OSS | **LEAVE** — same reasoning as above. |
| `web/oss/src/components/pages/observability/components/ObservabilityHeader/useBatchAddTracesToQueue.tsx:37` | `const PRIMARY_COLOR = "#1c2c3d"` — comment above it literally says *"matches antd's `colorPrimary`"* | JS constant, OSS | **TOKENIZE** → `var(--ag-colorPrimary)`. The author's own comment documents the intent; this is a clean, low-risk fix. |
| `web/oss/src/components/pages/prompts/components/PromptsHouseIcon.tsx:10` | `<HouseIcon size={14} color="#1C2C3D" .../>` | icon component prop, OSS | **TOKENIZE** → `var(--ag-colorIcon)` / `var(--ag-colorIconHover)` per the active/hover branch |
| `web/oss/src/components/pages/prompts/components/SetupWorkflowIcon.tsx:23` | `fill="#1C2C3D"` | inline SVG icon, OSS | **TOKENIZE** → `var(--ag-colorText)` or `var(--ag-colorIcon)` |
| `web/oss/src/components/pages/prompts/hooks/usePromptsColumns.tsx:32` | `<FolderFilled style={{fontSize: 16, color: "#BDC7D1"}} />` | inline style, OSS | **TOKENIZE** → `var(--ag-colorTextQuaternary)` |
| `web/oss/src/components/pages/prompts/modals/DeleteFolderModal.tsx:32` | `<FolderFilled style={{fontSize: 16, color: "#BDC7D1"}} />` | inline style, OSS | **TOKENIZE** → `var(--ag-colorTextQuaternary)` |
| `web/oss/src/lib/helpers/colors.ts:44` | `{backgroundColor: "#eaeff5", textColor: "#758391"},` (1 of 12 in `colorPairs`) | data config, OSS | **LEAVE** — a fixed, hash-indexed avatar initials-color palette (12 arbitrary pairs, e.g. `#BAE0FF`/`#1677FF`, `#D9F7BE`/`#389E0D`). This one pair coincidentally equals `zinc-2`/`zinc-6`; it's decorative, not a theme role. Note: this exact 12-pair list is duplicated in `web/packages/agenta-ui/src/components/presentational/avatar/utils.ts` — a dedup opportunity, unrelated to the retheme. |
| `web/oss/src/lib/helpers/colors.ts:4` | `"linear-gradient(to bottom right, #424242, #9F1239, #560BAD)",` (1 of 14 gradients) | data config, OSS | **LEAVE** — decorative avatar-gradient stop; `#424242` here is unrelated to the dark-mode border/zinc-4 role despite the coincidental hex match. |
| `web/packages/agenta-ui/src/components/presentational/avatar/utils.ts:14` | `{backgroundColor: "#eaeff5", textColor: "#758391"},` | data config, package | **LEAVE** — duplicate of the OSS avatar palette above, same reasoning. |
| `web/packages/agenta-ui/src/CellRenderers/EvaluatorMetricBar.tsx:21` | `const FALSE_SEGMENT_COLOR = "#bdc7d1"` | JS constant, package | **TOKENIZE** → `var(--ag-colorTextQuaternary)` |
| `web/packages/agenta-ui/src/CellRenderers/EvaluatorMetricBar.tsx:22` | `const FALSE_LABEL_COLOR = "#586673"` | JS constant, package | **TOKENIZE** → `var(--ag-colorTextTertiary)` |
| `web/packages/agenta-ui/src/CellRenderers/EvaluatorMetricBar.tsx:92` | `const TRACK_BG = "rgba(5, 23, 41, 0.06)"` | JS constant, package | **TOKENIZE** → `var(--ag-colorFillSecondary)` (exact alpha match) |
| `web/packages/agenta-ui/src/type-chip/TypeChip.tsx:184` | `e.currentTarget.style.boxShadow = "0 1px 3px rgba(5, 23, 41, 0.12)"` (hover handler) | inline JS style, package | **TOKENIZE** → `var(--ag-rgba-051729-10)` (closest existing alpha step; exact 0.12 isn't defined) |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/agentTemplate/itemDescriptors.tsx:188` | `color: "#1c2c3d",` | data config, package | **TOKENIZE** → `var(--ag-colorText)` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/agentTemplate/ProviderCredentialsSection.tsx:141` | `... text-[10px] font-semibold text-[#586673]"` | Tailwind arbitrary, package | **TOKENIZE** → `var(--ag-colorTextTertiary)`. Note the same `className` already correctly uses `var(--ag-colorBorderSecondary)` earlier in the string — this is the one raw literal in an otherwise tokenized line. |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/agentTemplate/HarnessSelectControl.tsx:40` | `pi_agenta: {label: "Pi (Agenta)", short: "Ag", color: "#1c2c3d"},` | data config, package | **TOKENIZE** → `var(--ag-colorText)` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/HarnessSelectControl.tsx:72` | `return {label, short, color: "#586673"}` | JS fallback, package | **TOKENIZE** → `var(--ag-colorTextTertiary)` |

### Recharts / chart config (stroke/grid colors, needs edit)

| File:Line | Excerpt | Kind | Recommendation |
|---|---|---|---|
| `web/oss/src/components/EvalRunDetails/components/EvaluatorMetricsChart/BarChart.tsx:129` | `tickLine={{stroke: "#05172933"}}` | Recharts config, OSS | **TOKENIZE** → `var(--ag-rgba-051729-18)` (`33`/`ff` ≈ 0.2 alpha, closest defined step) |
| `web/oss/src/components/EvalRunDetails/components/EvaluatorMetricsChart/BarChart.tsx:139` | `stroke="#05172933"` | Recharts config, OSS | **TOKENIZE** → same as above |
| `web/oss/src/components/EvalRunDetails/components/EvaluatorMetricsChart/HistogramChart.tsx:102` | `tickLine={{stroke: "#05172933"}}` | Recharts config, OSS | **TOKENIZE** → `var(--ag-rgba-051729-18)` |
| `web/oss/src/components/EvalRunDetails/components/EvaluatorMetricsChart/HistogramChart.tsx:112` | `stroke="#05172933"` | Recharts config, OSS | **TOKENIZE** → `var(--ag-rgba-051729-18)` |
| `web/oss/src/components/EvalRunDetails/components/views/OverviewView/components/OverviewPlaceholders.tsx:124` | `<PolarGrid stroke="#EAEFF5" />` | Recharts config, OSS | **TOKENIZE** → `var(--ag-colorBorderSecondary)` |
| `web/oss/src/components/Evaluations/MetricDetailsPopover/assets/ResponsiveMetricChart.tsx:291` | `stroke="#05172933"` | Recharts config, OSS | **TOKENIZE** → `var(--ag-rgba-051729-18)` |

### Tailwind arbitrary-value shadows / backgrounds (needs edit)

| File:Line | Excerpt | Kind | Recommendation |
|---|---|---|---|
| `web/oss/src/components/DrillInView/DrillInFieldHeader.tsx:243` | `...shadow-[0_12px_32px_rgba(5,23,41,0.16)]"` | Tailwind arbitrary, OSS | **TOKENIZE** → `shadow-[0_12px_32px_var(--ag-rgba-051729-18)]` (no exact 0.16 step defined; 0.18 is closest) |
| `web/oss/src/components/DrillInView/TraceSpanDrillInView.tsx:517` | `...shadow-[0_8px_24px_rgba(5,23,41,0.12)] ...` (same className already uses `var(--ag-rgba-051729-14)` and `var(--ag-c-FFFFFF)` for its border/background) | Tailwind arbitrary, OSS | **TOKENIZE** → `var(--ag-rgba-051729-10)`; follow the file's own precedent two tokens earlier in the same className |
| `web/oss/src/components/pages/settings/Secrets/SecretProviderTable/index.tsx:124` | `className="bg-[#0517290F] px-2 py-[1px] m-0"` | Tailwind arbitrary, OSS | **TOKENIZE** → `bg-[var(--ag-c-0517290F)]` (legacy shim already defines this exact 8-digit value) or `bg-colorFillTertiary` |
| `web/oss/src/components/TemplateStrip/components/CopiedToast.tsx:33` | `...shadow-[0_10px_26px_rgba(5,23,41,0.35)] ...` | Tailwind arbitrary, OSS | **TOKENIZE**, flagged: no alpha step at 0.35 exists (steps jump 0.18 → 0.45). Either accept `var(--ag-rgba-051729-45)` (visibly stronger) or add a new `alphaFill` entry during the retheme. |
| `web/oss/src/components/EvalRunDetails/components/views/SingleScenarioViewerPOC/ScenarioAnnotationPanel/RunOverlay.tsx:12` | `bg-white/90 dark:bg-[#141414]/90` | Tailwind arbitrary, dark variant, OSS | **TOKENIZE** → drop the `dark:` branch entirely and use `bg-colorBgContainer/90` — `colorBgContainer` is already CSS-var-backed and theme-aware, so one class covers both modes (today's `bg-white/90` light branch is also technically hardcoded, just not part of this hex set). |

### Plain CSS files (needs edit)

| File:Line | Excerpt | Kind | Recommendation |
|---|---|---|---|
| `web/oss/src/styles/animations.css:40` | `stroke: #d6dee6;` (in `@keyframes colorChange`) | CSS keyframe, OSS | **TOKENIZE** → `var(--ag-zinc-3)`. Note the same keyframe hardcodes `#36cfc9` (teal, out of this audit's scope) as its other stop — fix together for a coherent animation. |
| `web/oss/src/styles/code-editor-styles.css:594` | `--editor-gutter-border: #303030;` (inside `.dark .code-editor .editor-code`) | CSS custom prop, OSS | **TOKENIZE** → `var(--ag-colorBorderSecondary)`. Already `.dark`-scoped, so this is a safe 1:1 swap. |
| `web/oss/src/styles/editor-theme.css:696` | `background: #141414;` (`.dark .toolbar`) | CSS, OSS (Lexical editor theme) | **TOKENIZE** → `var(--ag-colorBgContainer)` |
| `web/oss/src/styles/editor-theme.css:702` | `background-color: #303030;` (`.dark .toolbar .divider`) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBorderSecondary)` |
| `web/oss/src/styles/editor-theme.css:720` | `border-left-color: #424242;` (`.dark .editor-quote`) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBorder)` |
| `web/oss/src/styles/editor-theme.css:730` | `background-color: #141414;` (`.dark .editor-tableFrozenRow ...`) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBgContainer)` |
| `web/oss/src/styles/editor-theme.css:741` | `border-color: #424242;` (`.dark .editor-tableCell`) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBorder)` |
| `web/oss/src/styles/editor-theme.css:746` | `border-color: #424242;` (`.dark .editor-tableFrozenRow ... :after`) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBorder)` |
| `web/oss/src/styles/evaluations.css:171` | `background-color: rgba(5, 23, 41, 0.2);` | CSS, OSS | **TOKENIZE** → `var(--ag-rgba-051729-18)` (closest existing step) |
| `web/oss/src/styles/globals.css:797` | `border-color: #d6dee6;` (`.auth-locked-input` disabled, light-only, no `.dark` counterpart today) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBorderSecondary)` |
| `web/oss/src/styles/globals.css:842` | `background: #141414;` (`.dark .evaluation-filters .ant-segmented-item`) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBgContainer)` |
| `web/oss/src/styles/globals.css:855` | `border-bottom-color: #303030 !important;` (`.dark .comparison-table` expanded row) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBorderSecondary)` |
| `web/oss/src/styles/globals.css:873` | `border-color: #424242;` (`.dark .auth-locked-input` disabled) | CSS, OSS | **TOKENIZE** → `var(--ag-colorBorder)` |
| `web/oss/src/styles/human-evals.css:2` | `--not-available-stripe: #eaeff5;` (`body` level, no dark override in this file) | CSS custom prop, OSS | **TOKENIZE** → `var(--ag-colorBorderSecondary)` or `var(--ag-zinc-2)` |

**Note on `editor-theme.css` / `code-editor-styles.css` / `globals.css`:** these three files
are the only place where dark-mode literals (`#141414`/`#303030`/`#424242`) are hand-written
inside real `.dark`-scoped selectors rather than sourced from `theme-variables.css`. They
don't import `palette.ts` at all — they're the biggest blind spot for the dark-mode leg of
the retheme, because a `pnpm generate:tailwind-tokens` run will silently leave all 9 of
these rules on the old navy-derived dark grays.

### Already tokenized — `var(--ag-c-X, #fallback)` idiom (LEAVE)

These use Tailwind's arbitrary-value syntax `[var(--ag-c-XXXXX, #fallback)]`. The CSS
custom property is always defined by `theme-variables.css` at `:root`, so the hex fallback
is inert — it only renders if the variable itself is undefined, which doesn't happen here.
No action needed; listed for completeness per the audit instructions.

| File | Lines | Var used |
|---|---|---|
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/agentTemplate/ItemRow.tsx` | 108, 274 | `--ag-c-EAEFF5` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/agentTemplate/useModelHarness.tsx` | 483, 490, 493 | `--ag-c-EAEFF5`, `--ag-c-97A4B0` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/CodeEditor.tsx` | 77 | `--ag-c-EAEFF5` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/InstructionsDrawer.tsx` | 149, 170, 171, 199, 212, 219, 221 | `--ag-c-EAEFF5`, `--ag-c-586673`, `--ag-c-97A4B0`, `--ag-rgba-051729-04` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/JsonObjectEditor.tsx` | 54 | `--ag-c-EAEFF5` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/MarkdownEditor.tsx` | 216, 225, 236, 284, 317 | `--ag-c-97A4B0`, `--ag-c-EAEFF5`, `--ag-c-586673`, `--ag-c-BDC7D1` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/SkillFormView.tsx` | 216, 225, 235 | `--ag-c-EAEFF5`, `--ag-c-586673`, `--ag-c-1C2C3D` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/SkillUploadZone.tsx` | 60, 61, 68, 70 | `--ag-c-D6DEE6`, `--ag-c-586673`, `--ag-c-F5F7FA` |
| `web/packages/agenta-entity-ui/src/DrillInView/SchemaControls/WorkflowReferenceSelector.tsx` | 478, 495 | `--ag-c-EAEFF5` |
| `web/packages/agenta-ui/src/components/presentational/section/ConfigAccordionSection.tsx` | 194, 228 | `--ag-c-586673`, `--ag-c-EAEFF5` |
| `web/packages/agenta-ui/src/Editor/MarkdownToolbar.tsx` | 78, 79, 314 | `--ag-c-586673`, `--ag-c-EAEFF5`, `--ag-c-1677FF` |

### Needs a small fix even though partly var-backed

| File:Line | Excerpt | Kind | Recommendation |
|---|---|---|---|
| `web/packages/agenta-annotation-ui/src/components/AnnotationSession/ConfigurationView.tsx:58` | `style={{borderBottom: collapsed ? undefined : "1px solid rgba(5,23,41,0.06)"}}` — one line above, the sibling `<div>` already uses `bg-[var(--ag-rgba-051729-02)]` correctly | inline style, package | **TOKENIZE** → `var(--ag-rgba-051729-06)` (exact match already exists; trivial fix) |
| `web/packages/agenta-annotation-ui/src/components/AnnotationSession/ConfigurationView.tsx:327` | `"var(--Components-Collapse-Global-lineWidth, 1px) solid var(--Colors-Neutral-Border-colorSplit, rgba(5, 23, 41, 0.06))"` | inline style, package (Figma-exported var name, hardcoded fallback) | **TOKENIZE** → replace the fallback with `var(--ag-colorSplit)` instead of the raw literal |
| `web/oss/src/components/EvalRunDetails/components/views/ConfigurationView/components/EvaluatorSection.tsx:224` | `"var(--Components-Collapse-Global-lineWidth, 1px) solid var(--Colors-Neutral-Border-colorSplit, rgba(5, 23, 41, 0.06))"` | inline style, OSS (same Figma-var pattern) | **TOKENIZE** → same fix, replace fallback with `var(--ag-colorSplit)` |

### Comments (no functional risk, LEAVE)

| File | Lines | Note |
|---|---|---|
| `web/packages/agenta-playground-ui/src/components/PlaygroundInputsBody/VariableCard.tsx:872` | `// beat the \`isEditorFocused && "!border-[#BDC7D1]"\` rule baked into` | Describes `SharedEditorImpl.tsx`'s actual rule, which is already `var(--ag-c-BDC7D1)` — the comment's shorthand is just stale/informal, not a real literal. **LEAVE**, optionally refresh the comment text for accuracy. |
| `web/packages/agenta-ui/src/utils/styles.ts` | 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 121, 127, 129, 131, 133, 137, 139, 141, 143, 145, 147, 149, 151, 163, 165, 167, 171, 183, 189, 191, 193, 195, 239, 241 | JSDoc header block ("Token Reference (from antd-tailwind.json): colorText: #1c2c3d …") and per-property doc comments listing the token's hex for human reference. No live code in this file uses raw hex — all real usage is via token-name strings (`colorText`, `colorTextSecondary`, …). **LEAVE**, recommend a documentation refresh pass after the retheme lands so these comments don't mislead the next reader, but zero functional/visual risk. |

## 3. Build-time consumers of the generated token files

Searched all of `web/` for importers of `tokens/antd-tailwind.json`,
`tokens/antd-themeConfig.json`, and `theme/antd-overrides.generated.ts`, plus any other
call to antd's `theme.getDesignToken()` or direct import of `palette.ts`.

| Consumer | Imports | When it runs | Bakes values? |
|---|---|---|---|
| `web/oss/tailwind.config.ts` | `tokens/antd-tailwind.json` (spread into `theme.extend.colors`) | Tailwind JIT, at `next dev`/`next build` time (Node) | **No for the spread** — `antdTailwind` values are static hex from the last generator run, same as `theme-variables.css`; regenerating and rebuilding picks up new values normally. **Yes for the Tremor block** — see below. |
| `web/oss/src/components/Layout/ThemeContextProvider.tsx` | `tokens/antd-themeConfig.json` (as `antdTokens`, passed into antd's `<ConfigProvider theme={{token: antdTokens, ...}}>`) and `theme/antd-overrides.generated.ts` (`DARK_TOKEN_OVERRIDES`, `darkComponents`) | Browser runtime, every render | **No** — these are read fresh from the committed JSON/TS files at runtime; once the generator is re-run and the files are committed, the running app picks up the new values with no separate build-time snapshot step. This is the expected, working path. |
| `web/scripts/generate-tailwind-tokens.ts` | `legacy-shim.ts`, `tokens/antd-themeConfig.json` (as its own output target), calls `theme.getDesignToken()` on the **dark** algorithm to snapshot dark values into `palette.ts`'s `componentsDark`/scale literals | Only when the generator itself is run | This is the generator — by design, it bakes a snapshot. Not a bug; it's how `palette.ts`'s dark literals were produced in the first place (see the file's own header comment). |

### The Tremor brand-color block — confirmed baked, and already wrong today

`web/oss/tailwind.config.ts` line 5:

```ts
import {theme} from "antd"
const token = theme.getDesignToken()   // called with NO seed/config argument
```

`theme.getDesignToken()` with no arguments returns antd's **stock default light seed
token**, not anything derived from Agenta's `ConfigProvider` (that provider only exists at
React runtime, in the browser; `tailwind.config.ts` runs in a separate Node process during
the Tailwind build step and has no access to it). Verified by running it directly:

```
colorPrimary: #1677ff
colorPrimaryBgHover: #bae0ff
colorPrimaryBorderHover: #69b1ff
colorPrimaryHover: #4096ff
colorPrimaryActive: #0958d9
```

That's antd's default blue, not Agenta's navy (`#1c2c3d`) — and it won't be the new brand
color either. Both the `tremor.brand` and `dark-tremor.brand` blocks (lines 148–176,
177–206) read from this same static `token`, so light and dark Tremor brand colors are
currently **identical** (both antd-default-blue), which also can't be right for a
dark-aware chart palette.

**Practical severity today:** low. `@tremor/react` is a declared dependency (its
`node_modules` path is even listed in `tailwind.config.ts`'s `content` array) but grepping
`web/oss/src` and `web/ee/src` finds **no component using `tremor-brand`/`dark-tremor`
classes** — only one file (`SectionPrimitives.tsx`) uses the unrelated `shadow-tremor-input`
box-shadow token. So nothing visibly breaks today.

**Verdict:** it will **not** pick up the new `colorPrimary` automatically after
regeneration — it was never wired to Agenta's palette in the first place, seed-token or
CSS-var. It needs a code fix regardless of the retheme: either seed `theme.getDesignToken()`
with Agenta's actual light/dark tokens (e.g. import `antdTailwind`/`antdTheme` JSON instead
of calling the bare antd default), or point the `tremor.brand.*` keys straight at the
`--ag-color*` CSS vars the way `themeAwareColors` already does a few lines above it. Worth
fixing in the same PR that does the retheme, since otherwise it's a second stale-brand
surface waiting for the first real Tremor chart to ship.

No other file in `web/` calls `getDesignToken()` or imports `palette.ts` directly for a
build-time snapshot.
