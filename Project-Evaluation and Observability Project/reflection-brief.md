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
| OS & version | Windows 11, 64-bit operating system, x64-based processor |
| Python version | 3.14.4 |
| Date run | 11/11/2026 |
| Ran any system live? (which) | Yes. The system 2. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 |
| Routing output file | evidence/system-1/policy_extractor_live_run.json |
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

It made 1 API calls, as you can see in the screenshot evidence/system-1/perturbation.png.

A futile case is one where the document lacks a required field. Retrying it is worse than escalating it for four reasons:

• It can't succeed. The model can only read what's in the document. Every retry gets the same input, so the same validation failure comes back.
• It wastes calls. Each retry is another API call. With  max_retries=3 , that's up to 4 calls instead of 1, with no gain.
• It risks fabrication. Repeated validator errors pressure the model to invent a value just to pass. A made-up premium that passes validation is worse than a visible failure.
• It delays the fix. Only a human can resolve it, by getting the missing information or the correct document. Escalating at once hands over the case with the field and reason attached.


**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

I've picked the POL-2025-004 from system-1/routing.json.

The signal sent it to a human was the reviewer. The independent reviewer disagreed with the extractor on deductible.

The other two signals were clean:
• Confidence: the lowest score was 0.95 (endorsements). That is above the 0.90 threshold (DEFAULT_CONFIDENCE_THRESHOLD  in routing.py), and fields_below_threshold is empty.
• Integration:  integration_failures  is empty.

If I had trusted confidence alone, the record would have been auto-approved, since every field was at or above 0.95. The deductible disagreement would have gone through unreviewed. The model reported 1.0 on deductible, so its confidence did not flag the problem. The reviewer did. Records 005 through 008 show the same pattern. Their only trigger was the reviewer disagreement.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

The callibration snippet can be found on evidence/system-1/policy_extractor_briefing.png:

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
| Document run | evidence/system-2/fixtures_documents_appraisal_informal.json |
| Classified type | Appraisal |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

Discrepancy-run output from evidence/system-2/fixtures_documents_income_sum_mismatch.json:

"discrepancies": [
  {
    "field": "total_monthly_income",
    "calculated": 9642.17,
    "stated": 10892.17,
    "delta": -1250.0
  }
]

Two different guarantees: shape vs. meaning.

• Tool use (schema) guarantees syntax and structure. The  tool_use  input must parse as JSON and match  input_schema : required keys are present, types are right, and enums are valid ( tools.py ). It checks each field on its own.
• The validator ( validator.py ) checks cross-field arithmetic. It compares  calculated_monthly_total , which is computed in Python from the income components, against  stated_monthly_total , which is extracted from the document. It flags a  Discrepancy  if they differ by more than the $1 tolerance. A JSON schema can't express "these fields must add up to that one."

So a document whose components sum to $8,000 but state $9,000 gives perfectly valid JSON. The validator is the one that catches it.

An error each can't catch:

• Schema can't catch: a bad sum. For example,  stated_monthly_total  is a valid number but doesn't match the components.
• Validator can't catch: a structural or type failure, such as a missing required field, a string where a number belongs, or an invalid enum value. It also can't catch a consistent but wrong extraction. If the model misreads a component and the stated total together so they still agree, or if both are  null  (the check is skipped), nothing flags it.


**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

The field output from evidence/system-2/fixtures_documents_income_missing_bonus.json: null.

{
  ...
  "bonus_monthly": null,
  "bonus_ytd": null,
  ...
}

Schema choice:

class Income(_BaseRecord):
    bonus_monthly: float | None = None
    bonus_ytd: float | None = None

It is null instead of an invented value, because a fabricated value looks like real data, while
null means "the document doesn't say it", which is better for code and reviewers to detect and handle than
an invented value. A made-up value can't be told apart from a real one.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

On the field "gross_living_area_sqft" from evidence/system-2/fixtures_documents_appraisal_informal.json is 2400 instead of "0.46 acres (approx. 20,037 sq ft)" from system-2/fixtures/documents/appraisal_informal_sqft.txt.

Why normalize at extraction time rather than downstream:

