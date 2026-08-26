\# Arbiter — PRD v1 (Day 2)



\## Users



\- \*\*Primary:\*\* AI Product Manager evaluating whether a change to an AI-powered feature is safe to ship

\- \*\*Secondary:\*\* Engineer/ML engineer who made the change; support/success teams affected by quality shifts; leadership tracking cost impact



\## Goals



1\. Given eval scores for V1 and V2 of an AI feature, produce a clear Ship / Hold / Limited Rollout recommendation

2\. Every recommendation is backed by visible evidence — no unexplained verdicts

3\. When a regression exists, attempt to trace a plausible root cause using connected tools (simulated GitHub/segment analytics)

4\. Never auto-ship anything — human approval is always required for any consequential action

5\. Demonstrated end-to-end using one real worked example — RAG chunking-strategy comparison, grounded in direct experience



\## Use Cases



1\. PM enters eval scores for V1 vs. V2 of a RAG pipeline → Arbiter shows deltas per metric (quality, groundedness, latency, cost)

2\. A metric has clearly regressed → Arbiter investigates using its tools and proposes a plausible cause with evidence

3\. Metrics conflict (quality up, cost up a lot) → Arbiter still produces a single recommendation, explicitly stating the tradeoff weighed

4\. No clear cause can be found → Arbiter says so honestly rather than fabricating a reason

5\. Tools disagree with each other → Arbiter surfaces the contradiction rather than silently picking a side

6\. PM reviews the recommendation and evidence, manually marks it Approved / Rejected — logged



\## Functional Requirements



\- Accept eval data for two versions (V1/V2) of an AI feature as input

\- Compute deterministic deltas across defined metrics, with statistical significance testing (not just raw deltas)

\- Flag which metrics crossed a regression threshold

\- Perform semantic diff of retrieved chunks between V1/V2 (embedding similarity) as hard evidence for RAG-specific claims

\- Agent investigates flagged regressions using simulated GitHub-like tool and simulated segment/document analytics

\- Agent produces a natural-language root-cause explanation, tied to specific, cited evidence

\- System outputs a structured recommendation (Ship / Hold / Limited Rollout) with confidence level

\- PM can view underlying evidence for any claim, not just the summary

\- PM can approve/reject the recommendation — logged, nothing auto-executes



\## Success Metrics



\- Correctly flags known regressions in seeded test scenarios (precision/recall against golden cases)

\- Root-cause explanations judged "plausible and evidence-grounded" against golden scenarios

\- Recommendation matches expert judgment on seeded scenarios with a known "correct" answer



\## Risks



\- Agent might produce confident-sounding but wrong root-cause explanations (mitigated by honesty policy below)

\- Scope creep across too many metrics/tools before core loop works (mitigated by MVP scope cut below)

\- Simulated data feeling too clean to be convincing in a demo — deliberately inject ambiguity into seeded scenarios



\## MVP Scope



\*\*Metrics (4):\*\* Quality, Groundedness, Latency, Cost — each with deterministic delta + statistical significance testing



\*\*Tools (2):\*\* Simulated GitHub-like tool (code/config changes), simulated segment/document analytics (which query or doc types got worse)



\*\*Differentiator:\*\* Semantic diff of retrieved chunks between V1/V2, giving the agent hard evidence rather than pure LLM inference



\*\*Output:\*\* Ship / Hold / Limited Rollout recommendation, evidence trail, confidence level, manual approve/reject logging



\*\*Deferred to V1.1/V2:\*\* Jira and Slack-like tools, multi-approver workflows, continuous/scheduled re-evaluation, real (non-simulated) integrations, confidence calibration validation, deterministic conflicting-signal rules



\## Output Schema



```json

{

&#x20; "evaluation\_id": "eval\_2024\_001",

&#x20; "feature\_name": "RAG Search Assistant",

&#x20; "version\_comparison": {

&#x20;   "v1\_label": "chunking-strategy-v1",

&#x20;   "v2\_label": "chunking-strategy-v2"

&#x20; },

&#x20; "metrics": \[

&#x20;   {

&#x20;     "name": "quality",

&#x20;     "v1\_score": 0.82,

&#x20;     "v2\_score": 0.75,

&#x20;     "delta": -0.07,

&#x20;     "delta\_pct": -8.5,

&#x20;     "significant": true,

&#x20;     "significance\_method": "t-test",

&#x20;     "p\_value": 0.02,

&#x20;     "regression\_flag": true

&#x20;   }

&#x20; ],

&#x20; "chunk\_diff\_analysis": {

&#x20;   "queries\_analyzed": 50,

&#x20;   "queries\_with\_different\_chunks": 15,

&#x20;   "pct\_changed": 30.0,

&#x20;   "sample\_cases": \[

&#x20;     {

&#x20;       "query": "What is the refund policy for enterprise plans?",

&#x20;       "v1\_retrieved\_chunk\_ids": \["doc\_12\_chunk\_4"],

&#x20;       "v2\_retrieved\_chunk\_ids": \["doc\_12\_chunk\_7", "doc\_12\_chunk\_8"],

&#x20;       "similarity\_score": 0.41

&#x20;     }

&#x20;   ]

&#x20; },

&#x20; "segment\_breakdown": \[

&#x20;   { "segment": "table-heavy documents", "quality\_delta\_pct": -22.0 },

&#x20;   { "segment": "short-form Q\&A", "quality\_delta\_pct": 2.0 }

&#x20; ],

&#x20; "root\_cause": {

&#x20;   "summary": "Groundedness and quality regressed primarily on table-heavy documents. Chunk diff shows 30% of queries retrieved different, often partial, chunks in V2.",

&#x20;   "evidence": \[

&#x20;     {

&#x20;       "source": "simulated\_github",

&#x20;       "finding": "Chunking strategy change in PR #142 reduced max chunk size from 512 to 256 tokens, likely splitting tables mid-row.",

&#x20;       "confidence": "high"

&#x20;     }

&#x20;   ],

&#x20;   "unresolved": false

&#x20; },

&#x20; "recommendation": {

&#x20;   "verdict": "Hold",

&#x20;   "confidence": "high",

&#x20;   "reasoning": "Significant regression in groundedness and quality, concentrated in table-heavy documents, with a plausible root cause identified and evidenced."

&#x20; },

&#x20; "human\_decision": {

&#x20;   "status": "pending",

&#x20;   "reviewer": null,

&#x20;   "decided\_at": null,

&#x20;   "override\_reason": null

&#x20; }

}

```



\## Hallucination / Unsupported-Claim Policy



1\. \*\*No evidence, no claim.\*\* If no tool call returns anything relevant to a flagged regression, set `root\_cause.unresolved: true` and say so explicitly rather than guessing.

2\. \*\*Confidence tied to evidence quantity/quality.\*\* "High" confidence requires agreement across at least 2 independent evidence sources.

3\. \*\*Never recommend "Ship" when `unresolved: true`\*\* on a flagged regression — hard, code-level guardrail, not left to LLM judgment.

4\. \*\*Distinguish correlation from causation in language\*\* — "consistent with," not "caused by," unless evidence is very direct.

5\. \*\*Every evidence item must cite a real, traceable source\*\* — no unattributed claims.

6\. \*\*Partial evidence — state what's missing, not just what was found.\*\* Confidence capped at "medium" when only one tool source returns relevant evidence.

7\. \*\*Conflicting tool results — surface the conflict, don't silently pick a side.\*\* When tools imply different explanations, report both and flag the contradiction. This forces `unresolved: true` or caps confidence at "low," since real contradiction means the answer is genuinely unclear.

