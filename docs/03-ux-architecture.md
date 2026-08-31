\# Arbiter — UX \& Architecture (Day 3)



\## Core Screens



\### 0. Evaluation History (Home)

\- Table of past evaluations: feature name, V1/V2 labels, timestamp, recommendation verdict (color-coded badge), approval status (Pending / Approved / Rejected)

\- Clicking a row opens that evaluation's Results Dashboard

\- "New Evaluation" button leads to Setup screen

\- Seeded with 3 demo scenarios: a clean ship, a clear hold with confirmed cause, and a conflicting/unresolved case



\### 1. Evaluation Setup

\- Feature name / label

\- V1 label + V2 label (free text)

\- Eval data input — structured upload (CSV/JSON) of per-query scores for both versions

\- "Run Evaluation" button, kicks off the agent loop



\### 2. Results Dashboard

\- Header: feature name, V1 vs. V2 labels, timestamp

\- Recommendation banner at top — Ship / Hold / Limited Rollout, color-coded, confidence level shown alongside

\- Metric cards (Quality, Groundedness, Latency, Cost): score, delta, delta %, "significant" badge only if the significance test confirms it

\- "View Evidence" link per flagged regression



\### 3. Evidence / Root-Cause Detail

\- Root cause summary sentence at top

\- Evidence list: source, finding, confidence tag per item

\- Segment breakdown table (e.g. table-heavy docs vs. short-form Q\&A)

\- Chunk diff samples — example queries showing V1 vs. V2 retrieved chunks side by side with similarity score

\- If unresolved: replaces "cause" framing with "No confirmed cause — manual review recommended"



\### 4. Approval

\- Recommendation restated

\- Approve / Reject buttons — Reject requires a reason, Approve allows an optional one

\- Becomes read-only once decided, showing who decided and when



\## UX States



\- \*\*Loading:\*\* step-by-step progress ("Computing metrics..." → "Checking for regressions..." → "Investigating with GitHub tool..." → "Analyzing chunk differences..." → "Synthesizing recommendation...") — not a generic spinner

\- \*\*Empty (History, first use):\*\* "No evaluations yet" + "Run your first evaluation" CTA

\- \*\*Error (tool call fails):\*\* deterministic metrics still shown; evidence section shows "GitHub tool unavailable — root cause investigation incomplete"; confidence automatically capped

\- \*\*Unresolved root cause:\*\* neutral amber/gray "Inconclusive" badge, explicit message, manual review note

\- \*\*Conflicting evidence:\*\* amber/neutral, both findings shown side by side, explicitly labeled "Conflicting signals"



\## Agent Loop



```

1\. RECEIVE eval data (V1 + V2 per-query scores)



2\. COMPUTE deterministic layer (pure Python):

&#x20;  - Aggregate scores per metric (quality, groundedness, latency, cost)

&#x20;  - Calculate delta, delta %, run significance test per metric

&#x20;  - Flag metrics where regression\_flag = true (significant AND negative)

&#x20;  - Run chunk-diff analysis (embedding similarity)

&#x20;  - Compute segment breakdown



3\. IF no metrics flagged:

&#x20;  -> Recommendation = "Ship" (still requires human approval), confidence = high



4\. IF one or more metrics flagged:

&#x20;  FOR EACH flagged regression:

&#x20;    a. Agent decides which tool(s) are relevant

&#x20;    b. Call simulated GitHub tool

&#x20;    c. Call simulated segment analytics tool

&#x20;    d. Cross-reference chunk-diff data against both tool findings



5\. SYNTHESIZE root cause (LLM, constrained by honesty policy):

&#x20;  - Both tools agree -> high confidence, clear summary

&#x20;  - Only one tool returns relevant evidence -> medium confidence, state what's unconfirmed

&#x20;  - Tools disagree -> surface both findings, flag contradiction, confidence = low, likely unresolved

&#x20;  - Neither tool returns anything -> unresolved = true, honest "no clear cause found"



6\. APPLY deterministic guardrails (code, not LLM):

&#x20;  - IF unresolved = true on any flagged regression -> verdict can never be "Ship"

&#x20;  - IF regression crosses a hard safety threshold -> verdict forced to "Hold"



7\. LLM produces final verdict + reasoning, constrained by step 6 guardrails



8\. WRITE result to schema, human\_decision.status = "pending"



9\. WAIT — nothing executes further until PM approves/rejects

```



\## Tool Contracts



\### Simulated GitHub-like Tool



\*\*Input:\*\*

```json

{

&#x20; "feature\_name": "RAG Search Assistant",

&#x20; "v1\_label": "chunking-strategy-v1",

&#x20; "v2\_label": "chunking-strategy-v2",

&#x20; "time\_range": { "from": "v1\_deploy\_date", "to": "v2\_deploy\_date" }

}

```



\*\*Output:\*\*

```json

{

&#x20; "changes\_found": true,

&#x20; "changes": \[

&#x20;   {

&#x20;     "pr\_id": "PR-142",

&#x20;     "title": "Reduce chunk size for faster retrieval",

&#x20;     "description": "Changed max\_chunk\_size from 512 to 256 tokens",

&#x20;     "files\_changed": \["chunking\_config.py"],

&#x20;     "merged\_at": "2024-01-15",

&#x20;     "relevance\_note": "Directly modifies chunking strategy"

&#x20;   }

&#x20; ]

}

```

If nothing relevant: `"changes\_found": false, "changes": \[]`



\### Simulated Segment/Document Analytics Tool



\*\*Input:\*\*

```json

{

&#x20; "feature\_name": "RAG Search Assistant",

&#x20; "v1\_label": "chunking-strategy-v1",

&#x20; "v2\_label": "chunking-strategy-v2",

&#x20; "metric": "quality"

}

```



\*\*Output:\*\*

```json

{

&#x20; "segments\_analyzed": true,

&#x20; "segments": \[

&#x20;   { "segment": "table-heavy documents", "query\_count": 18, "v1\_avg\_score": 0.85, "v2\_avg\_score": 0.63, "delta\_pct": -25.9 },

&#x20;   { "segment": "short-form Q\&A", "query\_count": 22, "v1\_avg\_score": 0.80, "v2\_avg\_score": 0.82, "delta\_pct": 2.5 },

&#x20;   { "segment": "long-form narrative", "query\_count": 10, "v1\_avg\_score": 0.83, "v2\_avg\_score": 0.79, "delta\_pct": -4.8 }

&#x20; ],

&#x20; "clearest\_pattern": "table-heavy documents"

}

```

`clearest\_pattern` is deterministic (largest negative delta), computed in code, not by the LLM.



\## Human Approval Boundaries



\- Every evaluation lands in `"pending"` state — no exceptions, even for high-confidence "Ship" recommendations

\- Approval screen is the only place a decision can be recorded — other screens are read-only

\- "Pending Your Review" shown persistently on the History list badge

\- Rejecting requires a reason; approving allows an optional one

\- Once decided, locked — no editing after the fact; changed circumstances mean a new evaluation, not an edit to history



