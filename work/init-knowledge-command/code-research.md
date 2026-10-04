# Code Research: init-knowledge-command

Research-only. No design decisions, no proposals — grounded facts with file:line references for
the interview and later drafting.

## 1. The initial Project Knowledge creation flow (`create-project-knowledge.md`)

File: `skills/documentation-writing/references/create-project-knowledge.md` (105 lines).

Routed to from `skills/documentation-writing/SKILL.md:19-24`: "Outside Feature Finalization Mode,
follow create-project-knowledge.md when the user starts or continues initial documentation and
either its interview is still in progress or Project Knowledge is missing, still a template, or
only partially filled."

Mechanics:
- **Phase 0 (Start or Resume)**, lines 8-25: inspects repo/config/`CLAUDE.md`/current docs. If an
  in-progress interview exists at `work/project-knowledge/interview.yml`, resumes from the earliest
  incomplete topic. Otherwise creates `work/project-knowledge/` and copies
  `../assets/project-knowledge-interview.yml` there as `interview.yml`. For a fresh interview with
  no substantive project description in the triggering message, first asks the user to describe the
  project freely (line 17-18).
- **Interview Loop** (lines 27-52), run separately per cycle: find topics below 85% score or
  structurally incomplete, ask 3-4 questions per batch, record Q&A into `conversation_history`,
  update topic `score`/`value`/`gaps`/`status`/`evidence`, save immediately after every response.
  Explicitly multi-turn/multi-batch: "Use as many batches as needed" (line 46).
- **Cycle 1: Project Definition** (lines 54-61) — identity, purpose, audience, capabilities, scope,
  MVP, phasing. Has its own checkpoint requiring user agreement.
- **Cycle 2: Architecture and Technical Decisions** (lines 63-75) — stack, structure, dependencies,
  data flow, constraints. For an existing project, "derive facts from manifests, configuration,
  source, and deployment files" and "launch bounded research subagents" if the repo is large (line
  68-69). Own checkpoint.
- **Cycle 3: Operations, Experience, and Remaining Gaps** (lines 77-86) — patterns, testing,
  deployment, migration, UX, domain areas; revisits every still-failing topic across all cycles;
  shows one final summary. Own checkpoint ("the user has corrected the final summary").
- **Write the Documentation** (lines 88-105) — applies `project-knowledge-structures.md`, writes
  facts in English, offers to add backlog items, "Return to Documentation Review in the main skill"
  (line 97), then after user approval sets `interview_metadata.status: completed` and offers a
  commit (line 103-105).

**No existing "automated" mode concept anywhere in this file.** Unlike `user-spec-planning` (which
has an `ONLINE_PIPELINE_AUTOMATED`-gated resume branch, per `skills/online-pipeline/SKILL.md:478-482`)
or `documentation-writing`'s own Feature Finalization Mode (gated the same way, SKILL.md:167-171),
`create-project-knowledge.md` has zero automation/non-interactive branches, zero literal-signal
checks, and no concept of skipping a question when nobody can answer. It is designed exclusively for
a live back-and-forth: 3 cycles × multiple question batches × explicit checkpoints requiring "the
user agrees"/"the user has corrected the final summary." This is structurally a multi-round
interview exactly like `user-spec-planning`'s interview step, not a one-shot — a single `claude -p`
turn cannot complete it; it needs a turn-per-issue-comment stage shape analogous to `userspec-turn`,
not a one-shot call.

