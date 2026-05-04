# session-docs-git-wrap-up

A [Cursor Agent Skill](https://cursor.com/docs/agent/skills) for ending a coding session cleanly: reconcile every doc in the repo with the current code, run every git hook stage without skipping tests, commit, and push — creating a private remote when none exists.

It's language- and host-agnostic. Works on Python / Node / Go / Rust / Ruby / mixed-language monorepos and on GitHub / GitLab / Codeberg / self-hosted.

## What it does

Seven phases the agent runs in order:

| Phase | What it does |
| --- | --- |
| **0. Detect toolchain** | Reads manifests (`pyproject.toml`, `package.json` + lockfile flavor, `go.mod`, `Cargo.toml`, `Makefile`, …) and picks the right runner / hook framework / test runner / docs framework / repo host before doing anything else. Never hardcodes `uv` / `npm` / `gh`. |
| **1. Identify changes** | `git status`, `git diff`, `git log --oneline` to enumerate added / removed / renamed / behavior-changed items. If nothing changed, says so and stops. |
| **2. Inventory every doc** | Builds the full list: `README`, `CHANGELOG`/`HISTORY`/`RELEASES`, `CONTRIBUTING`, `SECURITY`, `docs/`, ADRs, RFCs, monorepo per-package READMEs, agent files (`AGENTS.md`, `CLAUDE.md`, `.cursor/rules/`, Copilot, Windsurf), domain reports in any language, `.env.example`, README badges, code comments that describe project layout. |
| **3. Reconcile each doc** | Twelve specific failure modes: stale paths, stale commands, project-layout trees that rot, capability tables, CHANGELOG entries, cross-references, code examples, configuration docs, anchor/link integrity, terminology, generated-docs rebuild (sphinx / mkdocs / docusaurus / typedoc / mdbook), monorepo sync. |
| **4. Tests and hooks** | Runs the full hooked workflow under the detected toolchain (pre-commit / husky+lint-staged / lefthook / direct scripts). No `--no-verify`, no `SKIP=…`. Distinguishes "obsolete test" from "real failure". |
| **5. Commit** | Detects commit-message convention (commitizen / commitlint / project's own `git log` style). Heredoc form, no surprise prefixes. |
| **6. Push, private remote if needed** | If `origin` exists: just push. If not: create a **private** remote on the right host (GitHub / GitLab / Codeberg / self-hosted) and push. Stops and asks if it can't tell which host you want. |
| **7. Final verification** | Checks `git status`, last commit, remote sync, repo visibility, and that every doc in the inventory was either updated or verified-still-accurate. |

Full text: [`skill/SKILL.md`](skill/SKILL.md).

## Install

Cursor Agent Skills are markdown files placed in either your personal skills directory (available across every project) or a project's local skills directory (shared with everyone using that repo).

### Personal install (recommended)

```bash
git clone https://github.com/mynameisi/session-docs-git-wrap-up.git /tmp/sdgwu
mkdir -p ~/.cursor/skills/session-docs-git-wrap-up
cp /tmp/sdgwu/skill/SKILL.md ~/.cursor/skills/session-docs-git-wrap-up/SKILL.md
rm -rf /tmp/sdgwu
```

Or as a one-liner:

```bash
mkdir -p ~/.cursor/skills/session-docs-git-wrap-up && \
  curl -fsSL https://raw.githubusercontent.com/mynameisi/session-docs-git-wrap-up/main/skill/SKILL.md \
  > ~/.cursor/skills/session-docs-git-wrap-up/SKILL.md
```

> **Do not install into `~/.cursor/skills-cursor/`.** That directory is reserved for Cursor's built-in skills and is managed automatically.

### Project install (commit it with the repo)

If you want every contributor on a project to have this skill available when they open the repo in Cursor:

```bash
mkdir -p .cursor/skills/session-docs-git-wrap-up
curl -fsSL https://raw.githubusercontent.com/mynameisi/session-docs-git-wrap-up/main/skill/SKILL.md \
  > .cursor/skills/session-docs-git-wrap-up/SKILL.md
git add .cursor/skills/session-docs-git-wrap-up/SKILL.md
git commit -m "chore: add session-docs-git-wrap-up skill"
```

### Submodule (stay in sync with upstream)

```bash
git submodule add https://github.com/mynameisi/session-docs-git-wrap-up.git \
  .cursor/skills/session-docs-git-wrap-up-src
ln -s session-docs-git-wrap-up-src/skill .cursor/skills/session-docs-git-wrap-up
```

Pull updates with `git submodule update --remote`.

## Use

In any Cursor chat, ask the agent to follow the skill by name. Examples:

- "Wrap up this session — follow the **session-docs-git-wrap-up** skill."
- "Use **session-docs-git-wrap-up** to sync docs, run all hooks, commit, and push."
- "Time to ship: run **session-docs-git-wrap-up**."

The skill's frontmatter sets `disable-model-invocation` to its default (`true` per the Cursor docs convention this repo follows), so the agent loads it on demand when you name it. If you'd rather have it auto-trigger from ambient phrases like "wrap up" or "commit everything", remove `disable-model-invocation` from the frontmatter in your installed copy.

## Compatibility

- **Languages / package managers**: Python (uv, poetry, pip), Node (npm, pnpm, yarn, bun), Go, Rust, Ruby, anything with a Makefile / justfile / Taskfile. The skill detects the toolchain in Phase 0.
- **Hook frameworks**: pre-commit, husky + lint-staged, lefthook, raw `.git/hooks`, no-hooks.
- **Repo hosts**: GitHub (`gh`), GitLab (`glab`), Codeberg / Gitea, self-hosted. If no host CLI is installed and the user can't say which host to use, the skill **stops and asks** rather than guessing.
- **Shells**: command examples are bash/zsh. On Windows / PowerShell the agent will translate the heredoc form; the underlying logic is portable.
- **Network-required steps**: pushing and verifying remote visibility need network. Offline → skip Phase 6 explicitly.

## Why this exists

Most "wrap up the session" instructions stop at "update the README and commit." That misses:

- per-package READMEs in monorepos
- agent / IDE rule files (`AGENTS.md`, `CLAUDE.md`, `.cursor/rules/`)
- ADRs / RFCs / postmortems / non-English handoff reports
- README badges, project-layout trees, embedded code examples
- generated docs that need rebuilding

…all of which rot quietly until the next person notices. The skill makes the inventory step explicit and the test-skipping step impossible.

## Contributing

Issues and PRs welcome. The skill itself is a single markdown file ([`skill/SKILL.md`](skill/SKILL.md)) with YAML frontmatter; keep it under 500 lines per Cursor's skill-authoring guidance.

## License

[MIT](LICENSE).