1. The schema enforces it. The field is typed  integer , so  "about 2,400 sq ft"  would be rejected. Normalizing at extraction is the only way to produce a schema-valid value. Downstream cleanup would require loosening the type to  string , and you'd lose the type guarantee tool use gives you.
2. The model has the context. It can tell "~3,100 SF" from "3,100 sq ft (garage excluded)" and can read an unclear number as  null . A downstream regex only sees the string and can silently mis-parse, for example grabbing the wrong number. If the model is unsure, it can emit  null  ( prompts.py:29 ).
3. One rule in one place.  NORMALIZATION_RULES  is the single source of truth, shared by all extractor prompts. Downstream, every consumer (validator, reports, underwriting) would need its own parser, and they would drift apart.
4. The validator and other consumers can trust the data. The validator does arithmetic and needs real numbers. Mixed formats would push parsing errors and "is this a number?" checks into every consumer.
5. Errors are caught early. A mistake at extraction shows up next to the source document, where it's easy to evaluate. A parsing bug downstream is far from the source and harder to trace.


---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 |
| Briefing file | evidence/system-3/supply_chain_investigate_briefing.txt |
| Section the conflict landed in | on_time_delivery_rate |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

From evidence/system-3/supply_chain_investigate_briefing.txt:

The conflicting-metric pair is on_time_delivery_rate (flagged "Contested… ⚠️ ESCALATE"):

• 95.0 percent, from  supplier_audit , as of 2026-04-10
• 78.0 percent, from  logistics , as of 2026-04-05

Why preserving the conflict serves the reader better than one reconciled number:

- A 17-point gap is too large to be noise, and it likely reflects different definitions or vantage points. The supplier's audit may measure against promised dates, while logistics may measure actual receipt at the dock. Averaging to about 86.5% would hide that. It would also give a figure that neither source reported and that looks more certain than it is.

- With both values and their sources shown, the reader can see the disagreement and judge which source fits their purpose. The escalation flag then tells them to resolve it before acting, for example before sizing safety stock or deciding on supplier risk. The briefing's own table shows only the logistics value (78.0, as of 2026-04-05). That is a single-number view, and it silently drops the 95.0 figure, which is the kind of loss the preserved conflict avoids.


**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

From the file evidence/system-3/supply_chain_investigate_briefing_failed_timeout.txt:

"
> Sources unavailable: logistics unavailable (timeout)
...
## Incomplete
### late_shipment_count  _[missing source: timeout reading logistics]_
- missing source: timeout reading logistics

### production_capacity_utilization  _[missing source: no source reported this metric]_  ⚠️ ESCALATE
- escalation: high-impact metric has no usable source
- missing source: no source reported this metric
"

"Unreachable" vs. "nothing to report":

• Unreachable (timeout): The briefing names the failed source ( logistics ) in a header line. The metric it would have supplied,  late_shipment_count , is marked  missing source: timeout reading logistics . The cause is recorded, so a reader knows the data may exist and a retry could fix it.
• Nothing to report:  production_capacity_utilization  says  no source reported this metric . Every source was reached, and none had the metric. This is a real gap, and it is escalated because it's a high-impact metric.

Neither case is turned into a zero or a guess. Each is kept as a distinct kind of absence.

Why the run still finishes: a source failure is caught and recorded as data rather than raised as an error that aborts the run. The pipeline carries on with the sources it could reach. Note what changed as a result.  on_time_delivery_rate  is now "single source only" at 95.0%. With logistics down, the 78.0% value is missing, so the earlier conflict disappears, and  Contested  shows  _none_ . The "Sources unavailable" banner warns the reader that this apparent consensus is partial, so a hard failure isn't hidden behind a clean-looking report.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

From the file evidence/system-3/supply_chain_investigate_briefing, two dated claims about Meridian Components are:

1. "Meridian Components issued a voluntary precautionary recall of a single capacitor-assembly lot in Q1 2026 over potential elevated leakage current under thermal stress." (industry_news, 2026-02-19)
2. "A 72-hour dockworker strike at the Port of Long Beach beginning March 16, 2026 stranded Meridian Components' inbound container shipments, delaying customer deliveries by an estimated five to nine days." (industry_news, 2026-03-17)

These are not an exact contradiction pair. They are different events a month apart. The same logic applies to the metric lines, such as  defect_rate_ppm  at 180.0 (as of 2026-04-10) versus 190.0 (as of 2026-04-08).

