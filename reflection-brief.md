# Reflection Brief — Evaluation and Observability Capstone

**Name: Denis Fekete**
**Date: 13.11.2026**

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | 11 Enterprise (build 26200.9711) |
| Python version | Python 3.14.4 |
| Date run | 13.11.2026 |
| Ran any system live? (which) | 01-policy-pipeline, 02-mortgage-extraction, 03-supply-chain |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 48 |
| Routing output file | routing_decisions.json (captured terminal output at pipeline-run.txt) |
| auto_approve / human_review / spot_check counts | 0 / 8 / 1 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation record. How many API calls did the system make, and why is retrying a futile case worse than escalating it?

```json
"coverage_limit_absent": {
      "count": 1,
      "policies": [
        "POL-2025-002"
      ],
      "categories": [
        "missing_source"
      ]
    },
```
> The run made 20 API calls in total, however the POL-2025-002 policy was not retried because it was missing a required field, making any retry futile. This can be observed in the routing decisions file 01-policy-pipeline/routing_decisions_p.json at line 4 showing request for the policy and immediate validation_failed status not followed (application did not attempt to retry the request and immediatly escalated).

>  Retrying a futile case would result in repeated API calls without resolving the underlying issue, which is why the system escalates instead. In my perturbation run, the system made 1 API call before escalating due to the missing required field.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's confidence alone, what would have happened?

> The picked record was first one targeting the `POL-2025-001` policy. The signal that sent it to a human was the reviewer disagreement, as indicated in the routing decisions file `routing_decisions.json` at line 17, showing `reviewer_disagreements": ["coverage_limit"]`.

> If only the models reported confidence values were trusted, an incorrect auto-approval could have been assigned dispite the coverage limit of policy being incorrect.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a single number hides?

