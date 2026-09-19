---
name: draftmesh-cloud-agent
description: Become a wakeable reviewer for a HOSTED DraftMesh workspace — connect to the cloud MCP door over OAuth (no local install, no daemon), announce yourself, then wait to be handed documents when a reviewer clicks "Send to agent". Use this when asked to "be a DraftMesh agent for our team", "receive reviews from hosted/cloud DraftMesh", "wait for DraftMesh to send me work", or any task where a person on a shared DraftMesh will dispatch documents to you and you have no DraftMesh installed locally.
---

# DraftMesh cloud agent (wakeable receiver)

This skill is for an assistant that connects to a **hosted, multi-user
DraftMesh** over its cloud MCP door — no local install, no daemon on your
machine. It covers the one thing that connection can't teach you: how to make
yourself a target a reviewer can **Send to agent**, and how to wait to be
handed work.

**Not the local free tier.** If you run your own personal DraftMesh on this
machine, use the `draftmesh` skill instead — there you watch for reviews with
`watch_reviews`, a different tool. This skill is the cloud-only counterpart:
same "someone sends you a document, you act, you hand it back" loop, reached
without any local process.

## 1. Check whether you're already connected

Look for MCP tools named `list_workspaces` and `read_doc` (some harnesses
namespace them, e.g. *mcp__draftmesh__read_doc* — that still counts). If you
also see `connect_receiver`, you're connected to the hosted door and can skip
to §3. If you have no DraftMesh tools at all, connect below.

## 2. Connect to the hosted door

The hosted door is an OAuth-authenticated MCP endpoint your team's DraftMesh
publishes (its URL ends in `/mcp` — ask whoever runs your DraftMesh for the
exact address if you don't have it). Add it as a **remote / HTTP MCP server**
in your harness's MCP config and complete the OAuth sign-in it prompts for —
you act with your own DraftMesh permissions once signed in. Follow your
harness's own documentation for adding a remote MCP server and for how it runs
the OAuth flow; the specifics differ per harness and aren't something this
skill can script.

After connecting, **reload or restart the harness** if it doesn't pick the
server up immediately, then confirm the tools appear (§1).

If `connect_receiver` is absent even though other DraftMesh tools are present,
your session isn't the OAuth-on-behalf posture this needs (for example you
connected with a standing agent credential instead) — a standing agent is
already dispatchable and doesn't use this loop. Ask whoever runs your DraftMesh
which posture your connection uses.

## 3. Announce yourself once

Call `connect_receiver` a single time. It registers you so you appear in each
document's agent picker as a **Send to agent** target, and returns the name
you'll show up as (pass a `displayName` if you want to choose it). You hold no
standing credential and nothing is pushed to you — each handoff you later
receive carries its own task-scoped access, and only for the one document.

Tell the person you're working with that you're now connected and pickable, so
they know they can send you a document.

## 4. The loop: wait, act, hand back

1. **Wait for work.** Call `receive_review`. It blocks until a reviewer sends
   you a document, then returns the handoff(s). Treat this as a background wait
   and call it again whenever it returns.
2. **A handoff is yours the moment you receive it.** Each handoff's
   `payload.instructions` is the review to do and `payload.document` names the
   workspace and path. Read the document (`read_doc`); when it cites an image
   (`![…](path.png)`), `read_asset` with that `docPath` shows it to you — your
   access is per-document, so the image is served only if the document cites
   it. Then do what was asked —
   prefer `suggest_edit` and `add_comment` over `save_doc`: DraftMesh is a
   human-reviewed loop, and nothing you propose takes effect until a person
   accepts it.
3. **Finish the task.** When you've posted your comments and suggestions, call
   `complete_task` — exactly once, last — to report the work done. Completing
   accepts nothing; every suggestion still waits for a human. Hand back the
   `uiUrl` from your write responses so the person can see your work in context.
4. **Keep watching.** After each handoff, call `receive_review` again. When it
   returns with nothing (a `timedOut` result), that just means the wait
   expired — call it again to keep waiting.

<!-- generated:tools:start -->
## Tool reference

DraftMesh can expose these tools through this door; availability on a connection depends on its capabilities:

- `add_comment` — Comment on a document
- `answer_question` — Answer a question marker
- `ask_question` — Ask a question on a passage
- `complete_task` — Mark this task complete
- `connect_receiver` — Become a wakeable DraftMesh reviewer
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
- `receive_review` — Wait for the next review handoff
- `relate_workspace` — Relate two workspaces
- `reply_to_marker` — Reply to a marker
- `request_sign_off` — Ask a person to sign off on a document
- `request_workspace_link` — Ask another workspace owner for a link
- `save_doc` — Save a document
- `search_docs` — Search a workspace's documents
- `set_marker_status` — Resolve or reopen a marker
- `suggest_edit` — Suggest an edit
- `unlink_workspace` — Remove a part-of link between workspaces
- `unrelate_workspace` — Remove a related-workspace link
- `withdraw_workspace_link_request` — Withdraw your pending workspace link request
<!-- generated:tools:end -->

## Troubleshooting

- **No DraftMesh tools appear after connecting.** Your harness hasn't reloaded
  its MCP config yet — restart it, then re-check.
- **`connect_receiver` is missing but other tools are present.** Your session
  isn't the OAuth-on-behalf posture (see §2) — this loop needs it.
- **`receive_review` keeps returning empty.** That's normal — it holds for a
  while and returns `timedOut` when nothing arrived. Call it again. You only
  get a handoff after a reviewer actually clicks Send to agent and picks you.
- **A write fails with a stale-version or conflict error.** Someone changed the
  document since you read it — re-read it with `read_doc` and reapply; nothing
  is lost.
- **You were handed a document but can't read it after finishing.** Completing
  a task ends its access — do all your reading and writing before
  `complete_task`.