How requiring a date prevents a false contradiction:

Without dates, "quality problems (recall)" next to "shipments delayed" or "180 vs 190 ppm" could read as the supplier being inconsistent or the sources disagreeing. A required date turns each claim into a statement about a specific moment: "as of X, this was true." The reader can then see that two differing values may both be correct, because the underlying quantity changed between observations. The system can also treat a time difference as a trend rather than a conflict. It reserves the "Contested" flag for values that disagree at comparable dates, as with the 95.0 vs 78.0  on_time_delivery_rate  gap (5 days apart). That keeps real conflicts meaningful instead of burying them in false alarms.


---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

On system-2,  evidence\system-2\fixtures_documents_income_sum_mismatch.json - the "validation" block, from the  income_sum_mismatch.txt  pay stub.

The model extracted every field correctly and returned perfectly valid JSON:

• Base 5,416.67, commission 2,140.00, bonus 1,250.00, overtime 385.50, other 450.00
•  stated_monthly_total : 10,892.17

A trusting design would have shipped that record. The schema is satisfied and nothing looks wrong. But the validator recomputed the sum in Python and got 9,642.17. That is 1,250.00 less than the stated total, which is exactly the bonus amount. The stub's own "TOTAL MONTHLY EARNINGS" line counts the bonus twice.

The evidence shows:

"validation": { "consistent": false,
  "discrepancies": [{ "field": "total_monthly_income",
    "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0 }] }

A lender relying on the stated total would have overstated the borrower's income by $1,250 a month. Instead the discrepancy was flagged for human review, not silently corrected.

Neither schema enforcement nor the model's own claim of a faithful extraction could catch this. Only an independent check of the output against the arithmetic did.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

What I observed

1. The live run,  evidence\system-1\policy_extractor_live_run.json :
• It wrote 9 decisions:  auto_approve: 0 ,  human_review: 9 ,  spot_check: 0 . The pipeline auto-approved nothing on confidence alone.
• The pattern summary shows why.  POL-2025-010  has  premium_does_not_match_components  (category: consistency).  POL-2025-009  has  endorsements_absent  (category: missing_source).
• A model can report high confidence on a number and still be wrong, and the cross-field arithmetic check caught that for  POL-2025-010 .
2. The routing design ( routing.py ,  test_us04_routing.py ):
• Routing is "a pure function of (extractor confidence, reviewer agreement, integration findings)". The model's confidence is "one of three inputs, not the sole gate."
•  test_ac_04_06  encodes the rule: "All-0.99 extractor confidence cannot override a reviewer disagreement." High confidence plus reviewer disagreement still goes to  human_review .
3. Calibration ( routing.py , around lines 174–188): the system compares mean predicted confidence against observed accuracy and computes a Brier score per cell. This tests whether a 0.95 actually means 95% correct, instead of assuming it does.

Why it mattered most here: system 1 and system 3 have no confidence numbers to be fooled by. Only system 2 had a model grading its own work, so it had to be designed so that a confident wrong answer cannot auto-approve itself.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

First pattern: validated retry with escalation.

• The cheapest failures are mechanical ones: a malformed value, a missing required field, a premium that doesn't equal the sum of its components.
• A deterministic validator catches these for free. On failure, you retry once with the specific error fed back to the model. If it still fails, you escalate to a human.
• It needs no second model call in the common case, and it bounds cost and latency.

I'd add independent review with deterministic routing second. It targets the failure retry can't see, which is a confident, schema-valid, wrong value. Provenance-preserving annotation comes in when the same field has several sources, as in system-3.

What I'd instrument:

• Validator failure rate per field and per document source. A jump means a new template or a prompt regression.
• Retry success rate and escalation rate. Rising retries mean the first pass is degrading. Rising escalations mean retries can't fix it.
• Auto-approve rate. A sudden spike to near 100% (or a drop to 0, as in the live run) means a routing threshold or a model change broke something.
• Calibration: predicted confidence vs. observed accuracy, from a sampled human audit of auto-approved records, scored with Brier. This catches a model whose "0.95" quietly stops meaning 95%.
• A stratified spot-check sample of approved records, so silent errors show up even when nothing is flagged.
• Per-run traces (input hash, prompt version, model version, raw output, validator result), so any bad record can be replayed and attributed to a change.

