# Code Research: online-pipeline-github-actions

Research performed directly (background `code-researcher` subagent hit the account session rate
limit and failed before producing output; findings below were gathered by direct `Read`/`Bash`
inspection instead).

## 1. `user-spec-planning` commit points

`skills/user-spec-planning/SKILL.md` commits at exactly three milestones, no others:
- Draft: `draft(userspec): create user-spec for {feature}` (Step 4).
- Validation: `chore(userspec): validation round {N} — {summary}` after rounds 1 or 2 only, when an
  accepted finding changed the document (Step 5).
- Approval: `chore(userspec): approve user-spec for {feature}` (Step 6).

Interview turns themselves only "save" the `interview.yml` file (Interview Loop step 4: "save
immediately") — there is no commit instruction tied to each turn. This confirms the gap already
identified in the interview: the new skill must add a commit+push after every interview turn, since
each GitHub Actions run is a fresh checkout with no in-memory continuity between comments.

`scripts/init-feature-folder.sh` and `assets/*.template` are plain, non-interactive (bash +
template copy/substitution) — nothing in them assumes a live terminal session, so they run
unchanged inside a GitHub Actions job.

## 2. `project-initialization` scaffold

`skills/project-initialization/SKILL.md`: scaffolds a new project by copying
`assets/new-project/**` over the working directory, then git-inits with `.githooks/pre-commit`
wired via `git config core.hooksPath .githooks`.

`assets/new-project/` tree (relevant parts):
- `.claude/settings.json` — currently just `{"agents": {"enabled": true}}`. No existing pattern for
  opt-in feature flags beyond this one key; adding an "enable online pipeline" toggle would be a new
  precedent, not an extension of an existing mechanism.
- `.githooks/pre-commit` — project-local hook, separate from the framework repo's own
  `.githooks/pre-commit`/`pre-push` (those live at the framework repo root and govern
  Claude→Codex sync for *this* repo's own skills/agents, not generated projects).
- `backlog.md` — freeform feature/bug list, Russian template text, no structure to hook into.
- **No `.github/workflows/` directory anywhere in this tree, and none exists anywhere else in the
  repo** (`find . -type d -iname workflows` returned nothing). Wiring online-pipeline into
  project-initialization means introducing `.github/workflows/*.yml` to this scaffold for the first
  time — no naming conflict, but no existing convention to follow either.

## 3. `documentation-writing` Feature Finalization Mode

`skills/documentation-writing/SKILL.md`, "Feature Finalization Mode" (lines ~162-178):
1. Reads `user-spec.md`, `decisions.md`, the implementation, and Git history; compares against the
   spec.
2. **"If the feature is evidently incomplete, explain the concrete gap and ask whether to continue
   finalization."** This is a human-in-the-loop branch with no online equivalent defined yet.
3. Updates Project Knowledge, removes `work/{feature}/` from active links, moves it to
   `work/completed/{feature}/`, and creates one finalization commit.

Nothing here requires an interactive terminal — it's addressable via `claude -p` with a trigger
phrase and the feature path, same as the interview/implement steps. The one real gap is step 2:
in a fully automated post-merge trigger, there is no one present to answer "continue finalization?"
if a gap is found. This needs an explicit decision (see open question below) — options are (a) skip
the check and always finalize since a human already reviewed and merged the PR, or (b) keep the
check and, on a detected gap, post a comment and stop automation short of archiving, leaving
`work/{feature}/` in place for manual finalization.

## 4. Framework repo's own sync guards

`.githooks/pre-commit` / `.githooks/pre-push` (repo root) and `scripts/codex-sync-git-guard.py`
operate generically on `SOURCE_PATHS=(CLAUDE.md skills agents commands)` — they don't special-case
any particular skill or asset type. A new `skills/{new-skill}/` directory (including its bundled
`assets/*.yml` workflow templates and `assets/*.sh` setup script) is treated like any other skill
and synced to `.codex/skills/{new-skill}/` automatically. No blocker found.

`scripts/sync-to-codex.py` has a `SECRET_LINE_RE` pattern that scans for secret-looking lines
(`token\s*=\s*...`, `sk-...`, `AKIA...`, etc.) during sync. GitHub Actions' `${{ secrets.X }}`
colon-syntax references (not `=`) do not match this pattern, so bundling workflow YAML with
`${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}` placeholders should not trip the guard — worth a quick
confirmation at implementation time, not a spec blocker.

## 5. Existing `.github/workflows/`

None exist anywhere in the repository (framework root or any skill's bundled assets). No naming
collision risk for whatever workflow file name the new skill introduces.

## 6. Reviewer agents used by `user-spec-planning` validation

`agents/interview-completeness-checker.md`, `agents/skeptic.md`,
`agents/userspec-quality-validator.md`, `agents/userspec-adequacy-validator.md` — each is a
freshly-launched, stateless agent (`model: inherit`, no persisted state between invocations;
`interview-completeness-checker` explicitly "diagnoses gaps only" and the other three validators are
read/diagnose-only). They take the feature path and relevant file contents as input and return a
JSON result. Nothing in their design assumes an interactive session, so invoking them from inside a
non-interactive `claude -p` run in a GitHub Actions job is consistent with how they already work
locally.

## Open question raised by this research

Documentation-writing's Feature Finalization Mode has a built-in "ask the user if the feature looks
incomplete" branch (point 3 above) that has no defined behavior in a fully automated, PR-merge
triggered finalize job. This needs a decision before drafting acceptance criteria for the finalize
step.
