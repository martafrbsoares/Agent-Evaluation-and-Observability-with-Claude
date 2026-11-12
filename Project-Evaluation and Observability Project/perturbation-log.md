# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Removed PREMIUM SUMMARY - Total Policy Premium field
- **Command I ran:** .\.venv\Scripts\policy-extractor pipeline data/perturb --routing-out routing_decisions.json --seed 42
- **What I predicted:** The system would complete the script, but show an error in the end
- **What actually happened (paste the key output line):** 

{
  "decisions_written": 0,
  "auto_approve": 0,
  "human_review": 0,
  "spot_check": 0,
  "escalations": 1,
  "pattern_summary": {
    "premium_amount_absent": {
      "count": 1,
      "policies": [
        "POL-2025-001-PERTURBATION"
      ],
      "categories": [
        "missing_source"
      ]
    }
  },
  "output_path": "routing_decisions.json"
}

- **How this differs from the unperturbed run:**

The unperturbed run made multiple API calls, while the perturbed run only run 1 API call. The unperturbed run displayed the normal behaviour, assigning some to auto_approve, human_review or spot_check. The perturbed run didn't assign any to those fields and displayed one escalation.

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Removed Property Address field
- **Command I ran:** .venv/Scripts/mortgage-extract fixtures/documents/appraisal_informal_sqft_perturbation.txt --mode auto
- **What I predicted:** The system would break because that is a mandatory field. 
- **What actually happened (paste the key output line):**

Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "C:\claude_project\system-2\.venv\Scripts\mortgage-extract.exe\__main__.py", line 5, in <module>
    sys.exit(main())
             ~~~~^^
  File "C:\claude_project\system-2\mortgage_extractor\__main__.py", line 60, in main
    extraction = pipeline.run(document_text)
  File "C:\claude_project\system-2\mortgage_extractor\pipeline.py", line 60, in run
    return self.extract(document_text, classification.document_type)
           ~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "C:\claude_project\system-2\mortgage_extractor\pipeline.py", line 128, in extract
    return MortgageExtraction.model_validate(block.input)
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^
  File "C:\claude_project\system-2\.venv\Lib\site-packages\pydantic\main.py", line 792, in model_validate
    return cls.__pydantic_validator__.validate_python(
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^
        obj,
        ^^^^
    ...<5 lines>...
        by_name=by_name,
        ^^^^^^^^^^^^^^^^
    )
    ^
pydantic_core._pydantic_core.ValidationError: 1 validation error for MortgageExtraction
loan.amount
  Input should be a valid number [type=float_type, input_value=None, input_type=NoneType]
    For further information visit https://errors.pydantic.dev/2.14/v/float_type

- **How this differs from the unperturbed run:**

The unperturbed run was finished as expected, while the perturbed run didn't finish and threw an error. This is because the address field is required and the field couldn't be null as per the schema defined ("address": {"type": "string"}).

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Simulate a timout
- **Command I ran:** .venv/Scripts/supply-chain-investigate meridian --offline --simulate-timeout
- **What I predicted:** The system would be unable to read all the logistics
- **What actually happened (paste the key output line):**

## Incomplete
### late_shipment_count  _[missing source: timeout reading logistics]_
- missing source: timeout reading logistics

- **How this differs from the unperturbed run:**

The run still finished, but it wasn't able to gather the logistics due to the timout.

 late_shipment_count  says "timeout reading logistics", while  production_capacity_utilization  says "no source reported this metric". The first is a retryable failure and the second is a real gap.

 With logistics gone, the 95.0 vs 78.0 disagreement disappeared. The timeout run shows  on_time_delivery_rate  as a clean 95.0% under Well-Established, with no escalation. That looks reassuring but is misleading, because the number the supplier reports about itself is no longer being checked against the logistics data. The "Sources unavailable" banner is the only warning that this clean result is partial.

 The unperturbed run is able to check against the logistics data because there is no timeout.