# Example: Audit a Monte Carlo simulation report

A Claude Code agent just generated `eval_pass_rate_monte_carlo.report.md` summarizing a 10,000-iteration MC run on a trading-eval pass-rate model. Before you act on its conclusions, run the claim-auditor.

## The math errors this catches (real examples from production reports)

- **Probability stacking:** "3 evals at 35.7% per eval = ~90% chance one passes" → actual is `1 − 0.643^3 = 73.4%`. A 16.6 percentage-point gap that materially changes whether the strategy is +EV.
- **N-worst tail stacking:** "5 worst-case days back-to-back = 5 × −$5,580 = −$27,900" → joint probability of 5 worst-of-1000 days happening in succession is ~10⁻¹⁵. The realistic 5-bust streak loss is closer to `5 × mean_bust ≈ −$7,500`.
- **Conditional vs marginal confusion:** "P(pass ≤5d) at 30%" treated as "30% pass rate" — these are different. 30% of all evals pass within 5 days; the other 70% includes evals that pass later AND evals that bust.
- **Percentage-vs-percentage-points:** "Pass rate dropped from 60% to 50% — a 10% drop" → that's 10 *percentage points*. The relative drop is 16.7%. Reports often slur this and downstream math uses the wrong one.
- **Best-of-N selection bias:** "Top config has 67.7% pass rate (selected from 504 configs)." Out-of-sample, expect lower — that 67.7% includes random luck.

## Invocation

```
Use claim-auditor with:
- report_path: /tmp/eval_pass_rate_monte_carlo.report.md
- severity_floor: P2
- output_format: markdown
```

## Expected output (errors found)

```
# Claim Audit — eval_pass_rate_monte_carlo.report.md

## P1 (decision-shaping, must address)

| Quote | Why wrong | Correct number |
|---|---|---|
| "3 evals at 35.7% per eval = ~90% chance one passes" | P(≥1 of N) = 1−(1−p)^N, not N×p | 73.4%, not 90% |
| "5-worst stacked = 5 × −$5,580 = −$27,900" | Joint P(5 worst-of-10K) ≈ 10⁻²⁰; use typical bust × N | typical 5-streak ≈ −$7,500 |

## P2 (misleading framing)

| Quote | Why misleading | Correction or context |
|---|---|---|
| "Top config X has 67.7% pass rate" | Selected from 504 configs — selection bias inflates point estimate | Out-of-sample expectation likely 5-15 percentage points lower |

## P3 (pedantic)

(none flagged at this severity floor)

## Summary

- 2 P1 errors — both materially change the eval-cost economics
- 1 P2 framing — top-config pass rate has selection-bias inflation
- 0 P3 nits
- **Recommended action:** Recompute eval-cost economics with corrected probabilities BEFORE deploying. Out-of-sample pass-rate for top-selected config should also be discounted.
```

## Expected output (clean report)

```
# Claim Audit — eval_pass_rate_monte_carlo.report.md

No errors at or above P2. Report is decision-safe for quantitative claims within audit scope.
```

## JSON output (for CI integration)

Set `output_format: json` and the same data returns as machine-readable JSON. Pipe into your CI to **block PR merges** when P1 errors are found:

```bash
claude-code agent claim-auditor --report eval_report.md --json | \
    jq -e '.summary.decision_safe == true' || exit 1
```

## Pairs well with

The `audit-on-report-write` PostToolUse hook in `hooks/audit-on-report-write.json` auto-fires this subagent on any `*.report.md` Write/Edit. Add it to your project's `.claude/settings.local.json` and you get free quality enforcement on every report.