> The `umbrella` scores high value `0.865` (captured in `01-policy-pipeline/calibration_report.txt`) which in context of brier value indicates that while the model is generally confident, while its accuracy is lagging behind that predicted confidence. 
> The `policy_type × field` slicing catches that the umbrella policies for exclusions have low accuracy in comparison to its predicted confidence, highlighting specific weaknesses that would be obscured by an aggregate metric alone which shows a value of `0.291`, which could be considered acceptable if viewed in isolation but actually masks critical errors in specific umbrela policy with score of `0.865`.
---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 |
| Document run | extract-run.txt |
| Classified type | primary_residence |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.
```json
{
  "extraction": {
    "borrower": {
      "full_name": "Marcus T. Hollingsworth",
      "coborrower_name": null,
      "ssn_last4": null,
      "date_of_birth": null,
      "email": null,
      "phone": null
    },
    "property": null,
    "loan": null,
    "income": {
      "base_monthly": 5416.67,
      "bonus_monthly": 1250.0,
      "bonus_ytd": null,
      "commission_monthly": 2140.0,
      "overtime_monthly": 385.5,
      "other_monthly": 450.0,
      "stated_monthly_total": 10892.17
    }
  },
  "validation": {
    "consistent": false,
    "discrepancies": [
      {
        "field": "total_monthly_income",
        "calculated": 9642.17,
        "stated": 10892.17,
        "delta": -1250.0
      }
    ]
  }
}
```
Tool use already forces valid JSON, yet the validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot catch.
> Valid JSON does mean that the pipeline or workflow using the data from the model (through tool use) will not break due to syntax error, but it does not guarantee that the values themselves are semantically correct and consistent. An example would be the `income` field pasted above, the tool use forces a valid JSON structure, however, the structure itself does not guarantee that the values add up to the `stated_monthly_total`. These values could be inconsistent from the input data or from the model's hallucinations, and only a validation step that checks the arithmetic consistency can catch such errors. On the other hand, the validator cannot catch syntax errors or structural issues that would prevent the JSON from being parsed correctly in the first place.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> Example of such field from above pasted code could be: `"bonus_ytd": null,`.
> A `null` value is used to differentiate between a field that is genuinely missing or unknown and one that has a concrete value, even if that value could be inferred or fabricated. This ensures that the extraction process does not introduce potentially incorrect information and loudly signals to downstream consumers that the data is absent rather than assumed. 
> Schema define in Income class in `Build a Validated, Routed Insurance Policy Extraction Pipeline\04-hitl-routing\solution\models.py` at line 59 shows that field `bonus_ytd` is allowed to be `null`, which aligns with the extraction output when the value is genuinely missing or unknown.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> One example of normalization is the final apprised monetary value, the source test states it as a `$ 410,000` string (`Build a Validated, Routed Insurance Policy Extraction Pipeline\04-hitl-routing\solution\data\policies\appraisal_informal_sqft.txt` at line `92`) where as the extracted values is represented as `"appraised_value": 410000.0` numeric value in JSON document (from the `02-mortgate-extraction\extract-run.txt` line `21`).
> Normalization at extraction time ensures that all downstream consumers of the data can rely on a consistent format without having to implement their own normalization logic. For this example, different packages might interpret the `,` and `.` for thousands and decimal separators differently, leading to potential misinterpretation of the monetary value if normalization were deferred to downstream processing (fun fact, nice example would be NASA's measurement mismatch incident where two units from different parts of the world reported values in different format which lead to a catastrophic spacecraft failure). 

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 |
| Briefing file | investigation-run.txt |
| Section the conflict landed in | on_time_delivery_rate |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values, sources, dates. Give one way a reader is better served by the preserved conflict than by a single reconciled number.

> Conflicting metric pair from the briefing: `on_time_delivery_rate`, reports `95.0` percent from the `supplier_audit` on `2026-04-10` and the other metric reports `78.0` percent from the `logistics` on `2026-04-05` (both values can be seen in `03-supply-chain/investigation-run.txt` at lines `67` and `68` respectively).

> Preserving the conflict allows the reader to understand that there is a discrepancy between different sources, which may indicate underlying issues such as data entry errors, timing differences, threshold misconfigurations or a genuine difference in performance across different reporting channels. Having multiple data points enables the reader to make a more informed judgment rather than blindly trusting a single average value or from one that was chosen because it was more plausible.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the failed source. How is "unreachable" handled differently from "nothing to report," and why does the run still finish?

```
## Incomplete
### late_shipment_count  _[missing source: timeout reading logistics]_
- missing source: timeout reading logistics
```

> Unreachable indicates that the source could not be loaded due to an error such as a timeout, network failure, or other access issue, whereas "nothing to report" means the source was successfully accessed but contained no relevant data.

> The run still finishes because the system is designed to handle missing or unreachable sources gracefully. This ensures, that even if the source is unreachable, the overall process can and atleast partial informaton can be delivered to reader (for example procurement lead). The key idea is that the system does not treat a single source failure as fatal, allowing other available data to be processed and reported, maintaining continuity and reliability of the overall evaluation.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does requiring a date stop a time difference from reading as a contradiction?

> The `defect_rate_ppm` reports a two different values with an different dates: `180.0` ppm on `2026-04-10` and `190.0` ppm on `2026-04-08`. Both values are valid despite showing different readings because the dates indicate that they refer to different points in time. Requiring a date ensures that readers can distinguish between temporal differences and actual contradictions. An example would be that the supplier packaged and sent 190 units, where as the customer received only 180 units, without the context, and showing that both values are from different dates could misleadingly appear as a conflict.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the output, don't trust the model's word* most clearly caught something a trusting design would have shipped.

> A great example would be `02-mortgage-extraction` (`Build a Resilient Mortgage Document Extraction System`) system where the `validator.py` validates the extracted mortgage data against predefined rules and cross-checks it with other sources. A results from discrepency run (`02-mortgage-extraction/discrepency-run.txt`) shows that the discrepancies that have been missed by the model, but were caught by the validator. This error shows a smaller discrepency (a monetary delta of `-1250.0`) while a bigger, more obvious one would have been caught (like in `02-mortgage-extraction/extractor-run-p.txt`) where model itself flagged the issue and used `FlaggedForReviewError`.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using something you observed.

> For this, a `01-policy-pipeline/calibration-report.txt` demonstrates that the model can be highly confident in its predictions, yet still produce incorrect outputs. This is one of the main issues with the LLMs, as they can generate responses with high certainty even when they are factually wrong or misaligned with the intended policy. The report itself shows instances where the model assigned high confidence scores to policy decisions that were later flagged as incorrect by the validation logic. This highlights that confidence alone is not a reliable indicator of correctness, and independent validation is crucial to catch errors that the model might confidently assert.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> This heavily depends on the specific workflow as each system could have different failure modes and requirements for correctness. An example workflow could be any system where the model works with numeric data or other forms, that requires precise validation against predefined rules. In such a case, the validated retry with escalation pattern would be appropriate to ensure that any discrepancies are caught and addressed. For any high impact decisions or critical data points, the system should always rely on human review to confirm the results before finalizing any actions. Additionally, instrumentation should include logging of validation failures, retry attempts, and escalation events to monitor when the workflow breaks and ensure timely intervention.