Nothing in `documentation-writing/SKILL.md` or `create-project-knowledge.md` references
`ONLINE_PIPELINE_AUTOMATED`, `PROJECT_BOOTSTRAP_AUTOMATED`, or any other literal gating signal for
this specific flow — those only exist for Feature Finalization Mode (SKILL.md:167-171) and for
`user-spec-planning` (per online-pipeline's "Modified Skills and Why", SKILL.md:471-489).

The "/done" interactive trigger for Feature Finalization Mode lives at
`skills/documentation-writing/SKILL.md:117`: "**Feature finalization:** use this mode only when the
user explicitly asks to finish a feature (including `/done`) and provides or identifies
`work/{feature}/`." That is the exact pattern the new "/init-knowledge" trigger would mirror —
currently create-project-knowledge.md is only reached via the generic "Phase 1: Select the Evidence
Source" item 4 ("Full update or audit") or implicitly via missing/template/partial docs, never via a
dedicated slash-style trigger phrase of its own.

## 2. `online-pipeline/SKILL.md` in full detail

File: `skills/online-pipeline/SKILL.md` (507 lines). Already read in full above; key facts:

### Naming Contract (lines 30-67)
- `slug` = kebab-case(issue title, ≤40 chars) + `-` + issue number. Deterministic, recomputed by any
  job (line 35-38).
- Feature folder `work/{slug}/`; spec branch `userspec/{slug}` (label `userspec-spec`); code branch
  `feature/{slug}` (label `userspec-implement`).
- Status marker: `work/{slug}/logs/working/online-pipeline-status.yml`, single line `status: {value}`
  — values `in_progress`, `awaiting_decision`, `ready_for_review` (userspec only), `ready_for_pr`
  (implement only) (lines 42-47).
- Automation signal: literal `ONLINE_PIPELINE_AUTOMATED`, included in `userspec-turn` and `finalize`
  prompts only (line 48-50).
- `local-placeholder` label (lines 51-56): applied only to issues for features worked on locally;
  created defensively "exactly like `userspec-spec`/`userspec-implement`"; the route job in
  `online-pipeline-userspec.yml` ignores issues/comments carrying it (lines 74-80 of that YAML).
- Switch handoff comment `/switch-online` (lines 57-61).
- Slack mapping marker: first line of issue body, exactly
  `<!-- slack: channel_id={channel_id} thread_ts={thread_ts} -->` (lines 62-66) — absence means "no
  Slack thread to mirror into."

### Stage shape (template a new stage would follow)
Every stage (lines 188-285) is a numbered procedure read by `claude -p` from the prompt text a
workflow step assembles (see YAML `Run <stage>` steps above) referencing
`.claude/skills/online-pipeline/SKILL.md`, stage name by literal string. Each stage:
1. Is told issue number, slug, feature folder, branch explicitly in the prompt (never re-derives
   them itself).
2. Writes the status marker as its *last* committed action before any stop or exit, always
   committing+pushing the marker together with any other changed state **before** exiting — never
   after a push, since each run is a fresh checkout with no disk continuity (explicit for
   `userspec-turn` step 4, `implement` step 2, repeated verbatim as a hard rule).
3. Delegates all PR/merge/label mechanics to the deterministic bash in the workflow YAML, never
   calls `gh pr create`/`gh pr merge` itself (explicit in `userspec-turn` step 7, `implement` step
   3).

### `online-pipeline-userspec.yml` label filtering (grepped directly, lines 74-80 of that file)
```
if printf '%s' "$ISSUE_LABELS_JSON" | jq -e 'index("local-placeholder") != null' >/dev/null; then
  echo "action=skip" >> "$GITHUB_OUTPUT"
  exit 0
fi
```
This runs in the `route` job's "Determine stage" step, after the PR-comment (`/approve`) branch and
before slug/branch computation — i.e. it is checked once, early, for both `issues: opened` and
`issue_comment: created` events, using the JSON-array exact-match (not string contains/`==`, which
GitHub Actions treats case-insensitively — see the `ISSUE_LABELS_JSON` comment at lines 41-44
explaining why). A new `knowledge-init` stage's route job would need the exact same filtering
pattern against the *existing* `online-pipeline-userspec.yml` route job (the task says "the existing
online-pipeline-userspec.yml's route job needs to start ignoring issues carrying the new label") —
i.e. add a second `index("knowledge-init") != null` check alongside the existing
`local-placeholder` check, not a separate mechanism.

### `relay_to_slack` invocation — two different call shapes
1. **From a stage prompt's own bash** (what the agent itself runs, as described by SKILL.md Slack
   Bridge section lines 132-157): the agent is instructed to call `gh issue comment` then
   `relay_to_slack {issue_number} "..."` with the same text — this is prose instruction to the agent
   inside the stage definition (e.g. `userspec-turn` step 4, line 209-211), not a bash function the
   agent necessarily re-implements; in practice the agent runs `gh issue comment` itself (an actual
   tool call) and then would need `relay_to_slack` available as shell — but inspecting the actual
   stage definitions (`userspec-turn`, `implement`), the SKILL.md text says "call `relay_to_slack`"
   as an instruction, trusting the `claude -p` session to run the bash function. Where does that
   function come from inside a `claude -p` session? It is **not defined anywhere accessible to the
   agent's own bash environment** except as copy-pasted prose in SKILL.md itself (lines 136-153) —
   the agent must execute the literal function body via Bash tool calls itself; there is no sourced
   script.
2. **From a workflow YAML step directly** (e.g. `online-pipeline-userspec.yml` "Open or update the
   spec PR" step, lines 246-285, and `online-pipeline-finalize.yml`'s "Report success to Slack" /
   "Report technical failure" steps, lines 135-183): the *exact same* `relay_to_slack` bash function
   body (or an inlined equivalent) is duplicated verbatim inside the YAML `run:` block as real shell,
   executed directly by `bash`, not by the agent. This is deterministic bash, not agent-driven.

So: the helper is **one conceptual contract, duplicated as literal bash text in multiple places** —
once in SKILL.md prose (for the agent to reproduce via its own Bash tool calls) and once per
workflow YAML step that needs it outside the agent's own turn (route job, PR-open step, failure
report step). There is no single shared script file; grepping confirms the identical ~15-line
function body appears 4 times across the three workflow YAMLs (userspec, implement, finalize ×2) and
once more in SKILL.md itself.

