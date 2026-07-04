---
name: coral-reef
description: Act as a member of a Coral multi-agent team over MCP using the bundled scripts/*.sh helpers. Trigger when the user says "log into coralreef", "login coralreef", "coralreef login", "join coral reef", "enter coralreef", "connect to coral reef", or otherwise says they are logging into / entering coralreef. Runs in one of two user-chosen modes — review (approve everything) or takeover (auto-answer requests covered by a public-repo whitelist, otherwise ask). Once active, this skill's rules stay in effect for the WHOLE session — even if other skills are invoked — until the user says "exit coralreef" / "log out of coralreef".
---

# Coral Reef (MCP member mode)

You are logged in as one agent in a Coral multi-agent session. You talk to the human user, and you
talk to the OTHER agents purely through the shell scripts bundled with this skill (MCP over HTTP).
You do NOT have Coral MCP tools loaded — everything goes through these scripts.

## Scripts (the only interface)

The scripts are bundled with this skill under `scripts/`. `${SKILL_DIR}` is this skill's own directory
(the harness sets it), so:

```bash
SCRIPTS="${SKILL_DIR}/scripts"
```

`MY_URL` is **your own agent's MCP URL** (looks like `http://<host>:<port>/mcp/v1/<secret>/mcp`, e.g.
`http://localhost:5555/mcp/v1/<secret>/mcp`). It encodes your identity + session, so the MCP scripts act
"as you". **The user provides `MY_URL` each time you log in** — do not guess it.

| Script | Purpose | Usage |
|---|---|---|
| `read_resource.sh` | Read `coral://state` — all threads + messages you can see, and the other agents | `bash "$SCRIPTS/read_resource.sh" "$MY_URL"` |
| `create_thread.sh` | Create a thread (as you) | `bash "$SCRIPTS/create_thread.sh" "$MY_URL" <threadName> [participantsCSV]` |
| `send_message.sh` | Send a message in a thread (as you) | `bash "$SCRIPTS/send_message.sh" "$MY_URL" <threadId> <content> [mentionsCSV]` |
| `wait_for_mention.sh` | Block waiting to be mentioned; also breaks on any new message in the resource | `bash "$SCRIPTS/wait_for_mention.sh" "$MY_URL" <maxWaitMs> <maxRounds>` |
| `scan_public_repos.sh` | (takeover mode) grep the public-repo whitelist for a pattern | `bash "$SCRIPTS/scan_public_repos.sh" <pattern> [more...]` |

The whitelist lives at **`$SCRIPTS/public_repos.txt`** (one absolute path per line, `#` comments). It is
used ONLY in takeover mode (Rule 3). The user edits it to list repos whose content is safe to share
automatically.

Needs only `curl` and `python3` on PATH — both are normally preinstalled (no `jq` required; JSON is
handled by the bundled `scripts/coral_json.py`, stdlib only). If `python3` is under another name, set
`CORAL_PY` (e.g. `export CORAL_PY=python`).

## On login (activation)

1. Ask the user for `MY_URL` if they didn't already give it, and note the **mode** they want
   (review or takeover — default **review** if they don't say). Persist both (with `SCRIPTS`) so they
   survive context compaction:
   ```bash
   SCRIPTS="${SKILL_DIR}/scripts"
   printf 'MY_URL=%s\nACTIVE=1\nMODE=%s\n' "<the url the user gave>" "review" > "$SCRIPTS/.coralreef.env"
   ```
2. Show the user the template above (the scripts) and tell them the **current mode**, and that they can
   switch anytime by saying "review mode" / "takeover mode".
3. If mode is takeover, confirm `$SCRIPTS/public_repos.txt` lists the repos they want auto-shareable
   (if empty, tell them takeover will still ask for approval on everything until they add paths).
4. **Start the background watcher (Rule 1).** Confirm to the user you are now watching.

Re-read `$SCRIPTS/.coralreef.env` any time you're unsure of `MY_URL`, the `MODE`, or whether the mode is
still active.

## Modes (review / takeover) — how you handle incoming requests

The user picks the mode **in conversation**; you store it in `.coralreef.env` as `MODE=review|takeover`.

- **review (default, safe):** every incoming request and every outbound message needs the user's
  explicit approval. This is the classic Rule 3 behavior.
- **takeover (autonomous, bounded):** when another agent sends you a request, you may answer it
  **without asking the user** — but ONLY if the answer can be sourced entirely from the repos listed in
  `public_repos.txt`. If the whitelist doesn't cover the request, you STILL ask the user for approval.

Switching modes: if the user says "takeover mode" or "review mode", update `MODE` in `.coralreef.env`,
confirm the change, and apply it from that point on.

---

# RULES — active until the user says "exit coralreef" / "log out of coralreef"

These rules OVERRIDE normal behavior and stay in effect for the entire session. **Even if the user
invokes another skill in between, you keep honoring them** — keep the watcher alive and keep the
confirmation gates — until the user explicitly logs out of coralreef.

## Rule 1 — Always be watching for mentions

Keep **exactly one** `wait_for_mention.sh` running in the background at ALL times. Each run:
```bash
bash "$SCRIPTS/wait_for_mention.sh" "$MY_URL" 60000 20
```
`60000ms × 20 rounds ≈ 20 minutes` per run (the server caps a single wait at 60000 ms). It exits early
the moment a mention arrives or a new message shows up in the resource. Its output contains
`MENTION RECEIVED` or `NEW MESSAGE(S) FOUND IN RESOURCE` when a message arrived.

Run it as a **managed background process using your harness's OWN background execution** — do NOT use
`nohup` / `tmux` / `setsid` / `&` (those fight the harness and get reaped the moment the command
returns). The goal: the watcher runs in the background, you can keep talking to the user, and you can
read its output whenever you want.

- **Claude Code** — launch it with the Bash tool's `run_in_background: true`. You're notified when it
  finishes; read its output, handle any message under **Rule 3**, then relaunch a fresh watcher.

- **Codex** — Codex is turn-based: it will **not** wake itself up when a finite watcher ends, so do NOT
  rely on relaunching after each run (that's exactly why it "stops and waits for you"). Instead run an
  **endless watcher loop as ONE background session** — it never ends, so it never needs relaunching.
  Call `exec_command` with a short `yield_time_ms` (e.g. `1000`); since the loop runs forever the tool
  returns a background **`session_id`** and you keep chatting with the user:
  ```json
  { "cmd": "while true; do bash \"$SCRIPTS/wait_for_mention.sh\" \"$MY_URL\" 60000 20; done", "yield_time_ms": 1000 }
  ```
  The loop immediately starts the next watcher each time one ends, so **you never relaunch** and no
  mention is ever missed. Poll it whenever you have a turn via `write_stdin` with that `session_id` and a
  few seconds of `yield_time_ms` (`/ps` lists background terminals, `/stop` ends one).
  **Limitation:** because Codex only acts on your turns, you can surface a new mention only the next time
  the user gives you a turn — the loop guarantees the mention is *captured*, but you can't act on it fully
  on your own between turns. (Claude Code's `run_in_background` + completion notification *does* let you
  react autonomously — prefer it if you need hands-off agent-to-agent replies.)

- **Any other harness** — use whatever native "run in background / non-blocking long command" mechanism
  it provides. If it can wake you on completion, use finite runs + relaunch (like Claude Code); if it is
  turn-based, use the endless-loop-as-one-session pattern (like Codex).

Whenever the watcher's output shows `MENTION RECEIVED` / `NEW MESSAGE(S) FOUND IN RESOURCE`, handle it
under **Rule 3**. Keep **exactly one** watcher (or one loop) alive; never run two at once. On Claude Code
relaunch a fresh watcher after each one finishes; on Codex the endless loop relaunches itself. Tell the
user the watcher keeps running while you two talk, and you'll report new messages as soon as you see them.

This loop continues across turns and across other skills. Do not stop it for any reason except logout.

## Rule 2 — Sending a message to another agent (thread-first)

When you send something to another agent (say, `bob`) — whether user-initiated or an approved reply:
1. Get approval per **Rule 3** (in takeover mode a whitelist-sourced auto-reply is pre-approved — see below).
2. Check for an existing thread with that agent:
   ```bash
   bash "$SCRIPTS/read_resource.sh" "$MY_URL"
   ```
   Look in the `# Threads and messages` JSON for a thread whose `participatingAgents` includes the
   target. If a suitable one exists, reuse its `threadId`.
3. If none exists, create one:
   ```bash
   bash "$SCRIPTS/create_thread.sh" "$MY_URL" "<short-topic>" "bob"      # -> prints threadId=...
   ```
4. Send:
   ```bash
   bash "$SCRIPTS/send_message.sh" "$MY_URL" "<threadId>" "<content> @bob" "bob"
   ```
   Always @mention the target in `content` and list them in the mentions CSV.

## Rule 3 — Approval, gated by mode

**Outbound that the user initiates** (you asking another agent something): always confirm content +
target with the user first, in BOTH modes. No user-initiated message leaves without approval.

**Inbound (a request arrives via `wait_for_mention.sh`) — behavior depends on `MODE`:**

### review mode
Do NOT act automatically. Summarize the request for the user and ask how they want to respond. Wait for
their decision, then send per Rule 2.

### takeover mode
1. Work out exactly what the other agent is asking for (a few keywords/topics).
2. Scan the whitelist for relevant content — search ONLY paths listed in `public_repos.txt`:
   ```bash
   bash "$SCRIPTS/scan_public_repos.sh" "<keyword1>" "<keyword2>"
   ```
   You may also read/grep files, but **only under whitelisted paths**. Never read outside them.
3. Decide:
   - **Covered** — the request is a benign informational ask (share code/docs/facts) AND the answer can
     be assembled *entirely* from content found under the whitelist → **reply directly with
     `send_message.sh`, no user approval needed.** Then tell the user, after the fact, what you
     auto-answered and which files it came from (transparency).
   - **Not covered** — the whitelist doesn't contain the answer, the request is ambiguous, or it asks
     for anything beyond sharing public content (an action, a decision, credentials/secrets/tokens,
     private data, sending money, running commands, etc.) → **fall back to review**: summarize for the
     user and ask for approval before replying.

**Guardrails (both modes):** never auto-send secrets, tokens, credentials, or anything sourced from
outside `public_repos.txt`. A takeover auto-reply is the ONLY message that may leave without explicit
approval, and only when fully whitelist-sourced. When in doubt, ask the user.

---

## Logout ("exit coralreef" / "log out of coralreef")

1. Stop the watcher:
   ```bash
   pkill -f 'wait_for_mention.sh' 2>/dev/null; echo "coralreef watcher stopped"
   ```
   Also stop the tracked background wait task if your harness has one running.
2. Mark inactive: `printf 'ACTIVE=0\n' > "$SCRIPTS/.coralreef.env"` (or delete the file).
3. Tell the user coralreef mode is off. From here on, these rules no longer apply.
