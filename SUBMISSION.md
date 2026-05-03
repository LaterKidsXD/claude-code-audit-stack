# Anthropic Marketplace Submission Draft

**Form:** https://clau.de/plugin-directory-submission
(redirects to https://code.claude.com/docs/en/plugins#submit-your-plugin-to-the-official-marketplace)

This file is for copy-pasting into Anthropic's submission form. Not part of the plugin itself; can be deleted post-submission.

---

## Plugin name

`claude-code-audit-stack`

## Plugin URL

https://github.com/LaterKidsXD/claude-code-audit-stack

## Author

LaterKidsXD

## Description (one-line — appears in `/plugin > Discover` list)

Adversarial verification primitives: catches silent skipped restarts, probability-stacking errors in reports, and lost-PID daemon spawns on remote hosts.

## Description (long — for the marketplace listing page)

Three production-tested subagents that catch the silent failures Claude Code's defaults miss:

- **`bot-deploy-verifier`** — Adversarial post-deploy verifier. Fires on every `systemctl restart`. Catches the common silent-skip pattern where an agent edits a config file but forgets to restart the service, and catches accidental cascade restarts of sibling services.

- **`claim-auditor`** — Quantitative report auditor. Reads any Markdown report containing probability/EV/MC claims and flags math errors at P1/P2/P3 severity: probability stacking (`1−(1−p)^N` vs `N×p`), conditional vs marginal pass-rate confusion, percentage vs percentage-points mixups, bootstrap-with-replacement implications, best-of-N selection bias.

- **`remote-agent-dispatcher`** — Mechanical scp-and-spawn for autonomous Claude Code agents on remote hosts. Captures the actual claude binary PID (not the bash wrapper PID, which is the common trap) via `pgrep` after a fixed wait. Used when delegating tasks that exceed Claude Code's interactive timeout.

Plus a PostToolUse hook (`audit-on-report-write`) that auto-fires `claim-auditor` on every `*.report.md` write.

The stack is intentionally narrow: it doesn't try to be a 185-agent collection. Each agent catches one specific class of silent failure that has shipped real production incidents — including a documented $106K backtest swing caused by a silent-skipped restart.

## Tags

verification, audit, deployment, devops, systemd, quantitative, trading, remote-execution

## Why this belongs in the marketplace

1. **Empty competitive space.** Verified May 2026 — no equivalent exists in the top 12 community subagent collections (wshobson/agents, VoltAgent/awesome-claude-code-subagents, 0xfurai/claude-code-subagents, etc.). Anthropic's `/security-review` covers a different surface; this is post-deploy state verification + quantitative auditing.

2. **Production-tested.** Each subagent has caught a real bug, documented in `docs/silent-skip-incident.md` and `reports/sample-findings.md`. Not a tutorial repo.

3. **Schema-aligned.** Follows Anthropic's `example-plugin` directory layout (`agents/` not `subagents/`), `plugin.json` minimal schema, `hooks/hooks.json` per `hookify`. JSON-array tools format. Action-routing trigger phrases in every description.

4. **Adversarial-by-design.** All three subagents assume the upstream agent's report is wrong unless every check passes. This is a different stance from generic code-review subagents and complements (rather than competes with) Anthropic's `/security-review`.

## Quality / security checklist

- [x] MIT license
- [x] Read-only auditor (`claim-auditor`) has restricted `tools: ["Read"]`
- [x] No external network calls beyond what `Bash` allows the user to invoke
- [x] No hardcoded credentials, paths, or hostnames
- [x] All user-facing examples use placeholder hosts/paths (`<your-vps>`, `<your-deploy-path>`)
- [x] PostToolUse hook is idempotent (jq one-liner, exits cleanly on non-matching tool calls)
- [x] No background processes started by the plugin
- [x] No filesystem mutations by `claim-auditor` (Read-only)
- [x] `bot-deploy-verifier` does not auto-rollback unless caller explicitly provides `backup_path`
- [x] `remote-agent-dispatcher` refuses to spawn on missing inputs rather than guessing

## Demo / screenshot suggestions (optional, attach to form if requested)

- `reports/sample-findings.md` — real audit findings, redacted
- `docs/silent-skip-incident.md` — the production incident that motivated the stack
- `examples/*.md` — three invocation walkthroughs with PASS + FAIL output

## Pre-submission verify

Run these locally before submitting to catch any last-minute issues:

```bash
cd /workspaces/claude-code-audit-stack
jq . .claude-plugin/plugin.json   # plugin.json valid JSON?
jq . hooks/hooks.json             # hooks.json valid JSON?
python3 -c "
import yaml, re
from pathlib import Path
for f in sorted(Path('agents').glob('*.md')):
    text = f.read_text()
    m = re.match(r'^---\n(.*?)\n---', text, re.DOTALL)
    d = yaml.safe_load(m.group(1))
    print(f'  {f.name}: {list(d.keys())}')
"
```

Expected: 3 agent files with `['name','description','model','color','tools']` keys each.