### `finalize` stage — commits straight to main (lines 270-285), the direct precedent
```
### Stage: finalize
Triggered once, when the code PR (branch feature/{slug}, label userspec-implement) merges. Runs
on `main` directly — there is no branch for this stage.
1. Append ONLINE_PIPELINE_AUTOMATED ... 2. Run Feature Finalization Mode ... 3. Commit straight to
`main` (there is no finalize branch) and push. If anything fails before this final commit/push,
`main` is untouched ... If it fails after push ..., the next run's `work/completed/{slug}/`
existence check (done by the workflow, not by you) already detects completion and skips re-running
you.
```
Mechanically, in `online-pipeline-finalize.yml`:
- Triggered by `pull_request: closed` (not `issues`/`issue_comment`) filtered to
  `github.event.pull_request.merged == true && contains(..., 'userspec-implement')` (line 22).
- Checks out `ref: main` directly (line 35) — no branch creation step at all, unlike userspec/
  implement which create/checkout a dedicated branch.
- Idempotency via directory existence: `if [ -d "work/completed/$SLUG" ]` (line 44) rather than a
  status-marker check, because there is no branch to persist a status marker across retries within
  main itself before the final commit.
- Retry loop (lines 100-131) resets to `origin/main` on failure (`git reset --hard origin/main`,
  `git clean -fd`) since "main is never reset here... failure before the final commit leaves main
  untouched" — contrasts with `implement`'s retry loop which force-pushes back to a last-good commit
  *on a branch*.
