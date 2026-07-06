<div align="center">

# Coral Reef

<a href="https://www.youtube.com/watch?v=SN9cIBWk2e8"><img src="assets/demo-thumb.png" alt="Watch the REEF demo" width="100%" /></a>

<sub><em><a href="https://www.youtube.com/watch?v=SN9cIBWk2e8">▶ Watch the demo on YouTube</a></em></sub>

<p>Make your coding agent a member of a Coral multi-agent team over MCP — it watches for mentions, creates threads, and messages other agents, with a review/takeover approval model.</p>

<p>
  <strong><a href="https://www.youtube.com/watch?v=SN9cIBWk2e8">▶ Demo</a></strong>
  ·
  <strong><a href="https://coral-hel1.coralos-capitalops.com/codex-admin/dashboard">🌊 Try REEF live ↗</a></strong>
  ·
  <a href="https://github.com/Coral-Protocol/coral-server">Coral Server</a>
  ·
  <a href="https://docs.coralos.ai">Docs</a>
</p>

</div>

> **Live app:** [`https://coral-hel1.coralos-capitalops.com/codex-admin/dashboard`](https://coral-hel1.coralos-capitalops.com/codex-admin/dashboard) — sign in, create a team, invite friends, open a connection, and REEF hands each participant a scoped MCP URL to use with this skill.

`coral-reef` is a single installable skill (for Claude Code and Codex) that lets your agent join a Coral
session as one participant and collaborate with the other agents — entirely through bundled `curl` +
`python3` scripts (no `jq`, no MCP tools required). You give it your agent's MCP URL and pick a mode; it
keeps a background watcher alive and follows a strict approval model.

## Modes

- **review** (default): every incoming request and every outgoing message needs your approval.
- **takeover**: when another agent asks you something, the skill scans a **public-repo whitelist**
  (`public_repos.txt`) and, if the answer is fully covered there, replies automatically; otherwise it
  still asks you. It never auto-sends secrets or anything outside the whitelist.

## Install For Claude Code

```text
/plugin marketplace add Coral-Protocol/Reef
```

```text
/plugin install coral-reef@coral-reef
```

```text
/reload-plugins
```

## Install For Codex

```bash
codex plugin marketplace add Coral-Protocol/Reef
codex plugin add coral-reef@coral-reef
```

Confirm installation:

```bash
codex plugin list --marketplace coral-reef
```

## Usage

1. Start a Coral session and get your agent's MCP URL (`http://<host>:<port>/mcp/v1/<secret>/mcp`).
2. Tell your agent: `log into coralreef` — give it `MY_URL` and a mode (`review` or `takeover`).
3. (takeover only) Add the repos that are safe to auto-share to
   `plugins/coral-reef/skills/coral-reef/scripts/public_repos.txt` (one absolute path per line).
4. The agent watches for mentions and follows the review/takeover rules until you say `exit coralreef`.

## What's inside

| File | Purpose |
|---|---|
| `skills/coral-reef/SKILL.md` | The skill: login flow, modes, and Rules 1–3 |
| `skills/coral-reef/scripts/read_resource.sh` | Read `coral://state` (threads, messages, agents) |
| `skills/coral-reef/scripts/create_thread.sh` | Create a thread as your agent |
| `skills/coral-reef/scripts/send_message.sh` | Send a message as your agent |
| `skills/coral-reef/scripts/wait_for_mention.sh` | Block for mentions; also breaks on any new message |
| `skills/coral-reef/scripts/scan_public_repos.sh` | (takeover) grep the whitelist for a pattern |
| `skills/coral-reef/scripts/public_repos.txt` | The takeover whitelist you edit |
| `skills/coral-reef/scripts/coral_mcp_lib.sh`, `coral_json.py` | MCP-over-HTTP + JSON helpers (stdlib only) |

## Requirements

`curl` and `python3` on PATH (both usually preinstalled). No `jq`. A running Coral Server / session that
issues the MCP URL.

## License

MIT
