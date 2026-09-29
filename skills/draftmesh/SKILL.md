---
name: draftmesh
description: Use DraftMesh to read shared documents, comment, propose edits, and hand work back for review. For a local shell, install the CLI and skill with draftmesh setup, inspect files on disk, and submit attributed changes through the CLI. Use this when asked to use DraftMesh, review a document with DraftMesh, or collaborate on documents other people or agents also edit.
---

# DraftMesh

DraftMesh keeps shared documents as real files and records comments, suggestions
and versions with their authors. Use the local CLI for an attributed
read → edit a copy → propose → hand-off → react loop.

## Choose the connection

- **Local files and a shell:** use your file and search tools for focused
  inspection, then use the DraftMesh CLI for versions, annotations and writes.
  Fetch an editable scratch copy as shown below so the CLI records its base.
- **A shell, but no registered workspace yet:** list workspaces with
  `draftmesh ws list`. Register the folder the person named with
  `draftmesh ws register <absolute-folder>`. This adopts the folder in place;
  it does not copy its files or turn sync on. Use `draftmesh ws create <name>`
  only when the person wants a new, empty workspace.
- **A hosted session without a local shell:** choose the `draftmesh-cloud`
  connector and use the `draftmesh-cloud-agent` skill. Claude Desktop's
  shell-less connector flow belongs here.

Cloud is also an explicit backup for a synced workspace: the person or harness
chooses that connector for the session. The CLI never switches to cloud
automatically after a local failure. Confirm the cloud copy is current before
continuing there, and wait for sync before resuming from local files.

`draftmesh status` reports, for every cloud-bound folder, when its sync last
reached the cloud. Read it before you trust local files against work someone
may have done elsewhere, and after handing work back if the person is waiting
on it there. When a folder is behind, ask the person whether to run
`draftmesh sync`, which syncs the workspace you are standing in now and reports
whether it ended current; it is a device command, so run it only when they ask.
Conflicts it reports are resolved by a person, never by you.

Use `draftmesh` for the commands below when it is installed on `PATH`. If the
shell reports that `draftmesh` is not found, run the same command with
`npx -y draftmesh` instead (for example, `npx -y draftmesh ws list`). This is
only an executable fallback; do not switch commands after a daemon, permission,
or API error.

## Setup

```sh
npx -y draftmesh setup
```

Setup shows its planned writes, asks for confirmation, copies the bundled skill
into detected harnesses and finishes with diagnostics. It does not add an MCP
server or require the daemon to be running. Install permanently with
`npm i -g draftmesh` if preferred. Start a new harness session if needed to
load the skill. Use `draftmesh doctor` when setup or the local connection needs
attention.

Keep the same agent name throughout a task: pass `--agent "Your agent name"`
on CLI operations, or set `DRAFTMESH_AGENT_NAME` in the task's environment.
The session and read ledger belong to that identity; changing names midway
does not transfer another agent's work. Never read or copy daemon/session
bearer tokens into commands or messages.

## Read, edit a copy, then propose

Start with `draftmesh ws list`, then `draftmesh ws map --ws <workspace-id>`.
The map gives document titles and summaries; a document path gives its outline.
Use `draftmesh doc list --ws <workspace-id>` for a plain listing. Include `--ws`
when the workspace is ambiguous; otherwise the CLI can resolve it from the
working directory. Use a command's `--help` for its optional flags.

Orient with the map before opening document bodies. Expand the relevant folder,
read any agent guidance the map names, then inspect the selected document's
outline. For a focused read, copy its heading slug exactly:

```sh
draftmesh ws map specs --ws <workspace-id>
draftmesh ws map specs/design.md --ws <workspace-id>
draftmesh doc read specs/design.md --ws <workspace-id> --section decision
```

A section includes its nested headings. Its marker offsets refer to the returned
content; `section.start` and `section.end` locate it in the whole document.
A missing slug means read the outline again. If the map reports `partial: true`,
retry before concluding a document is absent. Section reads are for inspection;
read a whole-document scratch copy before proposing or saving changes.

Use output controls when you need only part of a response:

```sh
draftmesh doc read specs/design.md --ws <workspace-id> --section decision --json --fields content,versionId,section.start
draftmesh ws map specs/design.md --ws <workspace-id> --format json --json --fields document.headings
draftmesh doc read specs/design.md --ws <workspace-id> --max-chars 4000
draftmesh doc read specs/design.md --ws <workspace-id> --json --fields content --max-chars 4000
```

