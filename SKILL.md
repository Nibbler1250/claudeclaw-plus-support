---
name: claudeclaw-plus-support
description: Troubleshoot ClaudeClaw-Plus (bus runtime) when an agent stops responding on Telegram/Discord/Slack — fresh-install silence, the agent spawns but never replies (or dies immediately), or it hangs after a restart. Walks top-down through daemon/poll health, whether the bus agent is actually being fed, and the confirmed startup causes (missing agents[]/busRouting config, the new Bypass-Permissions dialog killing the agent, the MCP multiplexer --mcp-config wedge, session-id collisions), plus the red herrings that waste hours.
---

# ClaudeClaw-Plus support — agent not responding (bus runtime)

ClaudeClaw-Plus defaults to the **bus runtime**: one long-lived `claude` PTY (the *agent*) is spawned at startup, and the bus feeds it **one turn per incoming message**; the agent answers through the `reply` MCP tool. A channel adapter (Telegram/Discord/Slack) long-polls for messages and hands them to the bus.

```
Telegram getUpdates → adapter → bus → agent (claude PTY) → reply tool → Telegram sendMessage
```

When "nothing happens on Telegram," the break is in one of those links. Work **top-down** and confirm each link before moving on. On a fresh install the silence is almost always one of the confirmed causes in Phase 3 — start there if you just installed.

---

## Phase 0 — Orient

- Find the daemon: `systemctl --user status claudeclaw` (or however you run it). Note how long it's been up (see 3d).
- Find its logs. The startup banner prints `Web UI listening on http://127.0.0.1:<port>` and the channels/agents it loaded. If you redirect stdout/stderr to files, app logs live there; otherwise `journalctl --user -u claudeclaw`.
- **Read the startup banner closely.** `no agents` and/or `no adapters` there is a direct hit on 3a.
- Have your bot token (settings.json `telegram.token`) and the web-UI `apiToken` handy.

---

## Phase 1 — Is the daemon actually polling Telegram?

1. **Token valid:** `curl -s "https://api.telegram.org/bot<TOKEN>/getMe"` → returns your bot.
2. **No webhook stealing updates:** `curl -s ".../getWebhookInfo"` → `url` must be empty. A configured webhook makes `getUpdates` return nothing. `pending_update_count > 0` means messages are queuing unconsumed.
3. **Is the long-poll alive?** Run one manual `getUpdates`:
   `curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates?timeout=2"`
   - **`409 Conflict — terminated by other getUpdates request`** → **GOOD.** It proves the daemon is actively long-polling. The poll link is healthy → go to Phase 2.
   - HTTP 200 with updates, or empty → the daemon is **not** polling (adapter never mounted or crashed). Most often the adapter didn't mount → 3a. Also check the error log.
4. If the error log shows `getUpdates failed … ConnectionRefused / FailedToOpenSocket`, the host can't reach `api.telegram.org` (egress / DNS / IPv6). If `curl` to getMe works but the daemon can't, restart and re-check.

---

## Phase 2 — Is the bus agent being fed and generating?

The poll can be healthy while the agent never answers (or no agent exists at all).

1. Find the agent: `pgrep -af 'claude --plugin-dir'`. **No process at all** → the agent never spawned or died immediately → 3a or 3b.
2. Send a test message and **watch the agent's CPU**:
   ```bash
   pid=$(pgrep -f 'claude --plugin-dir' | head -1)
   for i in $(seq 1 10); do ps -o stat=,%cpu= -p "$pid"; sleep 1; done
   ```
   - CPU spikes (STAT flips to `R`) → the agent **is** generating; the problem is downstream (reply delivery / adapter send).
   - **Flat ~0% the whole time** → the agent is **not being fed**. It's wedged or stuck at a startup prompt → Phase 3.
3. Bypass the channel entirely and test the agent through the web UI inject:
   ```bash
   curl -s -m 45 -X POST http://127.0.0.1:<port>/api/inject \
     -H "authorization: Bearer <apiToken>" -H 'content-type: application/json' \
     -d '{"message":"reply with just: ok"}'
   ```
   - `{"ok":true,"result":"ok"}` → the agent pipeline works; your issue is channel-specific → back to Phase 1 / 3a.
   - Hangs or errors → the agent turn never completes → Phase 3.

---

## Phase 3 — Root causes

The first two are the usual fresh-install culprits and are **confirmed in the wild**.

### 3a — Bus mounted with no agent / adapter (setup gap)  ✅ confirmed
The setup wizard may not write the bus config, so:
- `settings.agents[]` is empty → the bus runtime mounts with **zero agents** (startup log shows `no agents`).
- Even with `telegram.token` set, the Telegram **adapter refuses to mount without routing config** (`adapter-wiring.ts:149`) → startup log shows `no adapters`.

