\# Arbiter — Evaluation Framework



\## Metric Definitions



\*\*Quality\*\* (0.0 – 1.0 scale, LLM-judged)

How well the final answer addresses the query — correctness, completeness, clarity.

\- 1.0 = fully correct, complete, well-formed answer

\- 0.5 = partially correct or missing key details, but not wrong

\- 0.0 = wrong, irrelevant, or nonsensical answer



\*\*Groundedness\*\* (0.0 – 1.0 scale, LLM-judged)

How well the answer is actually supported by the retrieved chunks — separate from "quality," since an answer can sound great but not be backed by what was retrieved. This is the core hallucination-risk signal.

\- 1.0 = every claim in the answer is directly traceable to retrieved content

\- 0.5 = mostly grounded, but includes some unsupported or extrapolated claims

\- 0.0 = answer is largely unsupported or contradicts retrieved content (hallucinated)



\*\*Latency\*\* (raw milliseconds, deterministic, lower is better)

Time from query to final answer. Compared directly between V1 and V2 as a delta — no fixed scale.



\*\*Cost\*\* (USD per query, deterministic, lower is better)

Computed from token usage x model pricing. Compared directly as a delta — no fixed scale.



\*Why Quality and Groundedness are separate metrics:\* a RAG answer can be fluent and well-written (high quality) while being unsupported by the actual retrieved documents (low groundedness). That gap is exactly the kind of failure mode a chunking-strategy change can cause, which is why both metrics are tracked independently rather than blended into one "goodness" score.



\## Regression Thresholds



| Metric | Regression threshold | Reasoning |

|---|---|---|

| Quality | Drop of >5% AND statistically significant (p < 0.05) | Small dips are noise; a real, significant drop matters even if modest |

| Groundedness | Drop of >10% AND statistically significant | Slightly higher bar than quality due to natural query-to-query variance; this is the hallucination-risk signal |

| Latency | Increase of >20% | No significance test needed — directly measured, not sampled/judged |

| Cost | Increase of >15% | Cost compounds at scale even at moderate increases |



\*\*Hard safety threshold (deterministic override, cannot be bypassed by the LLM):\*\*

\- Groundedness drop >25% -> automatic "Hold" verdict, regardless of LLM reasoning. Groundedness is the sole hard-override metric for MVP — keeping overrides to one metric keeps the "AI reasons within guardrails" story clean and easy to defend.



\## Golden Dataset — Structure



Each test case:

```json

{

&#x20; "id": "tc\_001",

&#x20; "query": "What is the refund policy for enterprise plans?",

&#x20; "category": "table-heavy documents",

&#x20; "expected\_answer\_facts": \[

&#x20;   "Enterprise refunds are processed within 30 days",

&#x20;   "Requires written request to account manager"

&#x20; ],

&#x20; "expected\_source\_docs": \["doc\_12\_refund\_policy"],

&#x20; "difficulty": "medium",

&#x20; "notes": "Answer lives in a table with plan-tier-specific refund windows"

}

```



\*\*Dataset mix (50 cases total):\*\*



| Category | Count | Purpose |

|---|---|---|

| Straightforward Q\&A (clear single-source answer) | 15 | Baseline — a regression here is a serious signal |

| Table-heavy documents | 12 | Core differentiator scenario — where chunking changes are most likely to break things |

| Long-form/narrative documents | 8 | Tests whether chunking affects context spanning multiple paragraphs |

| Ambiguous queries (multiple valid interpretations) | 6 | Tests groundedness — does the answer honestly reflect uncertainty, or overclaim |

| No-good-answer-exists queries | 5 | Tests hallucination risk directly — ideal answer is "I don't have information on this" |

| Conflicting-source queries (two docs partially disagree) | 4 | Tests whether the system picks one confidently or acknowledges the conflict |



\*\*Build approach:\*\* pick or write 3-5 source documents first (at least 2-3 with actual tables), then generate queries against them so `expected\_source\_docs` and `expected\_answer\_facts` are grounded in real, controlled content.



\## LLM Judge Rubric



\*\*Quality:\*\*

```

Score the answer's quality on a 0.0-1.0 scale using these anchors:



1.0 - Fully correct, complete, directly addresses the query, well-formed

0.75 - Correct and addresses the query, but missing a minor supporting detail

0.5 - Partially correct OR complete but with a notable gap/ambiguity

0.25 - Mostly incorrect, or technically relevant but fails to actually answer the query

0.0 - Wrong, irrelevant, or nonsensical



Compare the answer against the expected\_answer\_facts for this test case.

Do not penalize different phrasing - only penalize missing or incorrect facts.

```



\*\*Groundedness:\*\*

```

Score how well the answer is supported by the retrieved chunks, 0.0-1.0:



1.0 - Every claim in the answer is directly traceable to the retrieved chunks

0.75 - Mostly grounded, one minor claim is a reasonable inference but not explicit

0.5 - Mix of grounded and unsupported claims

0.25 - Mostly unsupported - the answer goes well beyond what's in the retrieved chunks

0.0 - Answer contradicts the retrieved chunks, or retrieved chunks don't support it at all



Compare each sentence in the answer against the provided retrieved\_chunks.

Flag specifically which claims (if any) are unsupported.

```



\*\*Judge Output Schema:\*\*

```json

{

&#x20; "test\_case\_id": "tc\_001",

&#x20; "metric": "groundedness",

&#x20; "score": 0.75,

&#x20; "justification": "The refund window and process are directly stated in the retrieved chunk. The claim about 'requires written request' is a reasonable inference from the retrieved content but not explicitly stated.",

&#x20; "unsupported\_claims": \[

&#x20;   "Requires written request to account manager"

&#x20; ]

}

```



\*\*Design notes:\*\*

\- `unsupported\_claims` is a required field, even when empty — this gives root-cause investigation a concrete evidence trail (specific flagged claims), not just a bare score.

\- The rubric explicitly separates "different phrasing" from "different facts" to prevent the judge from becoming a style checker instead of a correctness checker.

\- 5-point anchor scale (0.0 / 0.25 / 0.5 / 0.75 / 1.0) used instead of a 3-point scale for finer-grained scoring resolution.



