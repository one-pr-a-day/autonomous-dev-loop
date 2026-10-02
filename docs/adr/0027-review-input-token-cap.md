# ADR-0027: Cap the PR review input tokens (`review_max_input_tokens`)

- **Date:** 2026-10-02
- **Status:** Accepted

## Context

The `review` stage of the pipeline assembles its LLM request from the diff (capped at 12,000 chars by `filterDiff`), the PR body, the dependency-manifest context, change classification, automation-gate context and the tool-evidence block (ADR-0024). Only the diff was bounded. On Groq's free tier for `openai/gpt-oss-120b` the per-request TPM limit is **8,000** (ADR-0025), so a review with a long PR body, a large manifest and failing checks (up to 2,000 chars of output tail per failing check from ADR-0024) could exceed the limit and receive HTTP 413 — after which no verdict is posted and the review fails closed.

Measured on PR #161: `input_tokens_est: 6743` plus `review_max_tokens: 1024` totals ~7,767, leaving ~230 tokens of margin under 8K. A PR with failing checks adds up to 2,000 chars of output tail per failing check (~500 tokens each), making it the most likely case to exceed the budget.

`autofix_max_input_tokens` already bounds the auto-fix stage the same way (ADR-0017, amended by ADR-0025). The review stage had no equivalent.

## Decision

1. New optional key `review_max_input_tokens` in `config/models.yaml`, validated by `loadLLMConfig('review')` through the existing `<stage>_max_input_tokens` branch (absent, valid, non-positive/invalid values throw). Default `5000`, chosen so that `system (pr-review-system.md, ~1,890 tokens) + review_max_input_tokens + review_max_tokens (1,024)` ≈ 7,914 stays within the 8K TPM. A config test asserts the sum with the system prompt measured by `estimateTokens()`, not a hard-coded constant.
2. `pr_review.mjs` bounds the total user prompt at `review_max_input_tokens`. When the assembled request exceeds it, sections are trimmed in priority order:
   1. dependency manifest context,
   2. PR body,
   3. diff.
   The tool-evidence block and the verdict format instructions (the static template wrapper) are never trimmed, since ADR-0024 and the verdict parser rely on them.
3. When anything is trimmed, the prompt ends with a `### ⚠️ Review input trimmed` notice naming the trimmed sections (mirroring the existing `diffTruncated` warning), so the model knows it did not see the full PR. The `review.llm_request` observability event logs a per-section `token_estimate` breakdown (`system`, `wrapper`, `classification`, `automation_gate`, `manifests`, `body`, `diff`, `evidence`, `trim_notice`, `max_input_tokens`, `trimmed`).
4. Removing the key restores the current behavior: `loadLLMConfig('review')` returns `maxInputTokens: undefined` and no trimming or notice is applied (the full model context window is used, e.g. on the Groq Developer plan or with Anthropic).

## Alternatives Considered

- **Raise `review_max_tokens`/lower it instead** — output is the wrong lever; the request is dominated by input, and a truncated review has no verdict line and fails closed to `REQUEST_CHANGES`.
- **Tokenize accurately** — rejected by the issue constraints: no new dependencies, `estimateTokens()` (chars/4) is acceptable for a safety cap.
- **Cap only the diff** — already the case; the remaining unbounded sections are exactly the ones that push failing-check reviews over the limit.
- **Cap the `generation` stage too** — out of scope; the issue explicitly defers a generation-stage cap to a separate issue.

## Consequences

- ✅ Reviews stay within the Groq free-tier 8K TPM: with a huge PR body, a large diff and failing checks, the built request stays ≤ `review_max_input_tokens` with tool evidence intact and a trim notice present (covered by `pr_review.test.mjs`).
- ✅ Operators can tune the budget without code changes — edit `config/models.yaml` and push.
- ⚠️ When the budget binds, the model may not see the full PR body or diff; the trim notice tells it so, and the classification context still reports `diff_truncated` from the final diff length.
- ⚠️ The cap applies on the Groq provider only (`loadLLMConfig` returns the budget keys for Groq), matching the `autofix_max_input_tokens` behavior; Anthropic uses the full context window.