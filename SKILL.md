---
name: claudeclaw-plus-support
description: Troubleshoot ClaudeClaw-Plus (bus runtime) when an agent stops responding on Telegram/Discord/Slack — fresh-install silence, the agent spawns but never replies, or it hangs after a restart. Walks top-down through daemon/poll health, whether the bus agent is actually being fed, and the known startup gotchas (bypassPermissions trust prompts, the MCP multiplexer --mcp-config wedge, session-id collisions), plus the red herrings that waste hours.
---

# ClaudeClaw-Plus support — agent not responding (bus runtime)

ClaudeClaw-Plus defaults to the **bus runtime**: one long-lived `claude` PTY (the *agent*) is spawned at startup, and the bus feeds it **one turn per incoming message**; the agent answers through the `reply` MCP tool. A channel adapter (Telegram/Discord/Slack) long-polls for messages and hands them to the bus.

```
Telegram getUpdates → adapter → bus → agent (claude PTY) → reply tool → Telegram sendMessage
```

When "nothing happens on Telegram," the break is in one of those links. Work **top-down** and confirm each link before moving on. On a fresh install the silence is almost always one of the three root causes in Phase 3.

---

## Phase 0 — Orient

- Find the daemon: `systemctl --user status claudeclaw` (or however you run it). Note how long it's been up (see 3c).
- Find its logs. The startup banner prints `Web UI listening on http://127.0.0.1:<port>` and the channels it loaded. If you redirect stdout/stderr to files, app logs live there; otherwise `journalctl --user -u claudeclaw`.
- Have your bot token (settings.json `telegram.token`) and the web-UI `apiToken` handy.

---

## Phase 1 — Is the daemon actually polling Telegram?

1. **Token valid:** `curl -s "https://api.telegram.org/bot<TOKEN>/getMe"` → returns your bot.
2. **No webhook stealing updates:** `curl -s ".../getWebhookInfo"` → `url` must be empty. A configured webhook makes `getUpdates` return nothing. `pending_update_count > 0` means messages are queuing unconsumed.
3. **Is the long-poll alive?** Run one manual `getUpdates`:
   `curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates?timeout=2"`
   - **`409 Conflict — terminated by other getUpdates request`** → **GOOD.** It proves the daemon is actively long-polling. The poll link is healthy → go to Phase 2.
   - HTTP 200 with updates, or empty → the daemon is **not** polling (adapter never started or crashed). Check the startup log for the Telegram adapter and the error log.
4. If the error log shows `getUpdates failed … ConnectionRefused / FailedToOpenSocket`, the host can't reach `api.telegram.org` (egress / DNS / IPv6). If `curl` to getMe works but the daemon can't, it's usually transient/IPv6 — restart and re-check.

---

## Phase 2 — Is the bus agent being fed and generating?

The poll can be perfectly healthy while the agent never answers.

1. Find the agent: `pgrep -af 'claude --plugin-dir'`.
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
   - `{"ok":true,"result":"ok"}` → the agent pipeline works; your issue is channel-specific → back to Phase 1.
   - Hangs (HTTP 000 after the timeout) → the agent turn never completes → Phase 3.

---

## Phase 3 — The three root causes (fresh install / wedge)

### 3a — `bypassPermissions` + startup trust/confirmation prompts
The bus spawns the agent with `--permission-mode bypassPermissions` so *tool* calls don't block. But interactive **startup prompts** can still freeze the PTY before it's ready:
- The first run in a new working directory shows a **"trust this folder?"** prompt. Plus self-heals by pre-writing `~/.claude.json` → `projects["<cwd>"].hasTrustDialogAccepted: true`. If the agent sits idle at 0% CPU right after spawn, verify that flag exists for the agent's cwd.
- Newer Claude CLI versions add a **confirmation for `--dangerously-load-development-channels`** ("1. I am using this for local development / 2. Exit"). If the supervisor doesn't auto-answer it, the agent waits at the dialog forever.

**Fix:** ensure trust is pre-accepted for the cwd and the agent runs with `bypassPermissions`. If a recent CLI update introduced a new startup dialog, that new prompt is your culprit.

### 3b — MCP multiplexer `--mcp-config` wedges the agent (ClaudeClaw-Plus #165)
If `settings.mcp.shared` lists servers, each bus agent is spawned with a **synthesized `--mcp-config`** pointing at the in-process MCP multiplexer. If those shared servers are slow or restart-looping at boot, the agent's startup races MCP init and the **first turn can wedge** — the agent then sits idle and every later message queues behind it.
- **Confirm + workaround:** set `settings.mcp.shared: []` and restart. The agent then spawns with **no** `--mcp-config` (pre-#165 behavior). If it replies now, this was your cause. (You lose the shared MCP tools until properly fixed.)
- **Keep the shared MCP:** after a restart, give the agent **~1–2 min** to finish MCP init **before** messaging it, then confirm with the inject test. A clean restart on a healthy boot is reliable.

### 3c — Session-id collision at boot
Log line: `claude rejected session_id=… as already in use — rotating to … and respawning`. This happens when a previous daemon instance was killed uncleanly and left its session lock. The rotation usually recovers, but a bad boot can leave the agent wedged.

**Fix:** a **clean restart** (`systemctl --user restart claudeclaw`) almost always clears it. Avoid `kill -9` on the daemon; prefer a graceful stop so the lock is released.

---

## Red herrings — don't burn hours here
- **`Claude CLI <ver> NOT in PTY parser's known-good list`** — *benign*. It only controls whether a warning prints, not runtime behavior. The PTY turn parser works across recent CLI versions (verified by replaying real PTY output through the parser).
- **`[mcp-multiplexer] server '<x>' crashed: transport closed`** — usually *benign lifecycle*. MCP servers are spawned on demand and exit on EOF; the multiplexer logs that as "transport closed." Before concluding a server is broken, test it directly: pipe one `initialize` JSON-RPC line into it and check for a valid response.
- An MCP server "crashing" is **not** the same as an expired OAuth token. Verify a token refresh actually fails (POST to the token endpoint) before re-authenticating.

---

## Phase 4 — Validate the fix
1. The inject test returns your reply (Phase 2.3).
2. A real Telegram message gets a reply, and the app log shows the turn routing.
3. After any restart, wait for the startup banner **and** the first turn to settle before trusting it.

---

## One-liners
```bash
# daemon + logs
systemctl --user status claudeclaw
journalctl --user -u claudeclaw -n 50

# telegram poll health  (409 == the daemon IS polling == good)
curl -s "https://api.telegram.org/bot<TOKEN>/getUpdates?timeout=2"
curl -s "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"

# is the agent generating?  (flat 0% == not fed == wedged)
pid=$(pgrep -f 'claude --plugin-dir' | head -1)
for i in $(seq 1 8); do ps -o stat=,%cpu= -p "$pid"; sleep 1; done

# bypass the channel, test the agent directly
curl -s -m 45 -X POST http://127.0.0.1:<port>/api/inject \
  -H "authorization: Bearer <apiToken>" -H 'content-type: application/json' \
  -d '{"message":"reply with just: ok"}'

# isolate the MCP multiplexer wedge (#165): set  mcp.shared: []  in settings.json, then restart
```