`--fields` requires `--json` and takes comma-separated dot paths. The output
keeps nested objects, omits missing fields, and projects array items individually;
selecting a parent includes its whole value. Array indices and wildcards are not
supported. A map also needs `--format json` for field selection.
`--max-chars` limits the preview in Unicode code points, after field selection.
Truncated text ends with the original UTF-8 byte count. Truncated JSON is a valid
`{truncated: true, originalBytes, preview}` envelope, where `preview` is a prefix
of the serialized result, not a complete document or JSON value. The envelope or
text suffix is additional to the preview limit. Read again with a narrower
section or a larger limit when you need the omitted content.
Output controls do not shorten errors. `--fields` and `--max-chars` cannot be
used with `--out` or `--from`; `--json` is also unavailable with `--out` and
`doc suggest --from`. Scratch exports always contain the exact whole document.

Choose a scratch directory outside the workspace. Replace the example paths
and workspace id with the requested document and your scratch file:

```sh
draftmesh doc read notes.md --ws <workspace-id> --out <scratch-file>
```

This writes the exact document bytes, including markers, to the scratch file
and records the version in your read ledger. The command prints metadata
instead of the document body. Inspect only the passages you need with local
tools, then edit that copy using your editor or patch tool. Keep whole-document
output out of the conversation: the scratch file is the source for the edit,
not a prompt to print its body.

Never edit a workspace file in place: the filesystem watcher would attribute
that edit to the person. Submit your scratch copy through the CLI so DraftMesh
stamps your session identity. Preserve every `<!-- @draftmesh… -->` marker;
use marker commands to reply or change status rather than editing marker bytes.

Propose the changes:

```sh
draftmesh doc suggest notes.md --ws <workspace-id> --from <scratch-file>
```

The CLI compares the copy with the current document and submits one suggestion
per changed hunk. Add `--dry-run` to inspect those hunks without writing.
If the document changed while you worked, read a fresh copy and reapply only
your intended edits before submitting; do not propose a stale copy wholesale.
After any marker operation, read the canonical document again before editing.

When the person asked you to apply the edit directly and **Agents may save
directly** is enabled for that workspace, use:

```sh
draftmesh doc save notes.md --ws <workspace-id> --from <scratch-file>
```

The save uses the recorded read version. `--base <version-id>` may supply a
known base explicitly. A missing or stale base is a reason to read again and
reapply the intended change, never to drop the version gate.

## Hand off and react

1. Return the `uiUrl` and version or suggestion ids from the response so the
   person can inspect the result in DraftMesh.
2. Use `draftmesh watch` under the same stable `--agent` identity to wait for
   the next review handoff. Use only your own session's returned wait command
   or agent id; supply `--agent-id` only when it was returned for your session.
3. Read the delivered task and any addressed marker. If you need only the
   current base for a marker reply or answer, run
   `draftmesh doc read <path> --ws <workspace-id> --json --fields versionId`;
   do not request `content` just to obtain a version. Reply with
   `draftmesh marker reply` or answer a question with `draftmesh marker answer`;
   use each command's `--help` for required version and text arguments.
   Long replies can come from `--text-from <file>` instead of inline text.
4. Revise, explain, or stop according to the person's response. When finished,
   call `draftmesh task complete <task-id> --summary "What you completed"`
   with the handoff's task id. Completing a task does not accept its suggestions.

For recurring work, the person can create a standing rule with **Connect an
agent…** in the Agents panel. Use that handoff mechanism instead of repeatedly
polling documents for changes.

## What your session can do

A scoped local assistant session can read authorized documents, comment,
ask and answer questions, propose edits, request human sign-off, and complete
its own tasks. Direct saves require the workspace's save switch; with it on,
`draftmesh doc save <dir>/images/<name>.png --from <file>` uploads an image
(`.png`/`.jpg`/`.gif`/`.webp`, never `.svg`) to embed with a relative
`![alt](images/<name>.png)` link. Accepting or
rejecting suggestions through `draftmesh decide` requires the separate
workspace permission and the person's instruction.

Approvals, approval answers, rollback, turning a workspace's sync on or off,
account changes and cloud connection changes are not assistant operations. Ask the person to use the UI
or their own CLI session for those actions. `draftmesh cloud connect` in the
reference below is for the person; do not invoke it to work around a denial.
These session permissions are an API boundary, not an operating-system sandbox.

## Etiquette

Prefer suggestions unless direct application was requested. Register only
folders the person named. Preserve their edits and comments, keep replies
focused, and hand back the actual result link. A permission denial is a
boundary to explain, not an invitation to try another identity.
<!-- generated:tools:start -->
## Tool reference

