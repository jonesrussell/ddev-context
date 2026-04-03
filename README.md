# Codified Context for DDEV

## The Problem

AI agents working on DDEV can read CLAUDE.md and AGENTS.md for workflow guidance (how to build, test, commit), but have no structured knowledge of the codebase's architectural boundaries: which packages own what, what the layer rules are, what invariants must hold, and what mistakes are common in each subsystem.

Without this, agents explore by reading code until they build enough context to act. This is slow, token-expensive, and error-prone: they may miss layer boundaries or modify the wrong package.

## The Hypothesis

Machine-readable subsystem specs ("codified context") give agents architectural grounding before they touch code. Each spec covers one subsystem with: file map, key types and interfaces, architecture notes, common mistakes, and testing patterns.

The claim is that agents with these specs make better architectural decisions, particularly around layer boundaries and method selection. This claim is testable (see [A/B Test](#ab-test) below).

## What's Here

Three experimental specs covering DDEV's core subsystems:

| Spec | Subsystem | Packages |
|---|---|---|
| [config.md](config.md) | Configuration system | `pkg/globalconfig/`, `pkg/config/` |
| [ddevapp.md](ddevapp.md) | Project lifecycle | `pkg/ddevapp/` |
| [dockerutil.md](dockerutil.md) | Docker operations | `pkg/dockerutil/`, `pkg/docker/` |

Plus routing skills in `skills/` that load the right spec based on which files an agent is modifying.

## How It's Used

An orchestration table in CLAUDE.md maps file patterns to specs:

| File pattern | Spec |
|---|---|
| `pkg/ddevapp/*` | ddevapp.md |
| `pkg/dockerutil/*` | dockerutil.md |
| `pkg/config/*`, `pkg/globalconfig/*` | config.md |

When an agent touches files in `pkg/ddevapp/`, the table points it to ddevapp.md. The agent reads the spec before making changes, getting architectural context without exploring the source tree.

## Status

**Experimental.** No validated metrics yet on whether this improves agent output.

An A/B test framework exists for measuring the difference. It uses a synthetic task that crosses all four architectural layers, with automated scoring on 7 criteria (correct file placement, layer compliance, method selection, test quality). See the experiment README in the jonesrussell/ddev fork under `docs/experiments/codified-context-ab/` for details and instructions.

## A/B Test

The experiment framework lives in the jonesrussell/ddev fork at `docs/experiments/codified-context-ab/`. It creates two git worktrees from the same commit: one with codified context specs and one without. Both receive the same synthetic task, and results are scored automatically on 7 objective criteria.

Phase 2 extends this to a model matrix (Opus, Sonnet, Haiku) to measure how architectural knowledge benefits vary by model capability.

## Research Basis

This approach is informed by research on providing structured architectural context to LLM-based coding agents. The core idea: workflow instructions (how to build and test) are necessary but not sufficient. Agents also need architectural instructions (where things belong and why) to make good decisions in unfamiliar codebases.

## Contributing

This is an experiment, not a recommendation. Ways to help:

- **Try it.** Clone this repo into a DDEV checkout, run the A/B test, share results.
- **Poke holes.** Are the specs accurate? Do they cover the right things? What's missing?
- **Suggest alternatives.** There may be better ways to give agents architectural knowledge.
- **Open an issue** on this repo or discuss in the DDEV Discord.