- The actual commit/push to main happens inside Feature Finalization Mode itself (Step 6 of that
  mode, documentation-writing/SKILL.md:178-179: "Commit the Project Knowledge changes and archive
  move with a concise documentation commit"), not in the workflow YAML's own bash — the YAML only
  wraps the `claude -p` call in a retry loop and checks the directory-existence side effect
  afterward.

## 3. `scripts/setup-online-pipeline.sh` and `vendor-skills.sh`

Both live at `skills/online-pipeline/scripts/` (not a separate top-level `scripts/` — the task
description's alternate location guess is wrong; confirmed by `find`).

### `vendor-skills.sh` (109 lines)
- `SKILLS=(user-spec-planning code-writing documentation-writing test-master online-pipeline)`
  (line 27) — **does not currently vendor `project-knowledge`** as its own entry (it's pulled in
  transitively as a reference inside `documentation-writing`, not copied as a top-level
  `.claude/skills/project-knowledge/` by this script — but the scaffold in
  `project-initialization/assets/new-project/.claude/skills/project-knowledge/` exists separately,
  vendored by `project-initialization` itself, not by `vendor-skills.sh`).
- `AGENTS=(code-researcher interview-completeness-checker skeptic userspec-quality-validator userspec-adequacy-validator code-reviewer security-auditor test-reviewer documentation-reviewer)`
  (line 28) — notably **does not include `documentation-reviewer`... wait it does** (last item);
  confirmed present.
- `WORKFLOWS=(online-pipeline-userspec.yml online-pipeline-implement.yml online-pipeline-finalize.yml)`
  (line 29) — a fourth workflow file for the new stage would need to be added to this array.
- Refuses to run over uncommitted changes to any of the above paths (lines 31-43), validates the
  full closure exists in source before changing anything (lines 68-86, "all-or-nothing"), then
  wholesale `rm -rf` + `cp -r` each skill directory (lines 91-95), copies each agent file (97-100),
  copies each workflow file (102-105). Never commits (line 108).
- A new `knowledge-init` stage reusing `documentation-writing` doesn't need a new skill added to
  `SKILLS` (already vendored) — but needs its new workflow YAML added to `WORKFLOWS`.

### `setup-online-pipeline.sh` (117 lines)
- Calls `vendor-skills.sh` first (line 29).
- Creates labels defensively: loop over `userspec-spec userspec-implement` (lines 32-37) plus a
  separate explicit block for `local-placeholder` (lines 40-43) — each checked via
  `gh label list --json name -q '.[].name' | grep -qx "$label"` before `gh label create ... || true`.
  A new `knowledge-init` label would follow the same `local-placeholder`-style explicit block (lines
  40-43 pattern), not the loop (which is reserved for the two PR-labels used by `gh pr create
  --label`).
- Offers to set `CLAUDE_CODE_OAUTH_TOKEN`/`ANTHROPIC_API_KEY`, `GH_PAT`, optionally
  `SLACK_RELAY_TOKEN`+`SLACK_WORKER_URL` (lines 45-91). No new secret is implied by the new stage
  itself (it reuses the same Claude auth + `GH_PAT` + Slack secrets already set up) — confirmed no
  new secret requirement from the feature description either.
- Prints manual steps: install Claude GitHub App, create Claude auth credential, create `GH_PAT`
  (lines 93-116).

Both scripts are duplicated verbatim into
`skills/project-initialization/assets/new-project/.claude/skills/online-pipeline/scripts/` (confirmed
identical paths exist there per the `find` above) — the "Setup Script Maintenance" section of
SKILL.md (lines 491-506) is the documented convention requiring both the source copy and this
vendored scaffold copy to be updated together whenever `assets/workflows/*.yml` changes.

## 4. `skills/project-initialization/assets/new-project/` exact current contents

Full listing obtained via `find` (31 files). Relevant structure:
```
.github/workflows/
  online-pipeline-finalize.yml
  online-pipeline-implement.yml
  online-pipeline-userspec.yml
.claude/skills/online-pipeline/
  SKILL.md
  assets/workflows/{the same 3 yml files}
  scripts/{setup-online-pipeline.sh, vendor-skills.sh}
.claude/skills/documentation-writing/
  SKILL.md
  assets/project-knowledge-interview.yml
  references/create-project-knowledge.md
  references/project-knowledge-structures.md
.claude/skills/{code-writing, test-master, user-spec-planning, project-knowledge}/...
.claude/agents/{9 reviewer/research agent .md files}
README.md, CLAUDE.md, backlog.md, .env.example, .githooks/pre-commit, .gitignore,
.mcp.json, .mcp.bot.json, work/completed/.gitkeep
```
This confirms **two separate vendored copies of the three workflow YAMLs** exist in this scaffold:
once under `.github/workflows/` (where GitHub Actions actually reads them from) and once under
`.claude/skills/online-pipeline/assets/workflows/` (the skill's own bundled copy, used as the source
`vendor-skills.sh` re-copies from when re-run inside that scaffolded project). A new workflow YAML
for the `knowledge-init` stage needs to be added in **both** of these locations inside the scaffold,
exactly mirroring how the existing 3 are duplicated.

`documentation-writing` is vendored as a full standalone `.claude/skills/documentation-writing/`
directory, separate from (not nested inside) `.claude/skills/online-pipeline/` — i.e. `claude -p`
inside a new repo resolves it at `.claude/skills/documentation-writing/SKILL.md`, matching the "Why
Skills Must Be Vendored" section's path convention (SKILL.md:84-86).

## 5. `project-initialization/SKILL.md` Automated Bootstrap Mode — exact scope

Full section quoted above (SKILL.md:117-141). Key confirmed facts, with exact lines:

- Gated on literal `PROJECT_BOOTSTRAP_AUTOMATED` (line 119), "never active in an interactive
  session, since nothing supplies that literal there."
- **Changes exactly one thing**: "the 'ask explicitly before pushing main' step in Step 4 is
  skipped" (lines 124-125). Nothing else changes — Step 1's ask never fires (fresh empty directory,
  line 127-129), Step 4's "origin already exists?" branch never fires (automated caller never
  pre-creates the repo, line 130-133), Step 4's "reuse existing dev?" ask never fires (brand-new repo
  has no `dev` yet, line 136-137).
