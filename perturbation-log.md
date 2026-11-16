# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** `POL-2025-002.txt` - removed monetary values from the liability limits.
- **Command I ran:** `policy-extractor pipeline data/policies/  --spot-check-pct 0.2 --seed 42`
- **What I predicted:** The system will fail to extract the liability limits and trigger escalation due to missing values.
- **What actually happened (paste the key output line):** 
```
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
- **How this differs from the unperturbed run:** The [routing_decisions_p.json](01-policy-pipeline/routing_decisions_p.json) shows that the policy `POL-2025-002` is missing from the routing decisions, indicating it was escalated due to the missing liability limits (can be compared with unperturbed run [routing_decisions.json](01-policy-pipeline/routing_decisions.json)). Additionally, the terminal's output captured in  [pipeline-run-p.txt](01-policy-pipeline/pipeline-run-p.txt) shows that the changed policy was added to the `patern_summary` due to `missing_source` (this can be compared to the unperturbed run [pipeline-run.txt](01-policy-pipeline/pipeline-run.txt) which did not report this policy as flawed). Furthermore, the [escalations (line 39)](01-policy-pipeline/pipeline-run-p.txt#L37) increased from `1` to `2`.
---

### System 2 — schema-enforced two-pass extraction
- **Change I made (file + what I changed):** `appraisal_informal_sqft.txt` - changed the indicated value from `$404,000` to `$800,000`.
- **Command I ran:** `mortgage-extract fixtures/documents/appraisal_informal_sqft.txt`
- **What I predicted:**
The system will flag the document for review as the values were changed to almos double the original amount.
- **What actually happened (paste the key output line):** 
```
flagged for review: Final appraised value of $800,000 is inconsistent with the reconciled value of $410,000 from the Sales Comparison Approach and Cost Approach value of $404,000; this significant discrepancy suggests a data entry error or correction that requires human verification.
```
- **How this differs from the unperturbed run:** The system did not produce a report and instead exited with output stating that the document contained unexpected values and was flagged for review (shown [extract-run-p.txt](02-mortgage-extraction/extract-run-p.txt)) in comparison to the expected extraction results using the defined JSON schema (unperturbed run [extract-run.txt](02-mortgage-extraction/extract-run.txt))

 - *Note: the [extract-run.txt](02-mortgage-extraction/extract-run.txt) contains two runs for `appraisal_informal_sqft.txt` and `income_missing_bonus.txt` separated by the command used to run each extraction respectively.*

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** used `--simulate-timout` parameter to change the input configuration
- **Command I ran:** `supply-chain-investigate meridian`
- **What I predicted:** The system simulated timeout will produce an result with logistics data missing, however, the rest of the report will be generated based on the available data instead of failing entirely.
- **What actually happened (paste the key output line):**
```
### late_shipment_count  _[missing source: timeout reading logistics]_
- missing source: timeout reading logistics
```
- **How this differs from the unperturbed run:** The perturbed run correctly shows that logistics data is missing [timeout-run.txt (line 38)](03-timeout-run.txt#L38) and indicates a timeout occurred while reading the logistics source but the report was finished. Additionally, the [## Contested](03-timeout-run.txt#L34) segment changed due to missing information that did not contest/challange the claims from the other sources of data, essentially affecting the overall conclusions drawn from the available sources completly ignoring the fact that the logistics reported different (`78%`) values compared to the  reported values from the supplier(`95%`).
