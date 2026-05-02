# The silent-skip incident — why this stack exists

Production trading bot, May 2026. Claude Code agent is asked to fix a config bug.

The agent:
1. Reads the affected `config.yaml`
2. Identifies the wrong value: `trailing_stop_min_dist: 4` should be `0`
3. Edits the file
4. Reports "DONE — fixed the trailing-stop floor"

Three days later, the bot's daily P&L is still anomalous. Investigation:

- ✅ The on-disk config has `trailing_stop_min_dist: 0`. Edit landed.
- ❌ The running journal still shows `Trail: 4/12 ATR (default)`. The service was **never restarted** to pick up the new config.

Three days of trading on the wrong config. A 3-year backtest replay later showed the off-spec trail had been silently destroying 98.7% of trades — a $106K swing on a 553-day window. Most of that was during this incident's three-day window.

The agent didn't lie. It did edit the file. It just didn't restart the service. And the "DONE" felt convincing enough that no one verified.

## The pattern

This is the **silent-skip pattern**: the agent did 80% of the work and reported 100%. The missing 20% — the *load-bearing* 20% — silently fails. Nobody notices because the deploy *appears* to have succeeded.

There are dozens of variants:
- Edited `config.yaml` but service uses `config.production.yaml` (different file)
- Restarted the service but the systemd unit ran an older Docker image (image not rebuilt)
- Restarted but the env var pointing to the config wasn't reloaded (only set on first start)
- Restarted but a sibling service was bound to it via `BindsTo=` and got bumped silently
- Edited and pushed but the deploy server pulls from a different branch
- Restarted on the wrong host (multi-host deploys)

Each one ends with: *the on-disk artifact looks correct, the running process is on the old behavior, the agent reported done, the user trusted the report.*

## What `bot-deploy-verifier` checks (each catches a real production incident)

| Check | What silent-skip it catches |
|---|---|
| `systemctl is-active <service>` post-restart | Service crashed on restart and was never re-checked |
| Grep journal for the new config keys | Edit landed, but service never restarted — journal still shows old |
| Grep journal for `config-drift check: ... matches HEAD <sha>` | Service running OLDER commit than git HEAD |
| Pre-restart timestamp snapshot of untouched services + post-restart compare | Sibling services accidentally cascaded by `BindsTo=` |
| `ActiveEnterTimestamp` change check | Service crash-restart-looped invisibly during the deploy window |
| Optional: backup-path rollback on failure | Failed deploy left the service in a broken state with no recovery |

The verifier is intentionally **adversarial**. It assumes the deploy went wrong unless every check passes. It's slower than `systemctl restart && echo done` — but it's the difference between a 3-day silent regression and a 30-second "FAILED, did not pick up new config" alert.

## Why a subagent and not a script

A standalone bash script would work for one specific deploy pattern. The subagent works because:

1. The caller passes in **what config keys to expect** as part of the invocation. No coupling to a specific bot's config schema.
2. The caller passes in **which sibling services should NOT be bumped**. Your specific deploy topology is encoded in the prompt, not the script.
3. The agent's adversarial framing ("assume the deploy went wrong") gets baked into its reasoning during verification — it actively checks for the patterns above rather than mechanically running through a checklist.
4. The auto-rollback path is conditional on `backup_path` being provided — the caller decides when rollback is safe vs when leaving the service mid-broken is preferable.

Scripts are good when the question is mechanical. Subagents are good when the question is "did this thing actually happen, and what's the simplest evidence that proves it?"

## The cost of the alternative

Fix the silent-skip pattern, you save:
- Hours of debugging the next time a deploy "worked" but produced wrong behavior
- The user-trust cost of "the agent said done" turning out false
- The opportunity cost of trading days lost to silent regressions
- The accuracy of any post-deploy analysis (your trade DB is corrupted by partial-deploy state)

The deploy-verifier subagent is ~150 lines of prompt. The 3-day silent regression cost (in the production incident this was modeled on) was ~$3,200 of mis-attributed P&L. The math is unkind to NOT having this.

## Use it on every deploy

Claude Code's defaults trust agents to self-report. That trust is the silent-skip vector. Run the verifier on every config swap, every code-change deploy, every container restart that touches a stateful service. The cost is 30 seconds; the upside is one less production incident.
