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

- **Change I made (file + what I changed):** Removed bonus field
- **Command I ran:** .venv/Scripts/mortgage-extract fixtures/documents/income_missing_bonus.txt --mode replay
- **What I predicted:** The system would make those fields null. 
- **What actually happened (paste the key output line):**

"bonus_monthly": null,
"bonus_ytd": null,

- **How this differs from the unperturbed run:**

The bonus fields are null because the file didn't have incoming bonus field. This is better than an invented field because it would be harder to differ it from a real value.

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