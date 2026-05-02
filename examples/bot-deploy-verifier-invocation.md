# Example: Verify a config swap deployment

You just edited a YAML config and pushed a new commit. You restarted the service. Did it actually pick up the new config? Did anything else get bumped as a side effect?

## The silent-skip failure mode this catches

A common Claude Code workflow:
1. Agent edits `config.yaml` to change `take_profit_pts: 100` → `take_profit_pts: 150`
2. Agent reports "DONE — config updated"
3. **Agent forgets to restart the service**
4. Service is still running on the old `take_profit_pts: 100` in memory
5. Hours later, you notice the bot is hitting the old TP

The deploy-verifier catches this by grepping the *journal* (where the running config is logged) — not the on-disk config. Disk and memory are separately verified.

## Invocation

```
Use bot-deploy-verifier with:
- host: my-vps-host
- service_name: trading-bot.service
- deploy_path: /opt/trading-bot
- expected_config_keys:
  - strategies.donchian.size: 6
  - strategies.donchian.session_end: '10:00'
  - live.take_profit_pts: 150
  - range_filter.threshold: 100
- untouched_services: [data-streamer.service, alert-bot.service, dashboard.service]
- backup_path:
    config_path: /opt/trading-bot/config/live.yaml
    backup_path: /opt/trading-bot/config/live.yaml.bak.20260502_pre_deploy
- restart_required: true
```

## Expected output (PASS case)

```
PASS: trading-bot.service deployed cleanly
- active since: Sat 2026-05-02 21:27:19 UTC
- config-drift: matched ad2629e
- keys verified:
    strategies.donchian.size: 6 ✓
    strategies.donchian.session_end: 10:00 ✓
    live.take_profit_pts: 150 ✓
    range_filter.threshold: 100 ✓
- untouched (unchanged): [data-streamer.service, alert-bot.service, dashboard.service]
```

## Expected output (FAIL with rollback)

```
FAILED + ROLLED BACK: trading-bot.service did NOT pick up the new config

Failure reason: live.take_profit_pts found in on-disk config (150) but journal still
shows old value (100). Service was edited but not restarted, OR was restarted but failed
to reload the config file.

Rollback executed:
- Restored /opt/trading-bot/config/live.yaml.bak.20260502_pre_deploy → live.yaml
- Restarted trading-bot.service
- Re-verified: now active since Sat 2026-05-02 21:30:42 UTC, journal shows take_profit_pts: 100

Action required: investigate why the restart didn't reload. Possible causes:
- Service unit has a stale BOT_CONFIG_FILE env var pointing to a different file
- File permission prevented the daemon user from reading the new config
- A second config layer (CONFIG_FILE_OVERRIDE) is shadowing the change
```

## When NOT to use

- Single-process apps with no journal output → there's nothing to verify against
- Services without a config-drift gate (the verifier will warn but not fail)
- V1/legacy services explicitly listed in `out_of_scope_services` → it refuses
