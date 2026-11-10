# MyCodery — private instance repo

This is **not Codery**. This repo holds what belongs to *this* deployment of
[Codery](https://github.com/CoderyOSS/Codery): instance inventory, instance
docs, and deployment config records. Codery itself lives in a separate
checkout (`/workspace/Codery` — the public clone) and is never edited here.

## Push rules

- Push **only** to `rm-rf-etc/MyCodery` (this repo's origin).
- Codery changes go to the Codery checkout → `github-push <branch> CoderyOSS/Codery`.
- **Verify every push against the remote URL, never a local remote:**
  `git ls-remote https://github.com/rm-rf-etc/MyCodery.git <ref>`
- Known hazard (until the paseo image update): the current `github-push`
  rewrites `origin` when given an explicit repo. If origin ever looks wrong,
  fix it with `git remote set-url origin <url>` — and prefer no-arg pushes in
  each checkout so it never comes up.

## Layout

| Path | Contents |
|---|---|
| `INSTANCE.md` | the running deployment: stacks, vhosts, extra env keys, known debt |
| `instances/` | deployment config records (e.g. `zoraxy-vhosts.list`) |
| `docs/` | instance-specific history: migration runbook, design specs, incident records |

Codery product docs (README/ARCHITECTURE/RUNBOOK/AGENTS/SETUP) live in the
Codery repo, not here.
