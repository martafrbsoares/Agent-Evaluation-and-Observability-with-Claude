# Reflection Brief — Evaluation and Observability Capstone

**Name:** Marta Soares
**Date:** 11/11/2026

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version |  |
| Python version | 3.14.4 |
| Date run | 11/11/2026 |
| Ran any system live? (which) | Yes. The system 2. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 |
| Routing output file | evidence/system-2/policy_extractor_live_run.json |
| auto_approve / human_review / spot_check counts | 0 / 9 / 0 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

Escalation record:

{
  "category": "missing_source",
  "detected_pattern": "premium_amount_absent",
  "field": "premium_amount",
  "kind": "escalation",
  "policy_id": "POL-2025-001",
  "reason": "Field 'premium_amount' returned null \u2014 the source document does not contain this information. Retry is futile; escalate to human review."
}

It made 1 API calls, as you can see in the screenshot evidence/system-2/perturbation.png.

A futile case is one where the document lacks a required field. Retrying it is worse than escalating it for four reasons.

• It can't succeed. The model can only read what's in the document. Every retry gets the same input, so the same validation failure comes back.
• It wastes calls. Each retry is another API call. With  max_retries=3 , that's up to 4 calls instead of 1, with no gain.
• It risks fabrication. Repeated validator errors pressure the model to invent a value just to pass. A made-up premium that passes validation is worse than a visible failure.
• It delays the fix. Only a human can resolve it, by getting the missing information or the correct document. Escalating at once hands over the case with the field and reason attached.


**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

I've picked the POL-2025-004 from system-2/routing.json.

The signal sent it to a human was the reviewer. The independent reviewer disagreed with the extractor on deductible.

The other two signals were clean:
• Confidence: the lowest score was 0.95 (endorsements). That is above the 0.90 threshold (DEFAULT_CONFIDENCE_THRESHOLD  in routing.py), and fields_below_threshold is empty.
• Integration:  integration_failures  is empty.

If I had trusted confidence alone, the record would have been auto-approved, since every field was at or above 0.95. The deductible disagreement would have gone through unreviewed. The model reported 1.0 on deductible, so its confidence did not flag the problem. The reviewer did. Records 005 through 008 show the same pattern. Their only trigger was the reviewer disagreement.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

The callibration snippet can be found on evidence/system-2/policy_extractor_briefing.png:

auto      premium_amount  n=3 conf=0.95 acc=1.00 brier=0.003
home      deductible      n=1 conf=0.90 acc=1.00 brier=0.010
umbrella  exclusions      n=2 conf=0.93 acc=0.00 brier=0.865
OVERALL brier=0.291

Cell whose accuracy lags its confidence:  umbrella × exclusions . It had n=2, mean confidence 0.93, observed accuracy 0.00, and a Brier score of 0.865.

Overall figure: Brier = 0.291.

Slicing by `policy_type × field` catches:

• The overall 0.291 looks like moderate error spread evenly. In fact, two cells are nearly perfect (Brier 0.003 and 0.010). One cell is badly miscalibrated, claiming 93% confidence while being wrong every time.
• A single number hides where the problem is. The slice names the exact  policy_type × field  combination to fix or send to human review.
• A policy-wide threshold would keep trusting 0.93 on umbrella exclusions.

The snippet uses hand-made labels: six samples, and only two in the failing cell. This shows how the report works, but it isn't a measured error rate for your system.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 |
| Document run |  |
| Classified type |  |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

Discrepancy-run output

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> 

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> 

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | |
| Briefing file | |
| Section the conflict landed in | |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> 

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> 

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> 

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> 

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> 

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> 
