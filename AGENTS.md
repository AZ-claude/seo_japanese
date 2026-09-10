# AGENTS.md

## Mission

Build `seo_japanese` as a reusable Japanese prose naturalization and verification component.

The goal is not AI-detector evasion. Optimize for natural Japanese while preserving factual meaning.

## Read first

Before implementation, read in this order:

1. `docs/HANDOFF.md`
2. `docs/DESIGN.md`
3. `docs/RESEARCH_NOTES.md`
4. `docs/UPSTREAM.md`
5. `docs/CODEX_GOAL.md`

Then inspect the current upstream `blader/humanizer`, especially `README.md`, `SKILL.md`, `AGENTS.md`, `LICENSE`, and `scripts/`.

## V1 boundaries

Implement only what is required by the V1 acceptance criteria in `docs/DESIGN.md` and `docs/CODEX_GOAL.md`.

Do not add SFT, DPO, RLHF, vLLM, Langfuse, MAUVE, detector optimization, or a large annotation platform in V1.

## Hard priorities

1. Factuality / meaning preservation
2. Immutable-region preservation in file mode
3. Japanese naturalness
4. Regression coverage
5. Reusability from other repositories

A rewrite that sounds better but invents or materially changes a fact is a failure.

## Architecture rules

- Keep writer, critic, and claim checker logically separate.
- Keep provider-specific model code behind adapters.
- Keep base/abstract AI-writing patterns separate from Japanese realizations.
- Do not make personal writing samples mandatory.
- Limit rewrite to one retry in V1.
- Do not optimize against AI detector scores.
- Preserve upstream license/attribution requirements when incorporating upstream material.

## Working style

Follow YAGNI. Continue through implementation and tests without ceremonial approval stops. Stop only for genuinely destructive, irreversible, credential-sensitive, or production-sensitive actions.

When implementation is complete, update docs with actual behavior, tests, remaining risks, and deferred V2 work.
