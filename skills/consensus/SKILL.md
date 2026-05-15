---
name: consensus
description: Verifies implementation plans and debugging hypotheses using multi-agent consensus. Use when creating, refining, or reviewing a technical plan before coding, or when forming a root-cause hypothesis or proposed fix before acting on it.
disable-model-invocation: true
---

# Multi-agent Consensus

Use this skill to validate a proposed direction before committing to it. Two supported modes:

- **Plan-mode**: validate an implementation plan before coding. Catches unclear requirements, risky assumptions, missing edge cases, architecture issues, and weak test strategy early.
- **Debug-mode**: validate a root-cause hypothesis or proposed fix before acting on it. Catches premature conclusions, alternative root causes that weren't considered, and fixes whose blast radius is wider than they look.

## Workflow

1. Validate the user's input.
   - Confirm the goal, scope, constraints, and context are clear enough to produce an actionable artifact (plan or hypothesis).
   - For debugging: confirm there's a clear symptom, repro path or relevant logs, and a definition of "fixed".
   - Ask the user for clarification before proceeding if any critical requirement is ambiguous.

2. Draft the artifact.
   - **Plan-mode**: technical approach, key files or modules, implementation steps, test strategy.
   - **Debug-mode**: suspected root cause, evidence supporting it, proposed fix, expected behavior after the fix, alternative hypotheses considered.
   - Call out meaningful edge cases and ask for feedback when product or architecture intent is unclear.

3. Launch planning reviewers in parallel.
   - Launch every available `planning-buddy-*` subagent in the same parallel batch.
   - Pass the original user request, the artifact, relevant code context (logs and repro steps for debugging), known constraints, and any open questions.
   - If project-specific reviewers exist, also launch `code-quality-architecture-reviewer` and `testing-quality-reviewer`.

4. Synthesize consensus.
   - Consolidate findings into one view.
   - Resolve conflicting feedback explicitly.
   - Challenge reviewer suggestions that do not fit the codebase or requirements.
   - Ask the user for guidance if consensus cannot be reached.

5. Present the refined artifact.
   - Use the output format below.
   - Do not implement (or apply a fix) until the user accepts the artifact or gives clear permission to proceed.

## Output Format

```markdown
## Consensus Summary

[Brief overview of specialist feedback and resolution]

## Refined Plan / Hypothesis

[Updated artifact with incorporated feedback. For debugging, include the verification step that will confirm or falsify the proposed root cause.]

## Risks & Mitigations

[Key risks identified and how to address them. For debugging: blast radius of the proposed fix and rollback plan.]

## Assumptions Validated/Revised

[What was challenged and the outcome]
```
