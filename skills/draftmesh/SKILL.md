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
still counts). If you have them, skip straight to §3 (or §4, if the workspace you need is
already registered) — you're connected. If
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

## 3. Get the person's folder in

Before you can read anything, this DraftMesh has to know about the folder the
work lives in. If `list_workspaces` comes back empty, or the folder the person
means isn't in the list, there are two moves and they are not the same one:

- **The folder already exists on this machine** — their own project, checkout
  or notes directory. `register_workspace` with its path (an absolute path, or
  one starting `~/`) registers it *where it is*: nothing is copied, nothing is
  moved, and registering never turns sync on by itself — a folder this
  DraftMesh has not synced before stays local until a person turns sync on, and
  one it synced before resumes the setting they already chose. This is the same act as
  **Register a folder…** in DraftMesh's own workspace picker. It is offered
  only by a DraftMesh running on the person's machine, and it is idempotent —
  registering a folder that is already a workspace answers the workspace it
  already is.
- **There is nothing yet and they want a fresh place for it** —
  `create_workspace` with a name makes a NEW, empty workspace under
  `~/DraftMesh`.

**Never copy someone's files into a new workspace.** If the documents already
exist somewhere, register that folder; creating a workspace and writing copies
of their files into it leaves them with two divergent sets and DraftMesh
watching the wrong one. Register a folder only when the person named it — don't
go looking for folders to add.

## 4. Get oriented

Once connected, use `list_workspaces` → `read_workspace_map` → `read_doc` to
find and read the document you were asked about. The map is the cheap first
look at a workspace: every document's title and a one-line summary, grouped by
folder, built by DraftMesh from the documents themselves (nobody has to write
an index for it). Pass `path` to open a folder, or a document's path for its
heading outline — then `read_doc` with `section` set to one of those slugs
reads just that part of a long document. `list_docs` is the plain path
listing if you need it. When the document cites an image (`![…](path.png)`),
`read_asset` shows it to you — pass `docPath` when your access is
per-document. Nothing else to memorize here — the MCP connection's own tool
descriptions cover the mechanics (arguments, pagination, version fields) in
detail.

## 5. The loop: propose, hand off, react

This is the part that spans multiple tool calls and isn't obvious from any one
tool's description on its own:

1. **Prefer proposing over overwriting.** Use `suggest_edit` (or `add_comment`
   for a question or note) rather than `save_doc`, unless the person you're
   doing this for explicitly asked you to apply the change directly. DraftMesh
   is a human-reviewed loop — writes are attributed to you as an agent, and
   nothing you propose takes effect until a person accepts it — unless they
   have switched **Assistants may accept and reject suggestions in this
   workspace** on for that workspace, or — where you are connected to a
   person's own account on the hosted service — they have switched on
   **Assistants connected to my account may accept and reject suggestions as
   me** in their Settings, in which case `decide_suggestion` is
   how you close a suggestion, and the accept is recorded as yours — as
   the assistant, in history and in the audit trail, with the person named as
   the one you acted on behalf of. Use it
   only when they asked you to manage suggestions; proposing is still the
   default. If it answers `forbidden`, the permission is missing — on your own
   machine the switch is off; on a hosted door the document's owner has not
   granted you decide, or the person whose account you are connected to has
   not switched on their own setting — and that is the product working, not an
   error to retry: reply on the suggestion with `reply_to_marker` or ask the
   person to turn it on.
2. **Hand back the `uiUrl`.** Write responses carry one whenever the daemon
   has a UI to link to — give it to the person you're doing this for so they
   can see exactly what you changed or asked, in context, rather than
   describing it secondhand.
3. **Don't poll silently for a decision.** If `watch_reviews` is available on
   your connection, use it to wait for a human's decision on what you
   proposed. Otherwise, use `query_markers` to check current status when
   asked. Either way, react to the outcome — reply, revise, or stop — rather
   than treating the hand-off as the end of the task. (The same call is also
   how you register yourself — see 4.)
