# Code Research: slack-pipeline-interaction

No prior `code-research.md` existed for this feature; this is the initial version. Interview
(`work/slack-pipeline-interaction/logs/userspec/interview.yml`) already fixed: Cloudflare Workers as
bridge, 1 channel/repo + 1 thread/feature, slash command starts a feature, Slack replaces GitHub
comments for all interaction including code-PR approve/merge (new capability), no per-user
permission restriction, enable once per repo.

## 1. Entry Points — `skills/online-pipeline/SKILL.md` and the three workflow YAMLs

### `skills/online-pipeline/SKILL.md` (427 lines) — exact current wording verified

- Naming Contract (lines 30–65): `slug`, `work/{slug}/` folder, branches `userspec/{slug}` /
  `feature/{slug}`, labels `userspec-spec` / `userspec-implement` / `local-placeholder`, status
  marker `work/{slug}/logs/working/online-pipeline-status.yml` (single line `status: {value}`,
  values `in_progress|awaiting_decision|ready_for_review|ready_for_pr`), automation signal literal
  `ONLINE_PIPELINE_AUTOMATED`, switch handoff literal `/switch-online`.
- Line 63–65: **"The agent communicates with the user exclusively through `gh issue comment
  {issue_number} --body "..."` — chat/stdout output is not read by anyone in a CI job."** — this is
  the single sentence that must change/extend for Slack: every such call is a candidate insertion
  point for an equivalent `chat.postMessage`.
- Stage `userspec-turn` (lines 113–147): step 4 (line 129–136) is where interview
  questions/decisions get posted — `gh issue comment` + status marker write to `awaiting_decision`
  + commit/push, in that literal order ("Write status: awaiting_decision to the status marker
  *before* this commit, not after"). Step 7 (142–144): on clean validation, status →
  `ready_for_review`; **the agent itself never calls `gh pr create`** — the workflow's own step does
  that after the agent exits (confirmed in YAML below).
- Stage `implement` (149–173): step 2 (157–166) — same pattern, `gh issue comment` for
  `awaiting_decision` stop. Step 3 (167–169): on success, status → `ready_for_pr`; again the agent
  never calls `gh pr create` itself — the workflow does.
- Stage `implement-resume` (175–188): resumes from `awaiting_decision`; recognizes literal
  `/switch-online` in the latest comment as a non-answer handoff, not a question answer.
- Stage `finalize` (190–204): runs on `main` directly, no `gh issue comment` call in this stage at
  all (documentation-writing's Feature Finalization Mode does the work; no explicit GitHub post is
  shown in this file for this stage).
- Hybrid Local/Online Switch section (206–390) is entirely local-CLI-facing (not GitHub-Actions
  facing) and posts `/switch-online` via `gh issue comment` at several points (235, 278, 337) — these
  are local-machine actions, not candidates for a Slack mirror by themselves, but establish the
  precedent this feature must follow for an analogous "switch to Slack-driven" posting style if ever
  needed.

### `/approve` comment flow — exact mechanics (`online-pipeline-userspec.yml`)

This is the existing precedent the new code-PR-merge-via-Slack capability must mirror:

- Trigger (lines 7–11): `issues: [opened]` + `issue_comment: [created]` — the workflow listens to
  **all** issue_comment events on the repo (not scoped to PRs only); routing decides what applies.
- `route` job `if:` (line 20): `github.event_name == 'issues' || github.event.comment.user.type !=
  'Bot'` — filters out bot-authored comments (so the bot's own `/approve`-driven commit or
  `gh issue comment` posts don't re-trigger routing).
- Route script (lines 46–120): if `IS_PR_COMMENT == true` (`github.event.issue.pull_request !=
  null`, line 39) **and** `COMMENT_BODY == "/approve"` exactly (line 54) **and** the PR is `OPEN`
  with label `userspec-spec` (lines 59–64) → `action=approve`, derives `slug`/`branch`/`pr_number`
  from `headRefName` (lines 65–70). Any other PR comment → `action=skip` (lines 55–56). This is the
  **entire** listener: a literal string match on comment body, nothing more — no Slack-specific
  concept needed to replicate, just an equivalent literal-match trigger from the Worker.
- `approve` job (lines 121–163): checks out the branch, `sed`-flips `status: draft` →
  `status: approved` in `user-spec.md` and `interview.yml` (149–153), commits, pushes, then **merges
  the spec PR** (line 162): `gh pr merge "${{ needs.route.outputs.pr_number }}" --merge
  --delete-branch`, using `secrets.GH_PAT` (not `GITHUB_TOKEN`) because "GitHub does not let the
  default token's actions trigger other workflow runs" (lines 159–161) — **this merge is what fires
  `online-pipeline-implement.yml`'s `pull_request: closed` trigger.** Direct precedent: a
  Cloudflare-Worker-driven code-PR merge must use an equivalent real token (PAT or GitHub App
  installation token), never `GITHUB_TOKEN`, to guarantee the follow-on workflow fires — though since
  the Worker calls the GitHub REST API directly (not through an Actions job), this `GITHUB_TOKEN`
  restriction specifically does not apply to the Worker itself; it only explains why the *existing*
  `approve` job needs `GH_PAT`. The Worker calling `PUT /repos/{owner}/{repo}/pulls/{pr}/merge`
  directly with its own GitHub App/PAT credential is unaffected by that restriction.

### Finalize trigger and merge-actor neutrality — confirmed, no special-casing

- `online-pipeline-implement.yml` route `if:` (lines 20–22): fires on `pull_request: closed` +
  `merged == true` + label `userspec-spec` — **no check of `sender`, `actor`, `merged_by`, or
  comment content.**
- `online-pipeline-finalize.yml` job `if:` (line 22): `github.event.pull_request.merged == true &&
  contains(github.event.pull_request.labels.*.name, 'userspec-implement')` — same shape, same
  absence of actor/sender checks.
- Repo-wide grep for `merged_by|actor|sender|GITHUB_ACTOR` across all three workflow YAMLs returned
  **zero matches** outside the two `pull_request.merged == true` lines above. Confirmed: a
  Slack-triggered merge performed by a Cloudflare Worker calling the GitHub REST API (`PUT
  /repos/{owner}/{repo}/pulls/{pull_number}/merge`) fires `pull_request: closed` identically to a
  human clicking "Merge" in the GitHub UI — GitHub Actions' `pull_request: closed` event fires
  regardless of merge actor or method, and no workflow in this repo special-cases either. The new
  code-PR-approve-via-Slack capability therefore needs **no new trigger or workflow file** for the
  finalize handoff itself — only a new way to *cause* the merge (Worker → GitHub REST API) and a new
  listener for the Slack-side "approve" message that calls it.

## 2. Setup Scripts — `setup-online-pipeline.sh` / `vendor-skills.sh`

### `skills/online-pipeline/scripts/setup-online-pipeline.sh` (100 lines)

- Line 29: calls `vendor-skills.sh` first (copies skills/agents/workflows).
- Lines 32–43: creates labels `userspec-spec`, `userspec-implement`, `local-placeholder` via
  `gh label create ... || true` (idempotent).
- Lines 54–65: `set_secret_if_confirmed()` helper — prompts y/N, reads secret value with `read -s`,
  calls `gh secret set "$name" --body "$value"`. **This is the exact natural extension point** for
  Slack secrets (bot token, signing secret) and any Cloudflare Worker deploy secret (e.g.
  `CLOUDFLARE_API_TOKEN`) — same helper, same pattern, just more calls.
- Lines 67–74: prompts for Claude auth method, then unconditionally prompts for `GH_PAT`.
- Lines 76–99: prints manual browser-only steps (install Claude GitHub App, create auth credential,
  create PAT). **Natural place to append**: "create Slack App, create channel-creation scope bot
  token, deploy the Cloudflare Worker (`wrangler deploy`), set the Worker's own secrets via `wrangler
  secret put`" — this script only touches **GitHub** repo secrets (`gh secret set`); it has no
  Cloudflare/Wrangler awareness today, so Worker-side secret provisioning would need either new
  `wrangler secret put` calls added here (if Wrangler CLI is assumed installed) or purely printed
  manual instructions, consistent with how GitHub App installation is already handled as a printed
  manual step (lines 81–83) rather than scripted.

### `skills/online-pipeline/scripts/vendor-skills.sh` (109 lines)

- Lines 27–29: `SKILLS=(...)`, `AGENTS=(...)`, `WORKFLOWS=(online-pipeline-userspec.yml
  online-pipeline-implement.yml online-pipeline-finalize.yml)` — fixed arrays, source of truth for
  what gets vendored into a target repo's `.claude/` and `.github/workflows/`.
- Lines 31–43: refuses to run over uncommitted changes in any vendored path (`git status --porcelain`
  check) before overwriting.
- Lines 68–86: validates the full closure exists in source before changing anything (all-or-nothing).
- Lines 88–105: `cp -r` for skills, `cp` for agents, `cp` for workflow YAMLs — **plain file copies,
  no build/bundle step.** A new Cloudflare Worker source file (e.g. `skills/online-pipeline/assets/
  worker/slack-bridge.js` or similar) would need its own new array entry and copy loop here (it is
  not a skill, not an agent, not a workflow YAML — a 4th category), or get folded into
  `online-pipeline`'s own skill directory (`skills/online-pipeline/assets/worker/`) and ride along
  automatically since `SKILLS=(... online-pipeline)` already copies that whole directory tree
  recursively (`cp -r "$SOURCE_DIR/skills/$s"`) — the latter requires no script change at all.
- If a new workflow YAML is added (e.g. a dedicated Slack-approve-merge workflow, if ultimately
  needed — current finding above says it is *not* needed since the Worker calls GitHub REST API
  directly), it would need a new `WORKFLOWS` array entry (line 29) plus the existing copy loop
  (102–105) handles it with no other change.

## 3. Precedent — `work/hybrid-local-online-pipeline/decisions.md`

- File is the untouched template (`<!-- Добавляй запись только для существенного решения -->`,
  3 lines of content) — **no decisions were recorded beyond what's already in `user-spec.md`**
  (already read in a prior session). No new precedent found here beyond what was already known:
  that feature scoped itself as a pure extension of `online-pipeline/SKILL.md` (new "Hybrid
  Local/Online Switch" section) rather than a new skill, and relied entirely on existing naming
  contract (slug/branch/label/status-marker) with one new label (`local-placeholder`) and one new
  literal comment signal (`/switch-online`). This is the direct style precedent for
  slack-pipeline-interaction: extend `SKILL.md` in place, reuse the naming contract and status
  marker unchanged, introduce the minimum new literal signals/labels needed, no new skill file.

## 4. Persistence for `slack_channel_id` / `thread_ts` ↔ `slug` mapping

Repo-wide search for KV store, `.claude` runtime config holding structured state, or GitHub repo
variables usage (`gh variable`, "repo variable", "KV", "kv store") found **nothing** beyond this
feature's own interview notes (`work/slack-pipeline-interaction/logs/userspec/interview.yml:229,
247–253` — the user's own open question, not an existing mechanism). Confirmed:

- The **only** existing precedent for small structured state keyed by feature/slug is the committed
  status marker `work/{slug}/logs/working/online-pipeline-status.yml` (single `status: {value}` line,
  read/written by both the agent and the route-job's `grep`/`sed`) — see Naming Contract line 42–47
  of `SKILL.md`, and its concrete producers/consumers in the three workflow YAMLs (e.g.
  `online-pipeline-userspec.yml:193–194`, `online-pipeline-implement.yml:111–112`,
  `online-pipeline-implement.yml:61` for the `grep -q '^status: awaiting_decision'` read).
- No GitHub repo-variable (`gh variable set`) usage anywhere in the repo. No Cloudflare KV, no
  database, no cache layer of any kind referenced by any skill or workflow.
- Implication for design: a `thread_ts ↔ slug` mapping has exactly two realistic homes given
  existing patterns — (a) a new committed file following the status-marker precedent, e.g.
  `work/{slug}/logs/working/slack-thread.yml`, readable by both the GitHub Actions job (via
  `git`/`grep`) and the Worker (via GitHub Contents API, since the Worker has no git checkout); or
  (b) Cloudflare Workers KV (native to the chosen bridge runtime, no GitHub round-trip needed for the
  Worker's own lookups, but then the GitHub Actions side has no direct way to read it without an
  extra API call to the Worker). The repo-level `channel_id` (1 per repo, created once) has the same
  two-option choice, except it changes far less often, making a single committed config file (not
  per-feature) plausible, e.g. `.claude/skills/online-pipeline/slack-config.yml` or a repo secret/
  variable set once by the setup script.

## 5. Existing Cloudflare Workers / serverless scaffolding

Repo-wide grep for `cloudflare`, `wrangler`, `worker` (case-insensitive, all file types, excluding
`.git/`) found **zero genuine hits**. The only lines containing the substring "worker" are unrelated
generic-English usage, not Cloudflare Workers:

- `AGENTS.md:23` — "orchestrate bounded worker/explorer subagents" (agent-orchestration terminology).
- `.codex/skills/code-reviewing/SKILL.md:128` / `skills/code-reviewing/SKILL.md:127` — "Multiple
  resource instances may be correct for tenant, configuration, process, **worker**, or test" (generic
  code-review guidance about resource-instance correctness).
- `scripts/sync-to-codex.py:382,395` — `("TeamCreate", "spawn_agent worker/explorer orchestration")`
  (same agent-orchestration terminology, appears twice in a mapping table).

**Conclusion: no Cloudflare Workers code, `wrangler.toml`/`wrangler.jsonc`, or any serverless
function scaffolding exists anywhere in this repo today.** This feature will introduce the first
instance of all three; there is no existing deploy/vendor mechanism for a Worker to extend (see §2 —
`vendor-skills.sh` has no concept of a 4th artifact category beyond skills/agents/workflows unless
the Worker source is nested inside the `online-pipeline` skill directory itself).

## 6. Vendoring mechanism — `project-initialization` scaffold

- `skills/project-initialization/SKILL.md` Step 2 (lines 33–49): `cp -rp
  "$INIT_SKILL_DIR/assets/new-project/." .` — the **entire** `assets/new-project/` tree is a literal
  file copy into the new project root, no build step, no templating/variable substitution beyond
  what the agent does afterward by hand (README/CLAUDE.md placeholders).
- `skills/project-initialization/assets/new-project/.github/workflows/` already contains plain
  duplicates of all three `online-pipeline-*.yml` files (confirmed via directory listing) — this
  mirrors `SKILL.md`'s own "Setup Script Maintenance" section (lines 411–419): whenever
  `assets/workflows/*.yml` changes, the same files must be **manually copied** over this scaffold
  path too ("plain duplicates, not a build step or symlink").
- `skills/project-initialization/assets/new-project/.claude/skills/online-pipeline/` is a vendored
  snapshot of the whole skill directory (confirmed: contains `SKILL.md`, `assets/`, `scripts/`
  subdirectories) — identical structure to what `vendor-skills.sh` produces in any other target
  repo. **Any new file added under `skills/online-pipeline/assets/` (e.g. a Worker source file or a
  `wrangler.toml` template) automatically rides along both vendoring paths** (the live
  `vendor-skills.sh` `cp -r` of the whole skill dir, and this scaffold's own `cp -rp` of the whole
  `assets/new-project/` tree) **with no script change**, provided it is placed inside
  `skills/online-pipeline/` itself rather than as a sibling top-level directory. A new top-level
  workflow YAML (if ever needed) requires exactly two manual edits: `vendor-skills.sh`'s
  `WORKFLOWS=(...)` array (line 29) and a manual copy into
  `skills/project-initialization/assets/new-project/.github/workflows/` (per the Setup Script
  Maintenance note) — this feature's plan should account for both if a new workflow file turns out
  to be necessary (current finding in §1 above suggests it is **not** necessary for the merge-trigger
  side, since `pull_request: closed` already fires Worker-initiated merges identically).
- `README.md` in the scaffold (lines 41–63) documents the 3 manual setup steps (GitHub App install,
  Claude auth secret, `GH_PAT`) in prose — the natural place to append Slack/Worker setup steps if
  the scaffold is kept in sync, per the existing "update the matching steps in that scaffold's
  README.md" instruction in `SKILL.md` line 419.

## 7. Open design questions this research surfaces (not decisions — flagging for user-spec)

- Where `thread_ts ↔ slug` and `repo ↔ channel_id` actually live is unresolved by precedent (§4) —
  no existing mechanism to defer to; this needs an explicit decision in the interview/spec, not just
  "follow existing pattern."
- Whether the Slack-side "approve" trigger for a code PR needs any new GitHub Actions trigger at all:
  current finding is **no** — the Worker can call the merge REST endpoint directly and the existing
  `pull_request: closed` triggers in `online-pipeline-implement.yml`/`online-pipeline-finalize.yml`
  fire unchanged (§1). The only new GitHub-side mechanics needed are (a) the Worker's own call to
  `gh issue comment`-equivalent (REST `POST .../issues/{n}/comments`) for `/approve` on the spec PR
  (already exactly what the existing route job listens for, per §1's `/approve` mechanics), and (b)
  a new direct `PUT .../pulls/{n}/merge` call for the code PR (net-new capability, no existing GitHub
  Actions listener needed since merging directly fires the close event).
- No existing Slack-specific secret-naming convention anywhere in the repo (confirmed via the same
  greps as §5) — names like `SLACK_BOT_TOKEN`, `SLACK_SIGNING_SECRET`, `CLOUDFLARE_API_TOKEN` would
  be entirely new, following the existing `gh secret set` pattern in `setup-online-pipeline.sh`.
