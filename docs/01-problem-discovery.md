# Arbiter — Problem Discovery (Day 1)

> Note: Failure points and workflow mapping are grounded in direct experience evaluating a RAG chunking-strategy change using Langfuse.

## Problem Statement

Teams shipping changes to AI-powered features have observability and evaluation tools that score outputs well, but still have to manually synthesize those scores into a ship decision — tracing regressions back to their root cause and weighing conflicting metrics (quality vs. cost vs. latency vs. segment impact) themselves. This synthesis step remains slow, inconsistent, and easy to shortcut under launch pressure, regardless of what kind of AI feature is being changed — a prompt, a model, a retrieval strategy, or a chunking approach.

## Job To Be Done

When I've made a change to an AI-powered feature and have evaluation scores in hand, I want an evidence-based recommendation on whether it's safe to ship — including why any regression happened — so I can make a confident ship/hold decision without manually digging through dashboards and reasoning it out myself under time pressure.

*Example: in a RAG pipeline, this shows up when a chunking strategy change moves eval scores in Langfuse, but someone still has to manually figure out why and whether it's safe to ship.*

## Current Workflow (without Arbiter)

- A team changes the RAG pipeline — chunking strategy, retrieval model, embedding model, or the synthesis prompt — hoping to improve quality
- They evaluate using an observability/eval tool (e.g. Langfuse) — which scores runs across quality, latency, and other metrics
- The scores tell them *what* moved, but interpreting *why* it moved and what to do about it is left to manual reasoning
- If chunking v2 improves one metric but hurts another, or helps some query types and hurts others, that synthesis is not something the eval dashboard decides — someone has to sit with the scores and make the call
- There's no automatic root-cause step — nothing checks "did the underlying document structure make this chunking strategy worse for certain doc types" without manual digging
- The final ship/hold decision is a judgment call made by eyeballing dashboards, not a structured, evidence-grounded recommendation

## Failure Points

1. Even good observability tools give you scores, not synthesis — going from "these are the numbers" to "is this safe to ship" is manual every time
2. No automatic causal tracing — when a metric regresses, nothing connects it back to the underlying change without someone manually digging in
3. Repeated re-evaluation is manual and easy to shortcut — under time pressure, this synthesis step is what gets rushed or skipped
4. Segment/query-type blind spots — aggregate scores can hide that a change helped one case and hurt another
5. No structured recommendation output — even with great eval data, there's no artifact translating scores into "ship / hold / limited rollout" with evidence attached

## Assumptions

- Arbiter is a general evaluation/decision layer, demonstrated with one worked example (RAG / chunking strategy)
- Input eval data is either real (from actual test runs) or realistically simulated — not scraped from live production
- Tool integrations (GitHub, Jira, analytics) for root-cause investigation are simulated/mocked with realistic seeded data for MVP
- Single PM/decision-maker per evaluation — not a multi-approver workflow

## Non-Goals

- Not an eval-scoring platform itself (not replacing Langfuse/Braintrust/Arize) — Arbiter consumes eval output, doesn't generate it from scratch
- Not a general-purpose "bring your own AI feature" self-serve product with full onboarding
- Not making autonomous ship/hold decisions — always a human-approved recommendation
- Not real-time continuous monitoring — per-change, on-demand evaluation
- Not multi-tenant/enterprise auth for MVP

## AI vs. Deterministic Boundaries

| Task | Deterministic or AI? | Why |
|---|---|---|
| Ingesting eval scores (quality, cost, latency, groundedness) | Deterministic | Facts/numbers — no room for interpretation |
| Computing deltas between V1 and V2 | Deterministic | Pure math — should never be hallucinated |
| Flagging regressions vs. improvements against a threshold | Deterministic | Rule-based, explainable, reproducible |
| Investigating why a regression happened (checking code changes, known issues, segment data) | AI (agent) | Requires judgment — deciding which tools to call and synthesizing relevance |
| Explaining root cause in plain language | AI | Synthesis across multiple sources into a coherent explanation |
| Detecting conflicting signals across segments | AI, working from deterministic segment scores | Detection of "this matters" is judgment; underlying numbers are facts |
| Final Ship / Hold / Limited Rollout recommendation | AI, constrained by deterministic guardrails | Hybrid — LLM reasons, but hard rules (e.g. never "Ship" below a safety threshold) can't be overridden |
| Actually shipping / merging / deploying anything | Human only | Non-negotiable trust boundary |