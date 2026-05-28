# claudeclaw-plus-support

A Claude Code **skill** for troubleshooting [ClaudeClaw-Plus](https://github.com/TerrysPOV/ClaudeClaw-Plus) (bus runtime) when an agent stops responding on Telegram / Discord / Slack — fresh-install silence, the agent spawns but never replies, or it hangs after a restart.

## Why

On the bus runtime the daemon can look completely alive (process up, polling Telegram) while the agent PTY never actually gets fed a turn — so the obvious checks don't surface the problem. This skill walks the message path top-down and goes straight to the handful of startup causes that produce the silence.

## What it covers

- **Poll health** — is the daemon really long-polling Telegram? (incl. the `409 Conflict` "this is the good sign" check, webhook hijack, egress errors)
- **Is the agent being fed?** — watch the agent's CPU on a test message, and a channel-bypassing `/api/inject` test
- **The three fresh-install / wedge causes**
  - `bypassPermissions` + startup trust / `--dangerously-load-development-channels` confirmation prompts freezing the PTY
  - the MCP multiplexer synthesized `--mcp-config` wedge (ClaudeClaw-Plus #165)
  - session-id collisions at boot
- **Red herrings** that waste time — the CLI "not in known-good list" warning, and the multiplexer "transport closed" log lines

## Install

```bash
git clone https://github.com/Nibbler1250/claudeclaw-plus-support
cp -r claudeclaw-plus-support ~/.claude/skills/claudeclaw-plus-support
```

Then either invoke it directly with `/claudeclaw-plus-support`, or just describe your symptom ("Plus agent not responding on Telegram") and it triggers automatically.
