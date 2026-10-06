# Code Research: smart-resume-on-token-limit

## Scope of this research

Direct file reads during interview (no subagent dispatch — evidence gathered firsthand, recorded
here to avoid re-deriving it during drafting/validation).

## Current retry/reset mechanism (the thing being changed)

All 3 `online-pipeline` stages share the same shape: a `for ATTEMPT in 1 2 3` loop around a single
`claude -p` call, with a rate-limit auto-switch helper (`try_rate_limit_switch`) added by the
already-completed `control-plane-token-switch` feature.

### `online-pipeline-implement.yml` (source: `skills/online-pipeline/assets/workflows/online-pipeline-implement.yml`)

- Line 215: `LAST_GOOD="$(git rev-parse HEAD)"` — captured once, before the loop.
- Line 216: `ATTEMPTS=3`.
- Lines 182-207: `try_rate_limit_switch()` — greps `$CLAUDE_OUTPUT_FILE` for the two confirmed-real
  rate-limit prefixes (`"You've hit your session limit"` / `"...weekly limit"`). On match, calls
  `POST $SLACK_WORKER_URL/switch-token/auto`. Success → updates `CURRENT_TOKEN`, returns 0.
  `{ok:false, error:"no_alias_available"}` → sets `SWITCH_EXHAUSTED=true`, returns 1. Any other
  failure (no match, Worker call error) → returns 1 with `SWITCH_EXHAUSTED` unset.
- Lines 226-290: the loop. Each attempt runs `claude -p "$(cat PROMPT_FILE)"` with
  `CLAUDE_CODE_OAUTH_TOKEN="$CURRENT_TOKEN"` (line 250-251), piped through `tee
  "$CLAUDE_OUTPUT_FILE"`. After the call, checks `STATUS_FILE` for `ready_for_pr` /
  `awaiting_decision`; anything else is a "technical failure" (line 278), which calls
  `try_rate_limit_switch`. **Regardless of whether the switch succeeded**, if
  `ATTEMPT -lt ATTEMPTS`, lines 285-289 always run: `git push --force origin
  "$LAST_GOOD:refs/heads/feature/$SLUG"` + `git reset --hard "$LAST_GOOD"` + `git clean -fd` —
  discarding all file edits and local commits made during that attempt — then the next loop
  iteration re-sends the **original** `PROMPT_FILE` (built fresh each iteration at lines 229-241,
  identical content every time for a given `STAGE_NAME`) with no memory of prior attempts.
- `PROMPT_FILE` content (lines 230-241) never changes between attempts of the same job — it always
  states the stage name, issue number, slug, feature folder, branch; only differs between
  `STAGE_NAME="implement"` (fresh start) vs `"implement-resume"` (separate, pre-existing concept —
  see "Naming collision" below).
- `ATTEMPTS=3` exhausted with no success → "Report technical failure" step (line 308-331), message
  literally says "failed after 3 attempts (2 retries)".
- `SWITCH_EXHAUSTED=true` → "Report rate-limit pool exhausted" step (line 333-356), stops
  immediately without burning remaining attempts.

### `online-pipeline-knowledge-init.yml` (source: `skills/online-pipeline/assets/workflows/online-pipeline-knowledge-init.yml`)

- Same `try_rate_limit_switch()` helper, byte-identical pattern.
- No `LAST_GOOD` SHA var — this stage commits directly to `main` only on success (its own last
  action per SKILL.md contract), so a failed attempt never has a local commit to roll back. Success
  signal is `AFTER_SHA != BEFORE_SHA` on `origin/main` (line ~196), not a status file value.
- On technical failure + no rate-limit-switch success: `git reset --hard origin/main` + `git clean
  -fd` (lines 216-217) — discards **uncommitted working-tree edits** only (nothing was committed
  yet), then retries with the same fresh prompt.
- `ATTEMPTS=3`, same semantics.

### `online-pipeline-userspec.yml` (source: `skills/online-pipeline/assets/workflows/online-pipeline-userspec.yml`)

- Same `try_rate_limit_switch()` helper.
- `LAST_GOOD` captured at line 283, same `git push --force` + `git reset --hard $LAST_GOOD` pattern
  on failure (lines 360-362), same `ATTEMPTS=3`.

### Tracked copies (must stay in sync — established project convention, not new)

Each of the 3 workflow YAMLs above has **6** tracked copies, not 4 (corrected after skeptic review
caught this undercount — verified via `git ls-files` + `diff -q`, all 6 byte-identical):

