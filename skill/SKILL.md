---
name: session-docs-git-wrap-up
description: >-
  Ends a coding session by reconciling every doc in the repo with the current
  codebase, isolating concurrent-session edits, running hooks plus unit tests
  without skipping, committing, and pushing — creating a private remote when
  none exists. Use when the user asks to wrap up, sync docs, commit everything,
  run hooked tests, or push with a private repo. Language- and host-agnostic;
  works for Python/Node/Go/Rust/etc. and GitHub/GitLab/Codeberg.
version: 1.2.0
---

# Session docs, hooks, git wrap-up

## Mandate (verbatim)

NOW doc what we have done(if not already done), and fix the previous wrong or incomplete docs (if there are any) and then sync the docs and then git commit everything. do not skip any hooked test, if the test is not up to date change or get rid of them, if the tests are reasonable, pass them. then push everything, if there is no git repo already, create one and push. make sure the repo is private.

## Phase 0 — Detect the project's toolchain (do this first)

Never hardcode `uv`, `npm`, `pre-commit`, or `gh`. Detect what this repo actually uses, then use those.

Read the relevant manifests with one batch of file reads / `git ls-files` and answer:

- **Package manager / runner.** Look for: `pyproject.toml` + `uv.lock` (uv) · `poetry.lock` (poetry) · `requirements*.txt` (pip) · `package.json` + lockfile (`pnpm-lock.yaml` → pnpm, `yarn.lock` → yarn, `bun.lockb` → bun, `package-lock.json` → npm) · `go.mod` (go) · `Cargo.toml` (cargo) · `Gemfile` (bundler) · `Makefile` / `justfile` / `Taskfile.yml` (make/just/task).
- **Hook framework and actual hook coverage.** Inspect `.pre-commit-config.yaml` (pre-commit), `lefthook.yml` (lefthook), `.husky/` (husky), `.git/hooks/` raw scripts, or CI-only (no local hooks). Read the hook scripts to check whether they run only static lint/formatting or also execute tests.
- **Test runner.** `pytest`, `unittest`, `vitest`, `jest`, `go test`, `cargo test`, `rspec`, `mix test`, etc. — look in scripts in `package.json`, `pyproject.toml [tool.*]`, `Makefile`, or CI config.
- **Docs framework (if any).** `mkdocs.yml` · `docs/conf.py` (sphinx) · `docusaurus.config.js` · `mdbook` · `typedoc.json` · `astro.config` content collections. Generated docs may need a rebuild step.
- **Repo host.** `git remote -v` → `github.com` (use `gh`), `gitlab.com` (use `glab`), self-hosted, or none. If no host CLI is installed, fall back to UI + `git remote add`.
- **Default branch.** `git symbolic-ref refs/remotes/origin/HEAD` or `git remote show origin | rg 'HEAD branch'`. Fall back to `main`/`master` by inspection.
- **Monorepo or federated subrepos.** Check for multiple `package.json` / `pyproject.toml` / `Cargo.toml` files (expect per-package READMEs/CHANGELOGs) and nested `.git` subrepos or submodules (check each subrepo's `git status` and commit/push child repos before updating parent references).
- **Project governance & deploy gates (workspace-conditional).** Check for project-local rule files (`AGENTS.md`, `GEMINI.md`, `CLAUDE.md`), decision/veto ledgers, architecture linters, hygiene verifiers, or deploy-gate scripts. Follow the repository's own documented entry points rather than inventing ad-hoc commands.

Record what you found and use it for the rest of the phases. When this skill writes commands like `<run> pre-commit …`, substitute the actual runner you detected (`uv run`, `poetry run`, `pnpm exec`, `npx`, bare command, etc.).

## Phase 1 — Identify changes and isolate concurrent edits

Goal: know exactly which behaviors, files, or interfaces moved in *this* session so docs can be reconciled accurately and unrelated work is never swallowed.

1. Inspect `git status --short`, `git diff`, `git diff --stat <default-branch>...HEAD`, and `git log --oneline -20` to enumerate changes since the last push.
2. **Concurrent-session & mixed-hunk boundary check.** Compare every modified or untracked path in `git status --short` against the files this session actually created or edited, and partition them into three buckets:
   - **Session-owned files**: commit in Phase 5.
   - **Unrelated concurrent files**: modified or created by another active session. Never edit, revert, delete, or stage them.
   - **Mixed-hunk shared files**: files touched by both this session and another session. Never stage the whole file blindly (which swallows unfinished concurrent edits or breaks partial commits). Stage only this session's hunks (`git add -p` / patch staging) or pause and coordinate with the user before committing.
3. List **added**, **removed**, **renamed**, and **behavior-changed** items owned by this session:
   - new or removed top-level scripts / entry points / public APIs
   - renamed CLI flags, commands, environment variables, config keys
   - changed defaults, hook stages, dependencies, supported runtimes
   - new or removed reports, generated artifacts, output paths

If nothing of substance changed in this session's scope and docs already reflect reality, say so and stop — do not invent a CHANGELOG entry.

## Phase 2 — Inventory every doc in the repo

Build a complete list before editing anything. Do not assume only `README.md` and `CHANGELOG.md` need updating.

Run patterns like:

```bash
git ls-files '*.md' '*.mdx' '*.rst' '*.adoc' '*.txt' 2>/dev/null
git ls-files | rg -i '(^|/)(readme|changelog|history|releases?|contributing|architecture|design|adr|rfc|notes?|agents|claude|cursor|gemini)\b' \
  | rg -i '\.(md|mdx|rst|txt|adoc)$'
ls .cursor/rules/ AGENTS.md GEMINI.md CLAUDE.md .cursorrules docs/ 2>/dev/null
```

Then add to the inventory (skip what doesn't exist):

- **Top-level**: `README.md`, `CHANGELOG.md` / `HISTORY.md` / `RELEASES.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `LICENSE`.
- **Project docs trees**: `docs/`, `wiki/`, `notes/`, `adr/`, `rfcs/`. Include rendered-doc sources (`docs/index.md`, `docs/conf.py`, `mkdocs.yml`).
- **Decision / veto ledger & session journal (if present)**: any repository decision/veto tombstone file (for example `docs/decisions/*.tsv` or `docs/adr/`) and progress journal declared by the project's agent rules. Whenever this session recorded an architectural decision, veto, or falsified premise, update the ledger so future sessions never repeat rejected paths.
- **Per-package READMEs in monorepos**: every `**/README*` under `packages/`, `apps/`, `crates/`, `services/`, `cmd/`, etc.
- **Agent / IDE guidance**: `AGENTS.md`, `GEMINI.md`, `CLAUDE.md`, `.cursorrules`, `.cursor/rules/*.md`, `.github/copilot-instructions.md`, `.windsurfrules` (respect any byte-parity rules the repo enforces, such as `AGENTS.md == GEMINI.md`).
- **Domain reports** in any language: `*报告.md`, `*-postmortem.md`, `*-rfc.md`, `*-design.md`.
- **Configuration "docs"**: `.env.example`, `pyproject.toml` / `package.json` / `Cargo.toml` `description`/`keywords` fields, `mkdocs.yml` nav.
- **Module-level docstrings / file headers** that read like docs (often describe project layout and rot when files move).
- **README badges** (build status, version, coverage, license, package). Update if URLs / package names changed.

## Phase 3 — Reconcile each doc against reality

For every doc in the inventory, scan for these specific failure modes and fix what you find:

1. **Stale file references.** Cross-check every path mentioned in docs against `git ls-files`. Remove or update references to deleted files; add references to new ones; fix renames.
2. **Stale commands.** Re-read `Quick start` / `Usage` / `Development` sections. Verify every command resolves under the **detected** toolchain (Phase 0). Update flags, scripts, and entry points.
3. **Project-layout trees.** ASCII trees in READMEs rot fast. Diff the tree in the doc against `git ls-files` (top two levels). Update the tree, and update prose that names the files.
4. **Capability tables / feature lists.** If a feature was added, removed, or relocated this session, the corresponding row must change.
5. **CHANGELOG / release notes.** Add an entry under `Unreleased` (or today's date, whichever the project uses). Match the file's own headings (`Added` / `Changed` / `Deprecated` / `Removed` / `Fixed` / `Security`) and the project's commit-message style. If the project doesn't keep a CHANGELOG, don't introduce one — note the change in whatever release-notes channel or journal the project uses, or skip.
6. **Cross-references between docs.** Every link from one doc to another must still resolve, and the target must still say what the source claims.
7. **Code examples / API snippets in docs.** Function names, flags, and return shapes shown in markdown must match current code. Grep the codebase for the example identifiers to confirm.
8. **Configuration docs.** `.env.example`, hook configs, CI configs — if a new env var, secret, or hook stage was introduced, document it; if one was removed, delete the mention.
9. **Anchor / link integrity.** Internal `#anchor` links and relative links (`./other.md`) must still resolve. Fix dead anchors.
10. **Terminology consistency.** Pick one term per concept and use it across all docs (don't mix synonyms for the same thing).
11. **Generated docs (if applicable).** If the project uses sphinx / mkdocs / docusaurus / typedoc / mdbook, rebuild after editing source. Don't commit only the source if the project tracks the built output (rare but exists).
12. **Monorepo / federated sync.** When something at the workspace root or a child repo changed, check whether per-package READMEs, manifests, or parent references also need to move.

When in doubt: edit the doc to match the code. Do not edit code to match the doc unless the user explicitly asks.

## Phase 4 — Tests and hooks (no skipping, no lint-only blind spot)

1. Run the full hooked workflow under the **detected** toolchain:
   - **pre-commit (Python or polyglot)**: `<run> pre-commit run --all-files` → `<run> pre-commit run --hook-stage pre-push --all-files`.
   - **husky / lint-staged (Node)**: `<pm> run lint && <pm> run test` (or whatever `package.json` scripts the husky hooks call).
   - **lefthook**: `lefthook run pre-commit && lefthook run pre-push`.
   - **Raw `.git/hooks/` scripts**: execute the hook's underlying check command directly before committing, and let `git commit` run the real hook.
   - **No local hooks**: run the project's documented lint and test commands directly.
2. **Close the lint-only hook gap.** If local hooks run only static linting or formatting (for example a `.git/hooks/pre-commit` that runs an AST linter without tests), you MUST also run the test suite covering this session's changes (or the full suite when required by the repo's deploy/release gate). Passing a lint-only hook never substitutes for passing tests.
3. Never use `--no-verify`, `--no-gpg-sign`, or `SKIP=…` unless the user explicitly approved it for this turn.
4. If a test fails:
   - First decide *why*. If the test still encodes correct intent, fix the code until it passes.
   - If the test is genuinely obsolete (the behavior it asserted was intentionally removed or replaced this session), update or delete it, and note that in the CHANGELOG / release notes.
   - Do not rationalize away a real failure to "ship the docs."
5. Re-run hooks and tests after any fix until everything passes cleanly.

### Hooks that regenerate artifacts (common pitfall)

If a pre-commit hook runs a **full rebuild** (e.g. an ingest script, doc generator, or formatter that rewrites tracked JSON), the hook may pass while leaving the working tree dirty — or fail on the second run in an infinite loop.

**Fix pattern:**
- Add a **`--check-only`** mode to the eval/rebuild script so it verifies invariants without writing derived files.
- Keep pre-commit on check-only; run full rebuilds explicitly or in CI.

### Agent-written script encoding (macOS)

After creating `.py` or `.sh` files in the same session, run `file <new_scripts>`. If any show **UTF-16**, rewrite them as clean UTF-8 before running hooks — otherwise interpreters fail with opaque `SyntaxError`.

## Phase 5 — Commit (selective allowlist, never blind `git add -A`)

1. **Stage explicitly by path allowlist** (`git add <file1> <file2> ...`) covering only this session's files and reconciled docs. Never run blind `git add -A` when concurrent sessions, untracked scratch files, or nested repos could be present. For a mixed-hunk file from Phase 1, stage only this session's hunks or pause for user coordination.
2. In a federated multi-repo workspace, commit touched child subrepos first (respecting any code-line isolation rules declared in `AGENTS.md`), then commit the parent repo.
3. **Match the project's commit-message convention.** Detect, don't assume:
   - Conventional Commits if `commitizen` / `commitlint` / `cz-conventional-changelog` is configured.
   - The repo's own `git log --oneline -20` style otherwise.
4. Prefer one focused commit when changes are tightly related; split into multiple when they are not (e.g. doc commit separate from a code-fix commit).
5. Pass the message via heredoc to keep formatting clean:

```bash
git commit -m "$(cat <<'EOF'
<type-or-prefix per project convention>: short summary in present tense

<optional body explaining why, not what>
EOF
)"
```

## Phase 6 — Push, with a private remote if needed

1. **If `origin` already exists**:
   - `git push` (or `git push -u origin HEAD` for a new branch).
   - Pre-push hooks must pass — don't bypass them.
2. **If there is no git repo at all**:
   - `git init -b main` (or `master` if that's the user's default).
   - Make an initial commit (after Phases 3–5).
3. **If there is a repo but no remote**: create a **private** remote, then push. Use whatever host CLI matches the user's platform:

   | Host             | Command                                                                                    |
   |------------------|--------------------------------------------------------------------------------------------|
   | GitHub           | `gh repo create <name> --private --source=. --remote=origin --push`                        |
   | GitLab           | `glab repo create <name> --private && git remote add origin <url> && git push -u origin HEAD` |
   | Codeberg / Gitea | API call or UI; then `git remote add origin <url> && git push -u origin HEAD`              |
   | Self-hosted      | UI / API to create as private; then `git remote add origin <url> && git push -u origin HEAD` |

   If no host CLI is installed and the user can't tell you which host to use, **stop and ask** rather than picking one.
4. **Verify privacy** before declaring done:
   - GitHub: `gh repo view --json visibility,isPrivate` → expect `"isPrivate": true` (unless the user explicitly maintains a public repo).
   - GitLab: `glab repo view --output json` → expect `visibility` ≠ `public`.
   - Other hosts: confirm via API or UI.
   If a newly created repo has the wrong visibility, flip it (`gh repo edit --visibility private`, equivalents on other hosts) and verify again. Never silently change visibility on a pre-existing public repo without confirming with the user; report it clearly instead.

## Phase 6B — Deploy Gate & Live Activation Closure (workspace-conditional)

Apply this phase **only** when the session modified code or config targeting a live deployed service, daemon, or remote fleet node. Skip completely for local-only library, script, doc, or skill changes.

1. **Run the repository's deploy gate first (if defined).** Check `AGENTS.md` or project scripts for a deploy gate (code-line check, full-suite green requirement, restart budget, peak/freeze window, or open observation window). Run that gate before syncing or restarting any node.
   - **A gate refusal is a valid completion state.** If the gate refuses, stop deploy execution immediately, mark affected nodes **`STAGED_PENDING_GATE`**, and include the exact refusal output. Never bypass a deploy gate or freeze window just to turn a table green.
2. **When no gate blocks and deployment proceeds**, follow the repository's live-activation protocol (such as `A1..A8`):
   - **Offline-safe sync**: use `git bundle` or verified artifact transfer for hosts without direct upstream access; confirm `disk_sha == target_sha` and `dirty == 0`.
   - **Decoupled reload**: hot-restart every affected service process in the node's process tuple while preserving protected sessions (never cold-restart logged-in browser/app sessions when the repo forbids it), and verify fresh process timestamps.
   - **Runtime provenance, config/cache parity & first-tick liveness**: confirm the running process reports `target_sha`, installed configs/caches match disk, upstream health returns `200 OK`, and post-boot ticks advance with zero crashes.
   - **Honest three-state delivery receipt**: classify every node strictly as `ACTIVATED_IN_RAM`, `STAGED_PENDING_UNLOCK` (awaiting human login/unlock), or `STAGED_PENDING_GATE` (held by deploy gate / freeze window).

## Phase 7 — Final verification (don't skip)

Before reporting completion, confirm all of:

- [ ] **Git cleanliness (concurrent-aware)**: either `git status` is completely clean, OR (when concurrent sessions hold unrelated dirty files) every file owned by this session is committed and pushed while untouched concurrent files are explicitly listed as preserved.
- [ ] `git log -1` shows the new commit on the expected branch (and touched nested subrepos are committed/pushed too).
- [ ] Remote is up to date (`git status -sb` shows up to date with `origin/…`).
- [ ] Repo visibility is verified (private for new/private repos; flag clearly if a pre-existing repo is public).
- [ ] Every doc in the inventory (including any project decision/veto ledger or session journal when applicable) was either updated this session or verified still accurate.
- [ ] Project-defined architecture linters and workspace-hygiene verifiers (when declared in `AGENTS.md` or repo scripts, run via their canonical entry points such as `lint_architecture.py --repo .`) pass with zero errors.
- [ ] All hook stages AND required unit test suites passed without `--no-verify` / `SKIP=…`.
- [ ] **If any live service/fleet node was in scope**: deploy-gate output and/or live-activation provenance (`ACTIVATED_IN_RAM` / `STAGED_PENDING_UNLOCK` / `STAGED_PENDING_GATE`) are verified and reported honestly.

Report concisely: what was changed, what was checked-and-found-already-correct, the commit SHA(s), any preserved concurrent files, and the visibility confirmation.

## Anti-patterns

- Hardcoding `uv` / `npm` / `gh` or project-specific IPs/hostnames instead of detecting the project's toolchain and rules in Phase 0.
- Running blind `git add -A` or staging a whole mixed-hunk file when another session is editing the same workspace.
- Treating a lint-only pre-commit hook as proof that unit tests pass.
- Bypassing a repo's deploy gate or freeze window to force a live restart during Phase 6B.
- Updating only `README.md` and `CHANGELOG.md` without inventorying agent rules, ledgers, ADRs, or per-package docs.
- Treating any module named `sync.py` (or similar) as "doc sync" — read the file before assuming.
- Adding a CHANGELOG entry for changes that didn't actually happen this session, or introducing a CHANGELOG to a project that doesn't keep one.
- Bypassing hooks "just this once."
- Pushing first and then noticing a newly created repo is public, or silently flipping visibility on a pre-existing public repo without telling the user.
- Picking a host (GitHub vs GitLab vs self-hosted) on the user's behalf when there's no remote and you can't tell which they want.
- Leaving session-owned untracked or modified files uncommitted when reporting completion.
- Letting pre-commit run full ingest/regenerate on every commit when a `--check-only` path exists.