- **Step 5 has no "ask" at all** (line 138): "Step 5 has no 'ask' — only its report destination
  changes." The destination change itself is specified earlier at lines 98-113: when
  `PROJECT_BOOTSTRAP_AUTOMATED` is present, the Step 5 report is sent via `POST
  ${SLACK_WORKER_URL}/relay` instead of chat output, and must be phrased as **in-progress status,
  never completion** (lines 104-106): *"Repo {url} created, finishing setup..." — not "ready" or
  "done"*. The exact reasoning (lines 106-113): *"The calling workflow (bootstrap-project.yml) still
  has to vendor online-pipeline's secrets onto the new repo and write the Slack channel-to-repo
  mapping after this skill returns; only its own later, dedicated /relay call — gated on all of that
  actually succeeding — is the authoritative 'ready' signal."*

This directly confirms the premise in the task description: **project-initialization itself returns
before any of the new repo's secrets (`GH_PAT`, Claude auth, `SLACK_RELAY_TOKEN`) exist on that
repo.** The quoted lines 106-113 are explicit that secret-vendoring happens in
`bootstrap-project.yml` *after* `project-initialization` (i.e. the `claude -p project-initialization`
call) returns — meaning any GitHub issue created to trigger a `knowledge-init` stage (which itself
needs `CLAUDE_CODE_OAUTH_TOKEN`/`ANTHROPIC_API_KEY` + `GH_PAT` to run its own `claude -p` job on the
new repo) cannot be created from inside `project-initialization`/Step 5 itself — those secrets
literally don't exist on the new repo yet at that point. The issue-creation step must happen in
`bootstrap-project.yml`, strictly after its own secret-vendoring step (matching
`work/completed/slack-pipeline-interaction/user-spec.md:502`'s row 6, which already documents that
`bootstrap-project.yml` "sau đó tự vendor+set-secret (GH_PAT, Claude auth, SLACK_RELAY_TOKEN) cho repo
mới" after the `claude -p` call with `PROJECT_BOOTSTRAP_AUTOMATED`).

## 6. `slack-pipeline-interaction` user-spec.md / decisions.md — relevant extracts

### `bootstrap-project.yml` actual steps/order (as specified, not yet built — control-plane repo is
not present in this working directory)
From `user-spec.md` Expected Behavior step 17 (lines 167-185) and Verification row 3/6 (lines
499/502):
1. Receives `repository_dispatch` from the Worker, payload carries `channel_id` and the repo name
   (explicitly **no** `thread_ts` at this point — no thread exists yet for a brand-new project,
   confirmed by Verification row 6: *"Nhận repository_dispatch đúng payload (channel_id, tên repo,
   KHÔNG có thread_ts)"*).
