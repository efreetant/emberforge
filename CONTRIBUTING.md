# Contributing To emberforge

Thanks for contributing. This skill is only useful when its public contract stays clear and internally consistent.

## Core Rule

If the skill documents disagree with each other, fix the contract so the skill reads as one self-contained package.

Treat these files as the contract surface:

- `SKILL.md`
- `README.md`
- `README.zh-CN.md`
- `demo.md`
- `agents/*`
- `references/*`

## Good Contributions

- tighten the skill's runtime accuracy
- fix stale or misleading contract language
- improve examples without weakening the rules
- clarify verification, recovery, or reporting behavior
- add references for new runtime artifacts or env toggles

## Changes That Need Extra Care

- anything touching completion gates
- anything touching verification order
- anything touching `feature-memory` failure records
- anything touching `progress.jsonl` event shapes
- anything that changes file paths or artifact names

## Contribution Workflow

1. Read `SKILL.md` and the relevant files under `references/`.
2. Compare the proposed change against the current runtime implementation.
3. Update the skill docs and references together; do not patch one file in isolation when the contract spans multiple files.
4. Keep paths project-relative and consistent with `PROJECT_DIR`.
5. Call out any intentional contract drift clearly in the PR description.

## PR Checklist

- the change matches the current runtime behavior
- file names and paths match the real artifacts
- no private `.skill-state.json` or `.skill-config` concepts were introduced
- verification still requires successful checks before a feature is marked passed
- recovery behavior still uses `agent/feature-memory/` and `agent/run-state.json`
- reporting behavior still writes `progress.jsonl` and `emberforge-report.html`

## Style

- prefer plain, direct language
- document the current contract, not aspirational behavior
- avoid adding host-specific installation steps unless they are verified
- keep examples short and concrete

## Testing Expectations

This skill is documentation-plus-contract. Validation is mostly contract review:

- read the skill files listed above
- confirm artifact names and field shapes
- confirm env toggles still exist in the published contract
- confirm workflow order still matches across `SKILL.md`, `agents/`, and `references/`

## Scope

This directory is intended to be portable. Keep unrelated monorepo details out unless they are required to explain the contract.
