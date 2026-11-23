# session-docs-git-wrap-up

An Agent Skill (`v1.2.0`) for ending a coding session cleanly across Cursor, Antigravity, Codex, and Agent Skills compatible harnesses. It reconciles every doc in the repo with the current code, isolates concurrent-session edits, runs every hook stage plus unit tests without skipping, commits by allowlist, and pushes to a private remote when creating a new repo.

It is language-, host-, and project-agnostic. It works on Python, Node, Go, Rust, Ruby, monorepos, and federated multi-repo workspaces across GitHub, GitLab, Codeberg, and self-hosted remotes.

## What it does

Eight phases the agent runs in order:

| Phase | What it does |
| --- | --- |
| **0. Detect toolchain** | Reads manifests (`pyproject.toml`, `package.json` + lockfile, `go.mod`, `Cargo.toml`, `Makefile`), checks actual hook coverage, inspects nested/federated subrepos, and detects project-local governance or deploy gates (`AGENTS.md`, `GEMINI.md`, decision ledgers) without hardcoding project paths or hosts. |
| **1. Identify changes & isolate concurrent edits** | Partitions `git status --short` and `git diff` into session-owned files, unrelated concurrent files (never touched), and mixed-hunk shared files (staged per hunk or coordinated with the user). Stops early if nothing changed. |
| **2. Inventory every doc** | Lists `README`, `CHANGELOG`/`HISTORY`/`RELEASES`, `CONTRIBUTING`, `SECURITY`, `docs/`, ADRs, RFCs, decision/veto ledgers and session journals when present, monorepo per-package READMEs, agent rules (`AGENTS.md`, `GEMINI.md`, `CLAUDE.md`, `.cursor/rules/`), domain reports, `.env.example`, README badges, and layout docstrings. |
| **3. Reconcile each doc** | Checks twelve failure modes: stale paths, stale commands, rotting layout trees, capability tables, CHANGELOG/journal entries, cross-references, code snippets, configuration docs, anchor/link integrity, terminology, generated-docs rebuilds, and monorepo/federated sync. |
| **4. Tests and hooks** | Runs the full hooked workflow (pre-commit, husky, lefthook, raw `.git/hooks/`, or direct scripts) with no `--no-verify` or `SKIP=…`. Explicitly closes the lint-only hook gap by running unit tests whenever hooks only perform static checks, and checks new macOS scripts for UTF-16 encoding. |
| **5. Commit** | Stages strictly by explicit path allowlist (or hunk staging for mixed files) rather than blind `git add -A`. Commits touched nested subrepos first, matches the repo's commit convention, and writes clean heredoc messages. |
| **6. Push, private remote if needed** | Pushes to `origin`, or creates a **private** remote on the matching host and verifies visibility. Reports pre-existing public repos clearly instead of silently flipping them. |
| **6B. Deploy gate & live activation (workspace-conditional)** | Applies only when upgrading a live deployed service or fleet node. Runs any repository deploy gate first and treats a refusal (`STAGED_PENDING_GATE`) as a valid completion state; when allowed, verifies offline-safe sync, decoupled process-tuple reload, runtime provenance, config/cache parity, and first-tick liveness across a three-state receipt (`ACTIVATED_IN_RAM`, `STAGED_PENDING_UNLOCK`, `STAGED_PENDING_GATE`). |
| **7. Final verification** | Confirms concurrent-aware git cleanliness, latest commit and subrepo state, remote sync, visibility, doc inventory reconciliation, project-defined architecture linters and hygiene checks, hooks plus unit tests, and honest activation status when in scope. |

Full text: [`skill/SKILL.md`](skill/SKILL.md).

## Install

Clone once into a canonical directory and symlink into each agent harness so every tool shares one source of truth:

```bash
git clone https://github.com/mynameisi/session-docs-git-wrap-up.git \
  ~/.cursor/skills/session-docs-git-wrap-up
ln -sfn skill/SKILL.md ~/.cursor/skills/session-docs-git-wrap-up/SKILL.md

# Share across Antigravity / Agents and Codex without drifting copies
ln -sfn ~/.cursor/skills/session-docs-git-wrap-up ~/.agents/skills/session-docs-git-wrap-up
ln -sfn ~/.cursor/skills/session-docs-git-wrap-up ~/.codex/skills/session-docs-git-wrap-up
```

> **Do not install into `~/.cursor/skills-cursor/`.** That directory is reserved for Cursor's built-in skills and is managed automatically.

### Project install (commit it with the repo)

```bash
mkdir -p .agents/skills/session-docs-git-wrap-up
curl -fsSL https://raw.githubusercontent.com/mynameisi/session-docs-git-wrap-up/main/skill/SKILL.md \
  > .agents/skills/session-docs-git-wrap-up/SKILL.md
git add .agents/skills/session-docs-git-wrap-up/SKILL.md
git commit -m "chore: add session-docs-git-wrap-up skill"
```

## Use

Invoke by slash command or name in any chat:

- `/session-docs-git-wrap-up`
- "Use **session-docs-git-wrap-up** to sync docs, run hooks and tests, commit, and push."

The frontmatter omits `disable-model-invocation` so harnesses can trigger it either via `/session-docs-git-wrap-up` or from explicit wrap-up phrases. Add `disable-model-invocation: true` locally if you prefer slash-only invocation.

## Compatibility

- **Harnesses**: Cursor, Antigravity (`~/.gemini/config/skills` -> `~/.agents/skills`), Codex (`~/.codex/skills`), and any Agent Skills loader.
- **Languages / package managers**: Python (`uv`, `poetry`, `pip`), Node (`npm`, `pnpm`, `yarn`, `bun`), Go, Rust, Ruby, and `Makefile` / `justfile` / `Taskfile` repos.
- **Hook frameworks**: `pre-commit`, `husky` + `lint-staged`, `lefthook`, raw `.git/hooks/`, or no hooks (always paired with explicit unit tests when hooks only lint).
- **Repo hosts**: GitHub (`gh`), GitLab (`glab`), Codeberg / Gitea, self-hosted.
- **Shells**: examples use bash/zsh; translate heredocs on Windows PowerShell while keeping the same logic.
- **Offline mode**: skip Phase 6 explicitly when network access is unavailable.

## Contributing

The skill lives in a single markdown file ([`skill/SKILL.md`](skill/SKILL.md)) with YAML frontmatter. Keep it under 500 lines and free of repository-specific secrets, IP addresses, or hostnames.

## License

[MIT](LICENSE).