2. Runs on a **completely empty working directory** — not a checkout of the control-plane repo
   itself (Verification row 3: *"working directory cho claude -p hoàn toàn trống (không checkout nội
   dung repo điều phối)"*).
3. Does **not** run `gh repo create` itself before calling `claude -p` — deliberately leaves that to
   `project-initialization`'s own Step 4 "origin does not exist" branch (lines 172-176: *"để skill TỰ
   chạy gh repo create ở Step 4 của chính nó... KHÔNG tự gh repo create trước rồi mới gọi claude -p"*)
   — avoids landing in the "origin already exists, confirm?" branch that nobody could answer in an
   unattended job.
4. Calls `claude -p` with `PROJECT_BOOTSTRAP_AUTOMATED` plus the repo name already supplied.
5. **After** `project-initialization` returns: the job itself vendors + sets secrets for *both*
   online-pipeline and Slack integration on the new repo — copies `GH_PAT`, Claude auth credential,
   `SLACK_RELAY_TOKEN` from control-plane's own secrets (line 182-185, Verification row 6: *"sau đó tự
   vendor+set-secret (GH_PAT, Claude auth, SLACK_RELAY_TOKEN) cho repo mới"*).
6. **Only on success** (repo created, vendor+secret done): calls `POST /relay` with a new
   `link_channel_to_repo` flag so the Worker writes the `channel_id<->repo` KV mapping (step 18,
   lines 186-201). On failure at any point before this, calls `/relay` to report the error, without
   that flag, and **does not roll back** the already-created repo (Verification row 6: *"không tự
   rollback khi lỗi giữa chừng"*).

This is the exact point in `bootstrap-project.yml`'s existing (specified, not-yet-coded) sequence
after which a `knowledge-init`-labeled issue could be created: strictly after step 5 above (secrets
already vendored onto the new repo), most naturally bundled with or immediately after step 6's
success branch — the new repo cannot run any `claude -p`-driven GitHub Actions job (including a new
`online-pipeline-knowledge-init.yml` workflow) until its own `GH_PAT`/Claude-auth/`SLACK_RELAY_TOKEN`
secrets exist, which step 5 is what creates them.

### `/new-feature` Worker slash-command pattern (for part B's new command to mirror)
From Expected Behavior step 9 (lines 97-107): `/new-feature <description>` triggers:
1. Worker ACKs within <3s (Slack's hard limit), does real work in `ctx.waitUntil` (background).
2. Posts a message to open a Slack thread *first* (to obtain that message's own `thread_ts`).
3. Creates the GitHub issue (title from the description, body with the
   `<!-- slack: channel_id=... thread_ts=... -->` marker line embedded) — thread created before
   issue deliberately, since the issue body needs `thread_ts` already in hand.
4. On issue-creation failure *after* the thread already opened: posts an explicit error message into
   that same thread (it already has a valid `thread_ts`) rather than leaving an "orphaned" thread
   with no issue and no explanation; does not treat the thread as active and does not write any
   `thread_ts -> issue` KV mapping in that failure case.
5. On success: writes `thread_ts -> issue` to Cloudflare KV, sends the result via `response_url`.
   The `userspec-turn` stage then runs, posting its first interview question to both the GitHub issue
   comment and (via the relay) the Slack thread.

A manual Slack command for part B's `knowledge-init` trigger (exact shape marked TBD in the feature
description) would need to decide: does it create a new issue directly (bypassing
`bootstrap-project.yml` entirely, since the target repo in a manual-trigger scenario presumably
already exists and already has secrets vendored), or does it reuse pieces of the `/new-feature`
pattern (thread-first, embed the Slack marker, `response_url` ack)? This user-spec's `/new-feature`
only ever targets a repo that is already onboarded (has `SLACK_RELAY_TOKEN` etc. already set) — the
V1 scope note in the feature description ("NEW repos only... not retrofitting existing repos") means
a manual-trigger path, if built, would likely only make sense for repos that went through
`/new-project` already, which is a different precondition than `/new-feature`'s "channel already
linked" assumption.

### Control-plane repo source location — confirmed NOT committed in this repo's history
`decisions.md` (lines 12-30) records the *intended* scaffold location during that feature's
implementation: a new top-level `control-plane/` directory (sibling to `skills/`, `agents/`,
`scripts/`, `work/`) containing `control-plane/worker/` (Worker source) and
`control-plane/.github/workflows/bootstrap-project.yml`, meant to be copied into a real, separately
created GitHub repo and never vendored per-project (unlike online-pipeline).

**Verified by direct inspection (not just the decision record) that this directory does not exist in
the working tree now and never appeared in any commit:**
```
$ git log --all --oneline -- control-plane   →  (no output, zero commits touched it)
$ ls molyanov-ai-dev/ | grep control          →  (no match)
```
The commit that implemented this feature, `4b8f77d` ("feat(online-pipeline): add Slack bridge and
automated /new-project bootstrap"), explicitly states in its own message: *"source staged under
control-plane/, pushed separately to its own control-plane repo since it's singleton infra, not
per-repo vendored code"* — confirming the staging was a working-tree-only, uncommitted step in this
repo, with the real source living only in the separate `github.com/thanhpd56/control-plane` repo,
which is outside this working directory and was not researched further here (consistent with the
task's framing that it is "source only described in" SKILL.md and the completed feature's
user-spec/decisions).

## 7. "Commit straight to main without a PR" precedent, and label-creation convention

### Straight-to-main precedent
The **only** existing straight-to-main-no-PR precedent in this codebase is the `finalize` stage
itself (`skills/online-pipeline/SKILL.md:270-285`, mechanics detailed in section 2 above). No other
stage, skill, or workflow YAML in this repo commits directly to `main` — `userspec-turn` works on
`userspec/{slug}`, `implement`/`implement-resume` work on `feature/{slug}`, and the Hybrid
Local/Online Switch section (SKILL.md:286-470) explicitly treats "no mid-finalize switch" as a fact
precisely because finalize's lack of a checkpoint branch makes it uninterruptible once started
(line 370-372: *"finalize commits straight to main with no checkpoint, so switching here only ever
means choosing where to run it... before starting it, never during it"*). A new `knowledge-init`
stage that also commits straight to main would be the **second** instance of this pattern, and could
reuse: the directory/file-existence idempotency check style (finalize checks
`work/completed/$SLUG`; a knowledge-init equivalent would need its own marker — Project Knowledge
files themselves, or a dedicated completion marker, since there's no `work/completed/` move involved
here), the `git reset --hard origin/main; git clean -fd` retry-safety pattern (finalize YAML lines
125-130), and the `concurrency: group: <fixed-name>` single-shared-group pattern finalize uses (line
15-18) if Project Knowledge writes need the same race protection finalize has for the same reason
(two runs must never race on the same files).

### Label-creation convention (grepped directly, all sites)
Every online-pipeline label is created *defensively* (check-then-create, never assumed to
pre-exist), using the identical two-line idiom in every site:
```
gh label list --json name -q '.[].name' | grep -qx "$label" || gh label create "$label" --description "online-pipeline" --color "..."
```
Confirmed sites:
- `setup-online-pipeline.sh:32-37` (loop over `userspec-spec`, `userspec-implement`, both color
  `0E8A16`).
- `setup-online-pipeline.sh:40-43` (separate explicit block for `local-placeholder`, color
  `D4C5F9`).
- `online-pipeline-userspec.yml:275` (inline, at spec-PR-creation time, `userspec-spec`,
  `0E8A16`).
- `online-pipeline-implement.yml:203` (inline, at code-PR-creation time, `userspec-implement`,
  `1D76DB`).
- Both of the above are duplicated verbatim in the project-initialization scaffold's vendored copies
  of the same two YAML files.

A new `knowledge-init` label would follow the `local-placeholder` block's exact shape (a standalone
defensive create, not folded into the `userspec-spec`/`userspec-implement` loop, since those two are
grouped because both are used by `gh pr create --label`, whereas `local-placeholder` and the
proposed `knowledge-init` are both *issue* labels applied by different actors — the control-plane
`bootstrap-project.yml` job in the new label's case, not `setup-online-pipeline.sh` or a workflow's
own PR-creation step). Unlike `local-placeholder` (created by `setup-online-pipeline.sh`, applied by
a local interactive session) and `userspec-spec`/`userspec-implement` (created defensively inside
the workflow YAMLs themselves at the moment they're first used), `knowledge-init` would be *applied*
by `bootstrap-project.yml` (a different repo's workflow, external to this one) but would need to be
*created* on the target repo somewhere before that — either `setup-online-pipeline.sh` (so it
pre-exists even on manually-onboarded repos, for consistency, though V1 scope is new-repos-only so
this may be unnecessary) or defensively inside `bootstrap-project.yml`'s own issue-creation call
(`gh label create ... || true` immediately before `gh issue create --label knowledge-init`), or
inside the new workflow YAML's own route job the same way `userspec-spec`/`userspec-implement` are
created at point of use.

## Open questions this research surfaces for the interview (not decisions — just gaps)

1. Where exactly is `knowledge-init` the label *created* (see label-creation convention above) given
   it's applied from a different repo's workflow (`bootstrap-project.yml` in control-plane) than
   where it's consumed (the new repo's own `online-pipeline-userspec.yml` route job and the new
   stage's own route job)?
2. `relay_to_slack` has no single shared script — it's copy-pasted bash in 4+ places. A new stage
   adds at least one more copy (its own route-job filtering step, if it needs one, plus whatever its
   stage prompt does). Worth noting as existing tech debt, not something this feature is scoped to
   fix.
3. `create-project-knowledge.md`'s interview has 3 cycles each with their own checkpoint requiring
   explicit "the user agrees" — for an `ONLINE_PIPELINE_AUTOMATED`-style stage, is every checkpoint
   still answered via issue comment (full fidelity, more turns, closer to how `userspec-turn` handles
   `user_decision_required` stops) or does a new literal signal need to skip/auto-confirm
   checkpoints (mirroring how `ONLINE_PIPELINE_AUTOMATED` currently skips exactly one question in
   Feature Finalization Mode)? Nothing in the current codebase answers this — it's new ground, since
   `create-project-knowledge.md` was never designed with any automated branch at all.
4. The finalize stage's idempotency check is directory existence (`work/completed/$SLUG`). A
   knowledge-init stage produces no `work/completed/` move (there is no `work/{slug}/` feature folder
   backing it in the same sense) — what file/state signals "this repo's initial Project Knowledge
   interview already completed" for retry/idempotency purposes is not yet established by anything in
   this codebase and needs a concrete answer before the workflow YAML can mirror finalize's retry
   pattern.
