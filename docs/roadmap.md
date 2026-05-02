# Roadmap

## v0.1 (current — May 2026)

Three production-tested subagents:
- `bot-deploy-verifier`
- `claim-auditor`
- `remote-agent-dispatcher`

Plus:
- `audit-on-report-write` PostToolUse hook
- `.claude-plugin/marketplace.json` for one-command install
- Three invocation examples
- This roadmap

## v0.2 (planned)

### `trading-bot-health` subagent

Vertical-specific subagent for live algorithmic trading bots. Combines two complementary failure modes that have zero direct competitors in the existing subagent ecosystem (verified May 2026):

**Live edge-decay detection**
- Pulls realized P&L vs backtest expectations, flags drift on win rate, avg trade, exit-reason mix
- Distinct from `tradermonty/claude-trading-skills` which covers *backtest* iteration stagnation; this covers *live* execution divergence
- Use after each trading day to catch strategy edge degradation before it accumulates

**Broker-state validator**
- OCO bracket integrity checks (parent + stop + TP all present, OCA-grouped, none orphaned)
- Phantom position detection (engine state vs broker state divergence)
- Position-sync gap auditing (where the polling layer would miss a fill)

Why merged into one subagent rather than two: the audience is identical (people running live algorithmic bots) and the two failure modes are usually investigated together when something goes wrong. A single-install footprint lowers the bar to adoption.

Target ship: v0.2, ~2-4 weeks after v0.1 lands.

## v0.3+ (under evaluation, no commitment)

Possible additions based on v0.1 user feedback:

- `mlops-eval-validator` — same shape as claim-auditor but tuned for ML training reports (overfit detection, train/val gap, learning-rate-schedule sanity, sample-size-vs-confidence)
- `incident-postmortem-author` — adversarial Q&A driven by the user's incident write-up; tries to find the *next* failure mode the postmortem hasn't surfaced
- `dependency-drift-detector` — adversarial supply-chain audit (`pip-audit` / `npm audit` / `osv-scanner` integration) with per-package pinning enforcement
- `tests/` directory with smoke tests using the [Claude Code subagent test harness pattern](https://github.com/anthropics/claude-code/issues/6854)

These are speculative; v0.1 needs to land and gather signal before any of them get built.

## What's deliberately NOT planned

- **Generic security scanner.** Anthropic ships `/security-review`, `anthropics/claude-code-security-review` has 4.5k stars, `AgentSecOps/SecOpsAgentKit` has 25+ skills, VoltAgent has `security-auditor`/`security-engineer`. Saturated category — no value-add from another generic entrant.
- **Prompt management / versioning.** Other tools own this layer (LangSmith, PromptLayer, Helicone).
- **MCP-server-as-subagent wrappers.** Different runtime model; out of scope for this stack.

## Contributing

This stack is opinionated. The quality bar for new subagents:

1. **Empty competitive space.** Verified by searching `claudemarketplaces.com`, `buildwithclaude.com`, `VoltAgent/awesome-claude-code-subagents`, `wshobson/agents`. If a similar agent has >100 stars and reasonable activity, don't compete — link to it instead.
2. **Adversarial framing.** This stack's identity is "assume the deploy/report/spawn went wrong unless every check passes." New subagents should fit that framing.
3. **One specific failure mode.** Each subagent should catch one named failure pattern (silent-skipped restart, probability stacking error, lost-PID daemon process). Generic "do all the audit things" subagents go in `wshobson/agents`, not here.
4. **Production-tested.** Don't ship a subagent that hasn't caught a real bug. The `silent-skip-incident.md` doc is the standard — if your subagent doesn't have a story like that, hold off until it does.

PRs welcome under those constraints.
