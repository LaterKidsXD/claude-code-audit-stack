# Example: Dispatch a long-running backtest to VPS

You have a 3-year walk-forward backtest that takes ~2 hours to run. Claude Code's interactive timeout is 15 minutes. The remote-agent-dispatcher hands the work off to a daemon-user Claude Code process on a VPS, frees your interactive session, and gives you a PID + heartbeat path to monitor progress.

## The PID-capture trap this avoids

Naive dispatch:
```bash
ssh vps "claude -p '$(cat handoff.md)' --model opus &"
PID=$!
echo $PID > pidfile
```

The `$!` captures the **bash wrapper's PID**, not the actual claude binary's PID. When the wrapper exits seconds later, your pidfile points at a dead process. The real claude (which is still running) becomes orphaned and untrackable.

This subagent uses `pgrep -f` after a fixed 18-second wait to find the actual claude process:

```bash
ssh <host> "ps -ef | grep -E ' claude\$' | grep -v grep"
```

The trailing `\$` matches lines where the command ends in literally "claude" (the binary), filtering out the bash wrapper whose command line ends in something like `cat handoff.md`.

## Invocation

```
Use remote-agent-dispatcher with:
- spec_path: /tmp/HANDOFF_3YR_BACKTEST.md
- host: research-vps
- working_dir_on_host: /opt/research/
- model: opus
- report_basename: HANDOFF_3YR_BACKTEST
```

## Expected output

```
DISPATCHED: HANDOFF_3YR_BACKTEST.md as PID 350721 on research-vps
- working dir: /opt/research/
- model: opus
- run log: /opt/research/HANDOFF_3YR_BACKTEST.md.run.log
- expected heartbeat: /opt/research/HANDOFF_3YR_BACKTEST.heartbeat
- expected report: /opt/research/HANDOFF_3YR_BACKTEST.report.md
- pid file: /opt/research/HANDOFF_3YR_BACKTEST_agent.pid

Caller should poll heartbeat for status. Recommended: schedule first check in 15-30 min.
```

## What "BLOCKED" looks like

```
BLOCKED: claude binary not found in daemon user's PATH on research-vps

Failure trace:
- scp: succeeded, file at /opt/research/HANDOFF_3YR_BACKTEST.md
- spawn: bash wrapper started, but claude binary did not start within 18s
- run log: "/usr/bin/env: 'claude': No such file or directory"

Suggested fix: ssh research-vps "sudo -u daemon which claude" — if empty, install
claude via npm-global as the daemon user, OR add the path to its PATH env in the
spawn command.
```

## First-time setup verification (smoke test)

Before pointing this at a real VPS, verify your local config works:

```
Use remote-agent-dispatcher with:
- spec_path: /tmp/HANDOFF_TEST.md
- smoke_test_localhost: true
```

This runs the full scp → spawn → kill flow against `localhost`, validates the PID-capture pattern, and reports `LOCAL_SMOKE_TEST_OK` or `LOCAL_SMOKE_TEST_FAILED <reason>` without requiring a real VPS or burning API calls.

## Keeping spawned agents alive across SSH disconnect

The `nohup` + `&` pattern in the dispatcher is sufficient for most cases. For agents expected to run >1 hour or that need restart-on-failure, layer one of these on top:

- **systemd one-shot service** wrapping the spawn (most robust)
- **tmux/screen** session on the host (the user can attach later for visibility)
- **PM2** if the host already runs Node.js infra (process supervisor with restart policy)

This subagent is a launch primitive only. It does NOT manage post-spawn lifecycle.

## Pairs well with

`bot-deploy-verifier` runs adversarial verification *after* a deploy completes. `remote-agent-dispatcher` runs *before* — when you need work to start somewhere else. Combined: dispatch a long-running CI job remotely, return immediately, and verify it on completion.
