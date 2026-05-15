---
name: consensus
description: Verify a proposed technical plan or debugging hypothesis via multi-agent consensus. Launches all planning-buddy-* subagents in parallel and synthesizes a refined output with risks and challenged assumptions before implementation or before acting on a fix.
---

# Multi-Agent Consensus

Use this workflow whenever you are about to commit to a non-trivial direction — either:

- **Plan-mode**: a technical implementation plan (architecture, steps, approach), or
- **Debug-mode**: a root-cause hypothesis or proposed fix for a tricky bug.

In both modes the structure is the same: draft an artifact, get parallel critiques from independently-modeled reviewers, and synthesize a single refined output. Coordinated specialist review beats single-model confidence.

## Step 0: Validate User Input

Before drafting, assess whether the user's prompt contains enough information for a complete, actionable artifact:

- **What's the goal?** For plans: what outcome are we shipping? For debugging: what symptom are we explaining?
- **What's the scope?** Files, services, features, suspected components.
- **What are the constraints?** Tech stack, performance, backward compatibility, timeline, blast radius of a fix.
- **What's the context?** Enough codebase, logs, repro steps, or domain context to make informed decisions?

If any of the above are ambiguous or missing, **ask the user for clarification before proceeding**. Do not guess at critical requirements.

## Step 1: Draft the Artifact

Identify areas/edge-cases the user didn't think about, ask for feedback on those.

Then produce one of:

- **Plan-mode**: a clear technical plan (steps, architecture, approach).
- **Debug-mode**: a structured hypothesis (suspected root cause, evidence supporting it, proposed fix, expected behavior after the fix, alternative hypotheses considered).


## Step 2: Launch Specialist Subagents in Parallel

Run these subagents **in parallel** to gather perspectives from different models. Pass each the full artifact (plan or hypothesis), relevant context (diff, files, requirements, logs, repro steps), and the user's original prompt so they can independently evaluate the approach.

Launch **all** available `planning-buddy-*` subagents simultaneously — each uses a different model to provide cross-model consensus. Do not hardcode specific names; discover all subagent types matching `planning-buddy-*` and launch every one of them.

Additionally, if project-specific reviewers exist, launch them in the same parallel batch:

- **code-quality-architecture-reviewer** (if available) — Architecture fit, code organization, project conventions
- **testing-quality-reviewer** (if available) — Test strategy, coverage gaps, edge-case testing

The `planning-buddy-*` agents cover architecture, code, testing, and edge-cases in their evaluation framework, so they are sufficient when project-specific reviewers are not available.

## Step 3: Synthesize Consensus

After all subagents return:

1. Consolidate findings into a single consensus view
2. Resolve conflicts (e.g., one says "overly complex," another says "adequate"—explain the resolution)
3. You can challenge proposals returned by `planning-buddy-*` agents, return the challenge response back and come to consensus
4. If in doubt, ask the user
5. Produce a **refined artifact** (plan or hypothesis) that incorporates:
   - Critical fixes and risks from the `planning-buddy-*` agents
   - Architecture and testing recommendations
   - For debugging: alternative root-cause hypotheses to rule out
   - Assumptions that were challenged and how they were addressed

## Output Format

```
## Consensus Summary
[Brief overview of specialist feedback and resolution]

## Refined Plan / Hypothesis
[Updated artifact with incorporated feedback. For debugging, include the verification step that will confirm or falsify the proposed root cause.]

## Risks & Mitigations
[Key risks identified and how to address them. For debugging: blast radius of the proposed fix and rollback plan.]

## Assumptions Validated/Revised
[What was challenged and the outcome]
```

## Notes

- Launch **all** `planning-buddy-*` subagents in **parallel** — cross-model consensus is the goal
- If a `planning-buddy-*` agent requests more context, gather it and re-run before implementation or before acting on a fix
- If consensus cannot be reached, ask the user for guidance