Local CLI commands and their required arguments. Use a command's --help for optional flags and file-backed forms; availability depends on session permissions:

- `draftmesh marker add <path> --ws <workspaceId> --base-version-id <baseVersionId> --anchor-json <anchor> --text <text>` — Comment on a document
- `draftmesh marker answer <path> <markerId> --ws <workspaceId> --base-version-id <baseVersionId> --text <text>` — Answer a question marker
- `draftmesh marker ask <path> --ws <workspaceId> --base-version-id <baseVersionId> --anchor-json <anchor> --prompt <prompt>` — Ask a question on a passage
- `draftmesh task complete <taskId>` — Mark this task complete
- `draftmesh cloud connect` — Sign this DraftMesh in to DraftMesh cloud
- `draftmesh ws create <name>` — Create a workspace
- `draftmesh decide <path> <markerId> <action> --ws <workspaceId> --base-version-id <baseVersionId>` — Accept or reject a suggestion
- `draftmesh doc describe <path> --ws <workspaceId> --content-hash <contentHash> --summary <summary>` — Leave a better one-line summary for a document
- `draftmesh marker questions <path> --ws <workspaceId>` — Find a document's open questions
- `draftmesh diagnostic health` — Read the DraftMesh service's health
- `draftmesh ws link <partOf> --ws <workspaceId>` — Make one workspace part of another
- `draftmesh doc list --ws <workspaceId>` — List documents
- `draftmesh ws list` — List workspaces
- `draftmesh marker query --ws <workspaceId>` — Find markers across a workspace
- `draftmesh doc asset <path> --ws <workspaceId>` — Read an image asset
- `draftmesh diagnostic read` — Read recent diagnostic records
- `draftmesh doc read <path> --ws <workspaceId>` — Read a document
- `draftmesh doc version <path> <versionId> --ws <workspaceId>` — Read an earlier version of a document
- `draftmesh doc history <path> --ws <workspaceId>` — Read a document's version history
- `draftmesh ws map [<path>] --ws <workspaceId>` — Read a workspace's map (table of contents)
- `draftmesh ws register <path>` — Register an existing folder as a workspace
- `draftmesh ws relate <relatedTo> --ws <workspaceId>` — Relate two workspaces
- `draftmesh marker reply <path> <markerId> --ws <workspaceId> --base-version-id <baseVersionId> --text <text>` — Reply to a marker
- `draftmesh marker sign-off <path> --ws <workspaceId> --base-version-id <baseVersionId> --assigned-to-json <assignedTo>` — Ask a person to sign off on a document
- `draftmesh doc save <path> --ws <workspaceId>` — Save a document
- `draftmesh doc search <query> --ws <workspaceId> --mode <mode>` — Search a workspace's documents
- `draftmesh marker status <path> <markerId> <status> --ws <workspaceId> --base-version-id <baseVersionId>` — Resolve or reopen a marker
- `draftmesh doc suggest <path> --ws <workspaceId> --base-version-id <baseVersionId> --anchor-json <anchor> --op <op>` — Suggest an edit
- `draftmesh ws unlink <partOf> --ws <workspaceId>` — Remove a part-of link between workspaces
- `draftmesh ws unrelate <relatedTo> --ws <workspaceId>` — Remove a related-workspace link
- `draftmesh watch` — Watch for review handoffs from DraftMesh
<!-- generated:tools:end -->

## Troubleshooting

- **Command, daemon, skill, or sandbox trouble:** run `draftmesh doctor` and
  follow the reported fix. `draftmesh setup` refreshes skill copies and the
  Claude Code loopback allowlist. If a newer CLI finds an older daemon,
  restart the daemon as instructed rather than repeatedly reinstalling.
- **"Semantic search is not installed on this computer":** the npm package
  leaves the semantic runtime out. The person runs `draftmesh setup --semantic`
  once (about 500 MB, CPU only); until then use `--mode text` or name search,
  which are unaffected.
- **No workspace found:** run `draftmesh ws list`, specify `--ws`, or register
  the exact folder the person named. A missing cloud workspace may require
  the person to sign in and enable sync; do not handle their credentials.
- **A version conflict or missing base:** read a fresh scratch copy, preserve
  the other edits, reapply yours and retry with the recorded base.
- **A save is forbidden:** propose with `draftmesh doc suggest --from` or
  ask the person to enable direct saves for that workspace.
- **An operation is not permitted for this session:** explain the boundary
  and let the person perform the owner action. Never substitute the owner
  bearer or silently change to a cloud identity.
