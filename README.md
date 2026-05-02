# claude-code-audit-stack

Adversarial verification primitives for [Claude Code](https://claude.com/claude-code). Three production-tested subagents that catch the silent failures Claude Code's defaults miss.

```
$ /plugin install claude-code-audit-stack
```

## What's in the box

| Subagent | What it catches | When to invoke |
|---|---|---|
| **`bot-deploy-verifier`** | Silent skipped restarts. Sibling services bumped by accidental cascade. Service running an old in-memory config because the agent edited the file but forgot to restart. | IMMEDIATELY after any `systemctl restart` or service redeploy |
| **`claim-auditor`** | Probability stacking (`1−(1−p)^N` vs `N×p`). Conditional vs marginal pass-rate confusion. Percentage vs percentage-points mixups. Bootstrap-with-replacement implications. Best-of-N selection bias. | PROACTIVELY whenever a Markdown report contains probability/EV/pass-rate/MC claims |
| **`remote-agent-dispatcher`** | Lost-PID daemon spawns (`$!` captures the bash wrapper, not the claude binary). Orphaned long-running agents. SSH-disconnect kills. | When delegating a >15-min task to a remote host |

Plus a **`audit-on-report-write`** PostToolUse hook that auto-fires `claim-auditor` on every `*.report.md` write/edit. Free quality enforcement on every report your agents generate.

## Why this exists — the silent-skip incident

A Claude Code agent was asked to fix a trailing-stop config bug. It read the YAML, edited the value, and reported "DONE." The on-disk file was correct. The running service had **never been restarted** to pick up the change.

Three trading days passed before anyone noticed. A 3-year backtest replay quantified the cost: a **$106K swing** on a 553-day window, almost entirely because the service was running on the wrong config.

The agent didn't lie. It just didn't verify. `bot-deploy-verifier` exists to be the layer that does verify — adversarially, on every deploy.

[Full incident write-up →](docs/silent-skip-incident.md)

## Quick start

### Install via Claude marketplace (recommended)

```bash
/plugin install claude-code-audit-stack
```

This installs all 3 subagents and the hook. Subagents auto-route via their `description` triggers; the hook fires on `Write|Edit` of `*.report.md`.

### Install manually

```bash
git clone https://github.com/LaterKidsXD/claude-code-audit-stack.git ~/audit-stack
cp ~/audit-stack/agents/*.md ~/.claude/agents/
# (optional) copy the hook into your project's .claude/settings.local.json
cat ~/audit-stack/hooks/hooks.json  # paste the .hooks.PostToolUse block
```

## Examples

- **`bot-deploy-verifier`:** [verify a config swap](examples/bot-deploy-verifier-invocation.md) — shows a real PASS report and a FAILED-WITH-ROLLBACK report
- **`claim-auditor`:** [audit a Monte Carlo report](examples/claim-auditor-invocation.md) — shows the structured P1/P2/P3 severity table on a buggy report
- **`remote-agent-dispatcher`:** [dispatch a 3-year backtest to VPS](examples/remote-agent-dispatcher-invocation.md) — shows the PID-capture trap and how the dispatcher avoids it

## How this is positioned

This stack is **opinionated**. It is NOT trying to be `wshobson/agents` (185 subagents across 25 categories) or `affaan-m/everything-claude-code` (48 subagents + 182 skills). Those collections optimize for breadth.

This stack optimizes for one thing: **catching the silent failures that bring down Claude Code-driven systems in production.** Each subagent is adversarial by design, ships with a documented failure mode it catches, and refuses to PASS until every check is verified.

If you want a frontend-developer agent or a SQL query optimizer, install one of the larger collections. If you want layers of verification that assume the agent above lied to you, install this one.

## Verticals

The general-purpose subagents above (`bot-deploy-verifier`, `claim-auditor`, `remote-agent-dispatcher`) work with any system using `systemd`, `journalctl`, and SSH. The author's home turf is **algorithmic trading bots and quant research pipelines**, and the failure modes catalogued in the subagents (broker-side fills not detected by engine, exit-chain priority bugs, off-spec trail config, claim-auditor catching probability errors in eval reports) come directly from that vertical.

A trading-specific `trading-bot-health` subagent is planned for v0.2 — see the [roadmap](docs/roadmap.md).

## What this stack is NOT

- ❌ Not a general-purpose code review system. Use Anthropic's `/security-review` for security review and `wshobson/agents` for breadth.
- ❌ Not a Claude Code feature replacement. These subagents complement Claude Code's defaults — they don't replace any.
- ❌ Not a prompt-versioning or LLM-Ops platform.
- ❌ Not "the only auditor you need." A real production stack has multiple auditors at different layers; these three are the ones that catch the silent failures most other agents miss.

## Compatibility

- **Claude Code:** any version supporting subagents (Skills + Plugins shipped Oct 2025; Marketplace launched March 2026)
- **Models:** `bot-deploy-verifier` and `remote-agent-dispatcher` use `sonnet` (cheap, mechanical). `claim-auditor` uses `opus` (heavier reasoning required for the math).
- **OS:** anywhere `bash`, `systemctl`, `journalctl`, `ssh`, and `jq` work. Tested on Ubuntu 22.04 / 24.04. Should work on any systemd-based Linux. Non-systemd hosts (Docker Compose, supervisord, pm2) are TODO — see [roadmap](docs/roadmap.md).

## License

MIT. Take it, fork it, repackage it.

## Author

Built and tested in production on a multi-bot trading infrastructure stack — see the case study in [docs/silent-skip-incident.md](docs/silent-skip-incident.md). Author at [LaterKidsXD](https://github.com/LaterKidsXD).

If this stack catches something for you, please open an issue or PR. The standard for new subagents is documented in [docs/roadmap.md](docs/roadmap.md).
