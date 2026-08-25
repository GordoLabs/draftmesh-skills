---
name: draftmesh
description: Connect this agent to DraftMesh — a git-backed, offline-first server that keeps documents as real markdown files and exposes them over MCP — and use it to read, comment on, propose edits to, and hand back for review a shared markdown document. Use this when asked to "use DraftMesh", "review/comment on this doc with DraftMesh", "connect to DraftMesh", or any task that needs an agent to read or write a document that other people or agents also work in, even if no DraftMesh MCP connection exists yet.
---

# DraftMesh

DraftMesh keeps documents as plain markdown files on disk — nothing is trapped
in a proprietary format — and runs a small local daemon that exposes them over
MCP so any agent, on any harness, can read them, comment on them, propose
edits, and hand the result back to a person to review. This skill covers the
part the MCP connection can't teach you itself: getting connected in the first
place, and the propose → hand-off → react loop once you are.

**Local only.** This skill covers a personal/local DraftMesh instance (the
free tier). A hosted, multi-user DraftMesh with OAuth and organization-managed
agent credentials is a separate setup this skill does not cover.

## 1. Check whether you're already connected

Look for MCP tools named `list_workspaces` and `read_doc` (some harnesses
namespace tool names by server, e.g. as *mcp__draftmesh__read_doc* — that
still counts). If you have them, skip straight to §3 — you're connected. If
you don't, bootstrap below.

## 2. Bootstrap: install the MCP connection

Every harness ends up running the same command via `npx`, so there is no
package to install ahead of time — only a config entry (or, for Claude Code,
one CLI call) that tells your harness how to launch it. Pick your harness:

| Harness | How |
|---|---|
| Claude Code | Run: `claude mcp add draftmesh --env DRAFTMESH_AGENT_NAME="Claude Code" -- npx -y draftmesh-mcp` |
| Codex CLI | Append the TOML block below to `~/.codex/config.toml` |
| Claude Desktop, Cursor, Windsurf, Gemini CLI, or any other harness reading a generic MCP JSON config | Add the JSON entry below under the `mcpServers` key (Cursor: `~/.cursor/mcp.json`; Windsurf: `~/.codeium/windsurf/mcp_config.json`; Claude Desktop: `~/Library/Application Support/Claude/claude_desktop_config.json` on macOS, `%APPDATA%\Claude\claude_desktop_config.json` on Windows, `~/.config/Claude/claude_desktop_config.json` on Linux; consult the harness's own docs if its config file lives elsewhere) |
| VS Code / GitHub Copilot | Same JSON entry, but under a **`servers`** key (not `mcpServers`), in `<VS Code user dir>/User/mcp.json` — e.g. `~/Library/Application Support/Code/User/mcp.json` on macOS, `%APPDATA%\Code\User\mcp.json` on Windows, `~/.config/Code/User/mcp.json` on Linux |

JSON (Cursor, Windsurf, Claude Desktop, and other `mcpServers`-keyed
harnesses — VS Code uses the same entry shape one level under `servers`
instead):

```json
{
  "mcpServers": {
    "draftmesh": {
      "command": "npx",
      "args": ["-y", "draftmesh-mcp"],
      "env": { "DRAFTMESH_AGENT_NAME": "<a short label for this harness>" }
    }
  }
}
```

TOML block (Codex CLI):

```toml
[mcp_servers.draftmesh]
command = "npx"
args = ["-y", "draftmesh-mcp"]
env = { DRAFTMESH_AGENT_NAME = "Codex CLI" }
```

`DRAFTMESH_AGENT_NAME` is a free-tier attribution label, not a credential — it
just names you in DraftMesh's history and comments.

After editing a config file directly (i.e. every row except Claude Code),
**reload or restart the harness** so it picks up the new MCP server before you
try to use it.

**No daemon to start by hand.** The first tool call auto-launches a local
DraftMesh daemon if one isn't already running. On a machine that has never
run DraftMesh before, that first call can take a couple of minutes — it's
downloading native components — and answers with a **retryable** error
("DraftMesh is starting…") while it does. That is expected, not a failure:
retry the call rather than trying to install or start anything yourself.
Later sessions find the daemon already running and start instantly.

**Windows caveat.** Auto-launch on Windows needs a `draftmesh-mcp` release
built after the Windows spawn fix landed. If a tool call fails immediately on
Windows with a spawn-related error (not the retryable "starting" message
above), start the daemon by hand first — `npm i -g draftmesh` then run
`draftmesh` — and retry once it's up. Off Windows this doesn't apply.

## 3. Get oriented

Once connected, use `list_workspaces` → `list_docs` → `read_doc` to find and
read the document you were asked about. Nothing else to memorize here — the
MCP connection's own tool descriptions cover the mechanics (arguments,
pagination, version fields) in detail.

## 4. The loop: propose, hand off, react

This is the part that spans multiple tool calls and isn't obvious from any one
tool's description on its own:

1. **Prefer proposing over overwriting.** Use `suggest_edit` (or `add_comment`
   for a question or note) rather than `save_doc`, unless the person you're
   doing this for explicitly asked you to apply the change directly. DraftMesh
   is a human-reviewed loop — writes are attributed to you as an agent, and
   nothing you propose takes effect until a person accepts it.
2. **Hand back the `uiUrl`.** Write responses carry one whenever the daemon
   has a UI to link to — give it to the person you're doing this for so they
   can see exactly what you changed or asked, in context, rather than
   describing it secondhand.
3. **Don't poll silently for a decision.** If `watch_reviews` is available on
   your connection, use it to wait for a human's decision on what you
   proposed. Otherwise, use `query_markers` to check current status when
   asked. Either way, react to the outcome — reply, revise, or stop — rather
   than treating the hand-off as the end of the task.

## Tools this skill uses

`list_workspaces`, `list_docs`, `read_doc`, `save_doc`, `suggest_edit`,
`add_comment`, `watch_reviews`, `query_markers` — all provided by the
DraftMesh MCP connection once installed (§2). Availability can vary by host:
if a tool named here isn't present on your connection, treat that as a
capability gate rather than an error.

## Troubleshooting

- **No DraftMesh tools appear after installing.** Your harness hasn't reloaded
  its MCP config yet — restart it.
- **"DraftMesh is starting…"** Expected on a fresh machine (§2) — retryable,
  can take a couple of minutes. Retry the call; don't install or start
  anything yourself.
- **A tool call fails outright** (not the retryable "starting" message
  above). See the Windows caveat in §2 if you're on Windows, or start the
  daemon by hand: `npm i -g draftmesh && draftmesh`.
- **A write fails with a stale-version or conflict error.** Someone else
  changed the document since you read it. Re-read it with `read_doc` to get
  the current version, then reapply your change — nothing is lost.
- **You don't see a workspace you expected.** Ask the person you're working
  with for the workspace it lives in; `list_workspaces` only shows ones this
  local daemon already knows about.