- `skills/online-pipeline/assets/workflows/{file}.yml` (source)
- `.codex/skills/online-pipeline/assets/workflows/{file}.yml` (mirror of source)
- `skills/project-initialization/assets/new-project/.claude/skills/online-pipeline/assets/workflows/{file}.yml` (vendor's skill copy, byte-for-byte same as source)
- `.codex/skills/project-initialization/assets/new-project/.claude/skills/online-pipeline/assets/workflows/{file}.yml` (mirror of the above)
- `skills/project-initialization/assets/new-project/.github/workflows/{file}.yml` (vendor's **installed** workflow copy — the file that actually runs as GitHub Actions in any scaffolded repo; documented as a required manual sync target in `skills/online-pipeline/SKILL.md`'s "Setup Script Maintenance" section)
- `.codex/skills/project-initialization/assets/new-project/.github/workflows/{file}.yml` (mirror of the above)

`skills/online-pipeline/SKILL.md` itself genuinely has only 4 tracked copies (source / `.codex`
mirror / vendor / vendor's `.codex` mirror) — the undercount only affected the 3 workflow YAMLs.

`skills/online-pipeline/SKILL.md` has the same 4-copy pattern (source / `.codex` mirror / vendor /
vendor's `.codex` mirror).

## Claude Code CLI session-resume capability (verified locally, `claude --help`)

- `-c, --continue` — "Continue the most recent conversation in the current directory." Works with
  `-p`/print mode (the help text's `--no-session-persistence` entry explicitly calls out "(only
  works with --print)" for session persistence, confirming print-mode sessions persist/resume by
  default).
- `-r, --resume [value]` — "Resume a conversation by session ID, or [none given] pick
  interactively." Also usable with `-p`.
- `--session-id <uuid>` — lets a caller pin a specific session id up front instead of relying on
  CLI-generated ids.
- `prompt` is a positional **argument**, not required to repeat the original task — a short nudge
  string (e.g. "Token refreshed, continue where you left off") is a valid full `-p` invocation
  combined with `-c`/`-r`.
- Session data is stored locally (keyed by working directory), so resume only works within the
  same filesystem/runner. GitHub-hosted `ubuntu-latest` runners are fresh VMs per job — no risk of
  accidentally resuming a stale session from an unrelated earlier job; conversely, a job that gets
  killed outright (e.g. GitHub Actions `timeout-minutes` or the 6h hosted-runner ceiling) loses the
  runner entirely, so there is nothing left to resume — confirmed out of scope per interview answer
  to Q1 (batch 1).

## Naming collision to resolve in the spec/implementation

The codebase already uses the word **"resume"** for a different, pre-existing mechanism:
`ROUTE_ACTION == 'resume'` (`online-pipeline-implement.yml` lines 61-65, 111-123) fires when a
**separate, later** GitHub Actions job run is triggered by a human answering a mid-implementation
decision question via `issue_comment` — `STATUS_FILE` was left at `awaiting_decision`, and the new
job runs stage `implement-resume` with the latest comment injected into a **freshly built prompt**
(lines 210-212, 236-240). This is a brand-new runner/VM, no CLI session continuity involved, and is
unrelated to rate-limit handling.

This feature introduces a second, different mechanism (Claude Code CLI's own `-c`/`-r` session
continuity, triggered by a rate-limit **within** the same job's attempt loop) that also fits the
word "resume." The spec/implementation should use distinct terminology (e.g. "rate-limit resume" /
"session resume" vs. the existing "decision resume" / `implement-resume` stage) to avoid confusion
in logs, `SKILL.md`, and code comments. The two mechanisms do not interact: a rate-limit mid-session
never changes `ROUTE_ACTION` or `STATUS_FILE`, and a decision-resume job never has rate-limit
session state to continue (it's a new runner).

## Prior related feature (for continuity of decisions, not to be re-litigated)

`work/completed/control-plane-token-switch/user-spec.md` — added `try_rate_limit_switch()` to all
3 workflows (Phase A) plus a Cloudflare Worker token pool + Slack slash commands (Phase B, separate
repo `control-plane`, out of scope here). Explicitly decided and recorded under "Accepted
Decisions": *"Rate-limit giữa lúc implement đang code: xử lý giống mọi lỗi kỹ thuật khác của loop
đó — reset về `$LAST_GOOD` rồi làm lại từ đầu với token mới, không xây cơ chế resume giữa chừng
mới."* This feature reverses that specific decision for the rate-limit case only, per explicit user
request and interview answers.

No changes needed in the separate `control-plane` repo — `/switch-token/auto` already returns the
token; this feature only changes how the **caller** (molyanov-ai-dev workflows) reacts to a
successful switch.
