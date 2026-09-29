# draftmesh-skills

An [Agent Skills](https://github.com/vercel-labs/skills) open-standard skill
for [DraftMesh](https://github.com/GordoLabs/draftmesh_storage) — a
git-backed, offline-first document collaboration server. Works with any
harness that supports the standard (Claude Code, Codex, Cursor, VS Code,
Gemini CLI, and others).

```sh
npm i -g draftmesh && draftmesh setup
```

Setup copies the bundled skill into your detected local harnesses. Then tell
your agent to use DraftMesh: read a scratch copy, edit it with local tools,
and submit the result through the CLI for attributed review.

For a hosted assistant without a shell, use the DraftMesh cloud connector
and the cloud-agent skill. Switching to cloud is explicit; the CLI never
fails over automatically. The local MCP mount remains an advanced door;
the published stdio shim is transitional compatibility only. Start with
`draftmesh setup` for local use.

**This repo is generated.** Its content is synced verbatim from
`skills/` in [GordoLabs/draftmesh_storage](https://github.com/GordoLabs/draftmesh_storage)
— send changes there, not here.
