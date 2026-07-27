# Stale LLMOps positioning outside docs/ (sweep 2026-07-23)

Agenta now positions as an agent-building platform (competitors: Gumloop, n8n,
ActivePieces). This sweep lists every surface outside the Docusaurus docs tree that
still carries the old LLMOps / prompt-management / observability positioning.
Mahmoud is fixing docs/ himself; this file covers everything else.

Already clean (verified, no action): all root READMEs (en/cn/ja/kr), readme.new.md,
sdks/python README + pyproject, website homepage meta + JSON-LD, Base/Site layouts,
auth SideBanner, all AGENTS.md files, .github templates, CONTRIBUTING.md, gtm.txt.

## Live marketing site (website/, highest impact first)

| File | Line | Stale copy |
|---|---|---|
| `website/src/components/InlineCta.astro` | 13 | "Agenta is the open-source LLMOps platform: prompt management, evals, and LLM observability all in one place." (reused block, appears on every page that embeds it) |
| `website/src/pages/pricing.astro` | 36 | meta description: "...prompt management, evaluation, and observability built in." |
| `website/src/pages/blog/index.astro` | 43, 64 | "...prompt management, evaluation, and observability for LLM apps." (meta + on-page heading) |
| `website/src/pages/authors/index.astro` | 19 | "...writing on the Agenta blog about prompt management, evaluation, and LLM observability." |

## Legacy blog archive (website/src/content/posts/)

~17 of 37 posts are built around the old category or old competitor set
(Langfuse, LangSmith, Phoenix, Helicone, Humanloop comparisons; prompt-management
guides). Decide per post: rewrite, redirect, or keep as dated archive. Worst
offenders: `top-open-source-prompt-management-platforms.mdx`,
`top-llm-observability-platforms.mdx`, `top-llm-gateways.mdx`,
`humanloop-sunsetting-migration-and-alternative.mdx`,
`the-definitive-guide-to-prompt-management-systems.mdx`,
`open-source-llm-observability.mdx`, `building-the-data-flywheel-*.mdx`
(frontmatter says "LLMOps platform").

## In-product copy (web/)

| File | Line | Stale copy |
|---|---|---|
| `web/oss/src/lib/onboarding/widget/config.ts` | 6-107 | Onboarding checklist categories: "Iterate on prompts", "Prompt management", "Run evaluation", "Tracing" (old workflow framing) |
| `web/oss/src/components/pages/evaluations/onlineEvaluation/EmptyStateOnlineEvaluation/EmptyStateOnlineEvaluation.tsx` | 12 | "Monitor and evaluate your LLM outputs in real-time..." |

## Package metadata

| File | Line | Stale copy |
|---|---|---|
| `web/_reference/agenta-sdk/package.json` | 6 | "core API client for prompt management, evaluation, and observability" (verify whether this `_reference/` copy is published anywhere before editing) |
| `web/_reference/agenta-sdk-tracing/`, `-mastra/`, `-ai/` package.json | 6/6/4 | tracing/prompt-centric descriptions |

## Draft content (Agenta landing page pivot/ at repo root)

`content/site.json` (lines 26, 30, 68, 74) and `content/pricing.json` (109, 120,
142) still carry the LLMOps hero/nav/footer copy; `handoff/landing-agents-copy.md`
line 140 has the old footer product list. Line 74 of site.json is the exact string
that leaked into the live `InlineCta.astro`.

## Internal references (low priority, arguably fine)

`.agents/analytics-api/ANALYTICS_API_COMPARISON.md` (Langfuse API comparison, an
engineering reference), `.agents/sso-login-flow/*` (Langfuse as auth precedent),
`.agents/skills/design-interfaces/SKILL.md:78` (langfuse as anti-pattern example),
`.agents/skills/write-issue/SKILL.md:91` ("prompt management" as a label name).