**Fix** — in `<cwd>/.claude/claudeclaw/settings.json`:
```jsonc
{
  "agents": [{ "id": "main" }],
  "telegram": {
    "token": "…",
    "busRouting": { "chats": {}, "defaultAgentId": "main" }
  }
}
```
Restart, then confirm the startup banner now shows an agent + the Telegram adapter.

### 3b — New "Bypass Permissions mode" dialog kills the agent  ✅ confirmed
Recent Claude CLI (2.1.x) added a `WARNING: Bypass Permissions mode` confirmation dialog whose **default selection is "No, exit"**. The spawn supervisor auto-dismisses startup dialogs by writing `\r` — but that handler was written for the older `--dangerously-load-development-channels` dialog (where the default is "I am using this for local development", so Enter = accept). On the **new** dialog, a blind `\r` selects **"No, exit"** → every spawned agent dies immediately. Tell-tale log: `No MCP connection for agent_id=<name>`, and `pgrep -f 'claude --plugin-dir'` shows nothing.

**Fix** — pre-accept it so the dialog never renders. Add to `~/.claude/settings.json`:
```jsonc
{ "skipDangerousModePermissionPrompt": true }
```
(this is the post-migration name of `bypassPermissionsModeAccepted`). Restart; the agent should now stay alive and connect MCP.

### 3c — MCP multiplexer `--mcp-config` wedge (ClaudeClaw-Plus #165)
If `settings.mcp.shared` lists servers, each bus agent is spawned with a **synthesized `--mcp-config`** pointing at the in-process MCP multiplexer. If those shared servers are slow or restart-looping at boot, the agent's startup races MCP init and the **first turn can wedge** — the agent then sits idle and every later message queues behind it.
- **Confirm + workaround:** set `settings.mcp.shared: []` and restart. The agent then spawns with **no** `--mcp-config`. If it replies now, this was your cause.
- **Keep the shared MCP:** after a restart, give the agent **~1–2 min** to finish MCP init **before** messaging it, then confirm with the inject test.

### 3d — Session-id collision at boot
Log line: `claude rejected session_id=… as already in use — rotating to … and respawning`. Happens when a previous instance was killed uncleanly and left its session lock. The rotation usually recovers, but a bad boot can leave the agent wedged. **Fix:** a clean restart almost always clears it; avoid `kill -9` on the daemon.

### 3e — First-run "trust this folder?" prompt
A brand-new cwd shows a trust prompt. Plus self-heals by pre-writing `~/.claude.json` → `projects["<cwd>"].hasTrustDialogAccepted: true`. If the agent is idle at 0% CPU right after spawn and you're on a fresh cwd, verify that flag.

---

## Red herrings — don't burn hours here
- **`Claude CLI <ver> NOT in PTY parser's known-good list`** — *benign*. It only controls whether a warning prints, not runtime behavior. The PTY turn parser works across recent CLI versions.
- **`[mcp-multiplexer] server '<x>' crashed: transport closed`** — usually *benign lifecycle*. MCP servers are spawned on demand and exit on EOF; the multiplexer logs that as "transport closed." Before concluding a server is broken, test it directly: pipe one `initialize` JSON-RPC line into it and check for a valid response.
- An MCP server "crashing" is **not** the same as an expired OAuth token. Verify a token refresh actually fails (POST to the token endpoint) before re-authenticating.

---

## Phase 4 — Validate the fix
1. Startup banner shows an agent **and** the channel adapter mounted (rules out 3a).
2. `pgrep -f 'claude --plugin-dir'` shows a live agent that stays up (rules out 3b).
3. The inject test returns your reply (Phase 2.3).
4. A real Telegram message gets a reply, and the app log shows the turn routing.

---

## One-liners
```bash
# daemon + logs (look for "no agents" / "no adapters" in the banner)
systemctl --user status claudeclaw
journalctl --user -u claudeclaw -n 80

# telegram poll health  (409 == the daemon IS polling == good)
curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates?timeout=2"
curl -s "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"

# is there a live agent, and is it generating?  (no process / flat 0% == 3a/3b)
pgrep -af 'claude --plugin-dir'
pid=$(pgrep -f 'claude --plugin-dir' | head -1)
for i in $(seq 1 8); do ps -o stat=,%cpu= -p "$pid"; sleep 1; done

# bypass the channel, test the agent directly
curl -s -m 45 -X POST http://127.0.0.1:<port>/api/inject \
  -H "authorization: Bearer <apiToken>" -H 'content-type: application/json' \
  -d '{"message":"reply with just: ok"}'

# isolate the MCP multiplexer wedge (#165): set  mcp.shared: []  in settings.json, then restart
```

---

*Causes 3a/3b were confirmed in the field via [moazbuilds/claudeclaw#216](https://github.com/moazbuilds/claudeclaw/issues/216) — thanks to @GordonWu for the detailed breakdown.*
