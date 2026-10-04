---
name: project-initialization
description: |
  Initializes a project from the standard dual-runtime template, preserves existing files,
  configures Git hooks, and creates or connects a private GitHub repository with main and dev
  branches.

  Use when: "инициализируй проект", "создай новый проект", "init project", "initialize project"
---

# Project Initialization

Initialize the project in the current working directory. Existing project files are preserved in
an `old*` directory for later review; do not merge them into the new scaffold during this workflow.

## Dual-Runtime Project Generation

The bundled template contains Claude project sources under `.claude/**`. Its pre-commit hook runs
the installed `~/.claude/scripts/sync-to-codex.sh` converter after those sources are staged. This
generates and stages the matching `AGENTS.md` and `.codex/**` runtime before the commit while
`.gitignore` excludes host-local `.codex/.sync/**`. A reported conflict or validation error stops
the commit.

## 1. Check the Current Directory

Work only in the current directory. If it is already a Git repository with uncommitted changes,
ask whether to commit them first, continue while preserving them in `old*`, or stop. Do not proceed
until the user chooses.

Resolve the directory of this loaded `project-initialization` skill and set `INIT_SKILL_DIR` to that
absolute path. Verify that its `assets/new-project/` directory exists before moving project files.

## 2. Preserve Existing Files and Apply the Template

If the current directory contains anything other than `.git`, move all such entries into the first
available directory named `old`, `old2`, `old3`, and so on. Do not move `.git`.

```bash
OLD_DIR=""
if find . -mindepth 1 -maxdepth 1 ! -name '.git' -print -quit | grep -q .; then
  OLD_DIR="old"
  N=2
  while [ -e "$OLD_DIR" ]; do OLD_DIR="old${N}"; ((N++)); done
  mkdir "$OLD_DIR"
  find . -mindepth 1 -maxdepth 1 ! -name '.git' ! -name "$OLD_DIR" -exec mv -- {} "$OLD_DIR/" \;
fi

cp -rp "$INIT_SKILL_DIR/assets/new-project/." .
```

If `OLD_DIR` is non-empty, inspect it for `.env*`, `*.key`, `*.pem`, `credentials.json`, and
`secrets/`. Ensure every sensitive path is covered by the new `.gitignore` before staging files.
Never print secret contents.

## 3. Initialize Git and Hooks

Initialize Git with `main` as the primary branch when needed. For an existing repository, preserve
its history and make the current primary branch `main`. Then:

1. Make `.githooks/pre-commit` executable.
2. Set `git config core.hooksPath .githooks`.
3. Stage the scaffold and preserved `old*` directory, subject to the secret check above.
4. Create the first initialization commit. The pre-commit hook generates and stages `AGENTS.md`
   and `.codex/**` while `.gitignore` excludes `.codex/.sync/**`. If generation reports a conflict
   or validation error, the commit stops. If the repository already has history, create a normal
   initialization commit instead of rewriting existing commits.

## 4. Connect GitHub and Create Branches

GitHub is required for this workflow. Verify that `gh` is installed and authenticated.

- If `origin` already exists, show its URL and ask whether it is the intended repository. Stop on
  a mismatch rather than replacing the remote. Confirm through `gh` that the repository is private;
  if it is public, stop and ask whether to make it private or use another repository.
- If `origin` does not exist, ask for the GitHub repository name unless already supplied. The
  supplied name authorizes creating the private repository. Create it with
  `gh repo create {name} --private --source=. --remote=origin`.

Before any push, inspect whether a local or remote `dev` branch already exists. If neither exists,
create `dev` from `main`. If either exists, show its relationship to `main` and ask whether to reuse
it; never reset or recreate an existing `dev`, and stop if local and remote histories conflict.

Ask explicitly before pushing `main` — unless the literal `PROJECT_BOOTSTRAP_AUTOMATED` signal is
present in context (see Automated Bootstrap Mode below), in which case push without asking. After
approval (or immediately, when automated), push `main`, push the created or approved `dev`, leave
`dev` checked out, and read the canonical repository URL through `gh`.

## 5. Report the Result

Report:

- the GitHub URL;
- the created or reused `main` and `dev` branches;
- the preserved `old*` directory, or that the directory was initially empty;
- that `dev` is the active branch;
- the next step: create the initial Project Knowledge with `documentation-writing`.

When `PROJECT_BOOTSTRAP_AUTOMATED` is present, send this same report to
`POST ${SLACK_WORKER_URL}/relay` (header `X-Relay-Token: ${SLACK_RELAY_TOKEN}`, JSON body
`{"channel_id": "${CHANNEL_ID}", "text": "..."}`) instead of chat output — nobody reads this
session's stdout in an automated bootstrap job. This is purely a destination change, not a new
"ask" or anything skipped: Step 5 has no "ask" in either mode.

Phrase this automated report as an **in-progress status, never as completion or readiness** (e.g.
"Repo {url} created, finishing setup..." — not "ready" or "done"). The calling workflow
(`bootstrap-project.yml`) still has to vendor online-pipeline's secrets onto the new repo and write
the Slack channel-to-repo mapping *after* this skill returns; only its own later, dedicated
`/relay` call — gated on all of that actually succeeding — is the authoritative "ready" signal (see
`skills/online-pipeline/SKILL.md`'s Slack Bridge section and the control-plane repo's
`bootstrap-project.yml`). A completion-sounding message here would race that gate: if the
secret-vendoring or KV write fails afterward, the user would already have seen a success-shaped
message moments before seeing a failure — the exact premature-"ready" problem this project's Slack
integration was deliberately designed to avoid at every other step.

Do not review or merge files from `old*` during initialization.

## Automated Bootstrap Mode

Gated on the literal string `PROJECT_BOOTSTRAP_AUTOMATED` appearing in context — never active in
an interactive session, since nothing supplies that literal there. Used by the
`slack-pipeline-interaction` control-plane repo's `bootstrap-project.yml` workflow to create a
brand-new project from a Slack `/new-project` command with no human available to answer prompts.

This mode changes exactly one thing: the "ask explicitly before pushing main" step in Step 4 is
skipped (see above). Nothing else in this skill changes:

- Step 1's "ask" only fires when the current directory is already a git repo with uncommitted
  changes. The automated caller always runs this in a fresh, empty directory, so that condition
  never arises here — no signal is needed to bypass it.
- Step 4's "origin already exists, is this the right repo?" branch only fires when `origin` is
  already configured. The automated caller never pre-creates the repo or configures `origin`
  before invoking this skill, so the "origin does not exist" branch runs instead — the one that
  calls `gh repo create` itself, using the repository name already supplied in context. That name
  satisfies "ask for the GitHub repository name unless already supplied," so that ask does not
  fire either.
- Step 4's "reuse existing `dev`?" ask only fires when a `dev` branch already exists. A brand-new
  repository never has one, so this does not fire either.
- Step 5 has no "ask" — only its report destination changes (see above).

The one real skip (push `main`) is safe here because the triggering Slack command is itself the
user's explicit confirmation to create and push this repository.