4. **`watch_reviews` is also how you introduce yourself.** Calling it
   registers *this session* under the agent name you declared on the
   connection, and the person then sees you by that name in DraftMesh's
   **Agents** panel and in the **Send to agent** picker — so they can hand you
   a document deliberately instead of shouting into a shared queue. Pass
   `workspaces` (a list of workspace ids) when you only watch some of them;
   omit it to be offered for every workspace this DraftMesh knows. It returns
   a wait command addressed to *your* session — use that one, not a remembered
   one, so another agent's work is never drained by you. If you are
   reconnecting under a *different* agent name than last time, pass the
   `agentId` the earlier call answered as `previousAgentId`: your standing
   connections and your mailbox then follow you to the new name instead of
   staying pinned to a row nobody drains.
5. **Close the loop on a hand-off.** When you finish work that arrived this
   way, call `complete_task({ taskId, summary })` with the `taskId` the
   hand-off carried. That is what moves the request from *Delivered* to
   *Completed* in the panel and shows the person your summary; skipping it
   leaves them watching a request that looks stuck.
6. **Standing work is a connection, not a loop.** On a DraftMesh running on
   the person's own machine, they can use **Connect an agent…** in the Agents
   panel to create a standing rule that wakes you whenever a document changes
   or gains a comment, suggestion or question, with instructions attached.
   Suggest it when they keep asking you for the same pass; never poll for
   changes yourself. They can also **Assign** a comment or question to you:
   the hand-off then carries `payload.marker.id` — read that marker and act on
   it (`answer_question` if it is a question, `reply_to_marker` otherwise),
   then `complete_task`.

<!-- generated:tools:start -->
## Tool reference

DraftMesh can expose these tools through this door; availability on a connection depends on its capabilities:

- `add_comment` — Comment on a document
- `answer_question` — Answer a question marker
- `ask_question` — Ask a question on a passage
- `complete_task` — Mark this task complete
- `connect_cloud` — Sign this DraftMesh in to DraftMesh cloud
- `create_workspace` — Create a workspace
- `decide_suggestion` — Accept or reject a suggestion
- `describe_doc` — Leave a better one-line summary for a document
- `find_questions` — Find a document's open questions
- `get_health` — Read the DraftMesh service's health
- `link_workspace` — Make one workspace part of another
- `list_docs` — List documents
- `list_workspaces` — List workspaces
- `query_markers` — Find markers across a workspace
- `read_asset` — Read an image asset
- `read_diagnostics` — Read recent diagnostic records
- `read_doc` — Read a document
- `read_doc_version` — Read an earlier version of a document
- `read_history` — Read a document's version history
- `read_workspace_map` — Read a workspace's map (table of contents)
- `register_workspace` — Register an existing folder as a workspace
- `relate_workspace` — Relate two workspaces
- `reply_to_marker` — Reply to a marker
- `request_sign_off` — Ask a person to sign off on a document
- `save_doc` — Save a document
- `search_docs` — Search a workspace's documents
- `set_marker_status` — Resolve or reopen a marker
- `suggest_edit` — Suggest an edit
- `unlink_workspace` — Remove a part-of link between workspaces
- `unrelate_workspace` — Remove a related-workspace link
- `watch_reviews` — Watch for review handoffs from DraftMesh
<!-- generated:tools:end -->

## Troubleshooting

- **No DraftMesh tools appear after installing.** Your harness hasn't reloaded
  its MCP config yet — restart it.
- **"DraftMesh is starting…"** Expected on a fresh machine (§2) — retryable,
  can take a couple of minutes. Retry the call; don't install or start
  anything yourself.
- **A tool call fails outright** (not the retryable "starting" message
  above). Start the daemon by hand — `npm i -g draftmesh && draftmesh` — and
  retry once it's up.
- **A write fails with a stale-version or conflict error.** Someone else
  changed the document since you read it. Re-read it with `read_doc` to get
  the current version, then reapply your change — nothing is lost.
- **You don't see a workspace you expected.** If it lives in the person's
  DraftMesh cloud account, this DraftMesh may not be signed in: call
  `connect_cloud`, show the person the URL and code it returns, and call it
  again after `pollAfterSeconds` until its status reads signed_in — you never
  handle a credential, only the one-time code they approve in their browser.
  If instead it is a folder sitting on this machine that nobody has registered
  yet, that is the other case: `register_workspace` with its path adds it in
  place (§3). Otherwise ask the person which workspace it lives in;
  `list_workspaces` only shows ones this local daemon already knows about.
- **A tool answers `unauthorized`.** Same remedy: `connect_cloud`, then retry.
