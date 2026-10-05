# Code Research: control-plane-token-switch

Research-only. Grounded facts with file:line references, gathered directly in this same session
(verified fresh, not from stale memory) across both repos this feature touches.

## 1. `control-plane` repo — current Worker structure

### `worker/src/index.js` (469 lines)

- `execute(descriptor)` (top of file) — the one place `fetch` is actually called, executing a
  plain `{url, method, headers, body}` descriptor built by `github.js`/`slack.js`. Every new
  GitHub API call this feature needs follows this same descriptor-builder-then-execute pattern.
- `getRepoMapping(env, channelId)` — reads `env.MAPPINGS.get(channelRepoKey(channelId))`, the
  existing channel→repo KV lookup every slash command already uses to resolve "the repo linked to
  this channel." This feature's new commands reuse it unchanged.
- `processSlashCommand(payload, env)` — the switch statement every slash command is registered in
  (`/new-feature`, `/link-repo`, `/new-project`, `/init-knowledge`). New cases
  (`/switch-token`, `/add-token`, `/remove-token`) are added here the same way `/init-knowledge`
  was added earlier this session.
- `handleRelay(rawBody, env)` — the `/relay` endpoint handler, the **only** endpoint any onboarded
  repo's GitHub Actions job is currently allowed to call, authenticated via `X-Relay-Token` /
  `SLACK_RELAY_TOKEN` (checked in the `fetch` export's `/relay` branch). It already supports a
  `link_thread_to_issue` flag (added earlier this session) as a non-Slack-posting, KV-only side
  effect — this feature's new CI-facing endpoint should follow the same auth scheme
  (`verifyRelayToken`) rather than inventing a new secret, but needs its **own URL path**
  (`/relay` always treats its body as a Slack-relay-shaped payload today; reusing that path for a
  structurally different auto-switch request/response would overload one endpoint with two
  unrelated contracts — a new path, e.g. `POST /switch-token/auto`, keeps them separate).
- `export default { async fetch(request, env, ctx) { ... } }` — the single router: checks
  `request.method !== "POST"` → 404, then branches on `url.pathname` (`/relay` via relay-token
  auth, `/slack/command` and `/slack/events` via Slack signature auth). A new CI-facing path needs
  its own `if (url.pathname === "/switch-token/auto")` branch here, using the same
  `verifyRelayToken` check `/relay` already uses (not Slack signature — the caller is a GitHub
  Actions job, not Slack).

### `worker/src/github.js` (122 lines) — exact current export list

```
buildGetRepoRequest, buildGetIssueRequest, buildCreateIssueRequest, buildCreateCommentRequest,
buildMergePullRequest, buildListOpenPullsByHeadRequest, buildRepositoryDispatchRequest,
buildCreateLabelRequest, buildListOpenIssuesByLabelRequest, buildGetFileContentRequest
```

**No existing builder for the GitHub Actions secrets API.** Two new ones are needed:
- `GET /repos/{owner}/{repo}/actions/secrets/public-key` — returns `{key_id, key}` (key is
  base64-encoded libsodium public key), needed before every secret write.
- `PUT /repos/{owner}/{repo}/actions/secrets/{secret_name}` — body `{encrypted_value, key_id}`,
  `encrypted_value` is the plaintext secret sealed (libsodium `crypto_box_seal`) against the
  public key just fetched, then base64-encoded. This is GitHub's documented, mandatory scheme
  (REST API "Create or update a repository secret") — there is no alternative encoding; every
  public example of "set a GitHub Actions secret via the API" (official docs, popular GH Actions
  like `gliech/create-github-secret`) performs exactly this sealed-box step.

### `worker/src/lib.js` (136 lines) — exact current export list

```
verifySlackSignature, verifyRelayToken, computeSlug, buildIssueBody, parseSlackMapping,
isBotEvent, classifyIncomingMessage, decideApproveAction, isValidRepoName,
isInterviewMarkedCompleted, channelRepoKey, threadIssueKey
```

`channelRepoKey(channelId)` → `"channel:" + channelId`, `threadIssueKey(channelId, threadTs)` →
`"thread:" + channelId + ":" + threadTs` — the two existing KV key-builder conventions this
feature's new ones (pool entries, per-repo current alias, per-alias cooldown) should match in
style: short, colon-separated, one pure function per key shape, each independently unit-tested in
`lib.test.js` (see `describe("KV key builders", ...)` there).

`verifyRelayToken(expected, provided)` — timing-safe string compare, already generic (not
`/relay`-specific); the new CI-facing endpoint reuses this exact function against the same
`env.SLACK_RELAY_TOKEN`, not a new secret.

### `worker/package.json` — current dependencies

```json
"devDependencies": { "vitest": "^2.1.0", "wrangler": "^3.80.0" }
```

Zero runtime dependencies today. This feature adds the **first** one — a sealed-box encryption
library. `tweetnacl` (+ a small sealed-box helper, or hand-rolled on top of `tweetnacl.box`) is
pure JS, no WASM/native bindings, ~8KB, and is the library GitHub's own official Node.js example
for this exact API uses (`tweetnacl` + `tweetnacl-sealedbox-js`) — safe for the Workers runtime
(no filesystem/native-module assumptions) and small enough to stay well inside Cloudflare's
script-size budget.

### `wrangler.toml` — current KV binding

```toml
[[kv_namespaces]]
binding = "MAPPINGS"
id = "1f90742a30114ffd8ef79466eb19e29d"
```

One KV namespace, bound as `env.MAPPINGS`, already used for `channel:*` and `thread:*` keys. This
feature's new keys (pool/alias/cooldown) share this same namespace — no new binding needed,
consistent with "lưu Cloudflare KV" from the interview (Q2).

### `worker/src/index.js` — existing slash-command shape to mirror (`handleInitKnowledge`)

Lines ~183-259 (added earlier this session) show the established shape every new slash-command
handler in this file follows: destructure `payload`, resolve the channel's repo via
`getRepoMapping`, do cheap pre-checks, call GitHub API builders through `execute(...)`, reply via
`respondToSlack(responseUrl, text)`. New handlers for `/switch-token`, `/add-token`,
`/remove-token` should follow this exact shape — none of them need Slack thread-opening
(`buildPostMessageRequest`/`buildJoinConversationRequest`) since they're simple
command-in-command-out replies, not thread-starting ones like `/new-feature`/`/init-knowledge`.

### `worker/src/index.test.js` — does not exist

Confirmed earlier this session: `index.js` is deliberately untested directly (its own top comment:
"This file only wires things together... Business-logic decisions... live in lib.js so they stay
unit-testable") — new pure logic for this feature (rate-limit message parsing, alias-selection/
cooldown logic, sealed-box helper if isolated) belongs in `lib.js`/`github.js` with tests there,
matching this established convention; the new slash-command handlers and the new endpoint's wiring
in `index.js` stay untested directly, same as every existing handler there.

## 2. `molyanov-ai-dev` repo — the three `claude -p` retry sites

All three live under `skills/online-pipeline/assets/workflows/`, each duplicated byte-identically
into `skills/project-initialization/assets/new-project/.claude/skills/online-pipeline/assets/
workflows/` and `.../\.github/workflows/`, and mirrored into `.codex/skills/...` by
`~/.claude/scripts/sync-to-codex.sh --claude-root "$PWD" --codex-root "$PWD/.codex" --apply` — the
exact same three-copy-plus-codex-mirror pattern every change this session already followed for
`online-pipeline-knowledge-init.yml`'s own retry loop and relay fixes.

### `online-pipeline-userspec.yml` — "Run userspec-turn" step (confirmed lines 225-293)

**Has no retry loop at all.** A single `claude -p` call (line 281) under `set -euo pipefail` — if
the process exits non-zero for any reason (including a rate-limit crash), the whole step fails and
the job ends with no retry, today. Per the interview (Q17), this feature adds a full 3-attempt
retry loop here matching the shape of the other two files below — both to support auto-switch and
as an accepted side benefit (today's "zero retry on any crash" becomes "3 attempts" like its
siblings).

Existing per-turn relay logic already in this step (lines 234-250, 252-256, 283-293) — the
`relay_to_slack` function, the `BEFORE_COMMENT_COUNT`/`AFTER_COMMENT_COUNT` diff, and the
`"github-actions"` author filter — must be preserved and now live *inside* the new retry loop,
exactly as they already do in the other two files below.

### `online-pipeline-implement.yml` — "Implement with retry" step (confirmed lines 137-233)

Existing 3-attempt loop: `ATTEMPTS=3` (174), `for ATTEMPT in 1 2 3; do` (183), the `claude -p` call
at line 206 (with `SLACK_RELAY_TOKEN="" SLACK_WORKER_URL=""` blanking for the child process, see
`c237ed2`), success classified by reading `$STATUS_FILE` for `ready_for_pr`/`awaiting_decision`
(line ~215 onward), and on failure resets to `$LAST_GOOD` via `git push --force` + `git reset
--hard` before the next attempt. A rate-limit detection step belongs right after the `claude -p`
call (alongside the existing comment-diff relay block, before the `$STATUS_FILE` check) — on match,
call the new Worker endpoint, export the returned token for the *next* attempt's `claude -p`
invocation, and continue the loop instead of falling through to "technical failure."

Already has `permissions: workflows: write` added this session (`bdbee60`) — unrelated to this
feature, just confirming the file's current state before editing it further.

### `online-pipeline-knowledge-init.yml` — "Run knowledge-init turn with retry" step (confirmed lines 77-171)

Same shape: `ATTEMPTS=3` (112), `for ATTEMPT in 1 2 3; do` (115), `claude -p` call at line 138,
success classified purely by whether a new commit landed on `origin/main` (`AFTER_SHA !=
BEFORE_SHA`, see this session's `c237ed2`/earlier fixes) — no status file involved here, unlike
implement. The rate-limit detection step fits the same way, before the commit-SHA check.

### Common mechanics already proven in this session (do not re-invent)

- Every `claude -p` invocation already blanks `SLACK_RELAY_TOKEN`/`SLACK_WORKER_URL` for that
  child process only, keeping the real values in the surrounding shell step
  (`c237ed2`) — the same pattern applies to whatever new env var carries the *current* token
  value into `claude -p` (it must **not** be the literal `CLAUDE_CODE_OAUTH_TOKEN`/
  `ANTHROPIC_API_KEY` job-level secret unconditionally — after an auto-switch, the retry needs the
  *newly returned* token value, not the original job-level secret, so the loop must overwrite a
  local shell variable with the Worker's response and pass *that* to the next attempt's `claude
  -p`, not re-read the job's fixed secret).
- `relay_to_slack`'s shared shape (same function body duplicated in all three files, confirmed
  identical) is the template for how the new "call the Worker's auto-switch endpoint" bash
  function should look and be duplicated: a small bash function defined once per file, called from
  inside the loop, tolerant of failure (`|| true` where failure must not crash the run; NOT
  tolerant where the response body — the new token — is actually needed for the retry to make
  sense).
- `SLACK_RELAY_TOKEN`/`SLACK_WORKER_URL` are already job-level env in all three workflows'
  relevant jobs — the new Worker call reuses these exact two secrets for its own auth, no new
  secret needed (matches the interview's technical decision to reuse the existing shared-secret
  scheme).

## 3. `bootstrap-project.yml` — current initial-secret-vendoring step

Confirmed current content (this session, `.github/workflows/bootstrap-project.yml`): the "Set
online-pipeline + Slack integration secrets on the new repo" step does:
```bash
gh secret set GH_PAT --body "${{ secrets.GH_PAT }}" --repo "$REPO_FULL_NAME"
if [ -n "${CLAUDE_CODE_OAUTH_TOKEN:-}" ]; then
  gh secret set CLAUDE_CODE_OAUTH_TOKEN --body "$CLAUDE_CODE_OAUTH_TOKEN" --repo "$REPO_FULL_NAME"
else
  gh secret set ANTHROPIC_API_KEY --body "$ANTHROPIC_API_KEY" --repo "$REPO_FULL_NAME"
fi
gh secret set SLACK_RELAY_TOKEN ...
gh secret set SLACK_WORKER_URL ...
```
`CLAUDE_CODE_OAUTH_TOKEN` here is copied from `control-plane`'s own fixed job-level secret
(`secrets.CLAUDE_CODE_OAUTH_TOKEN`) — the exact "every new repo starts on the same one account"
behavior the interview (Q13) asked to change. Per that answer, this step should instead ask the
Worker (same new pool-aware mechanism) for a pool alias to assign at bootstrap time, and set that
token's value as the new repo's `CLAUDE_CODE_OAUTH_TOKEN` secret instead of the fixed job secret.
This bash step has no direct KV access (only the Worker does) — it needs a Worker call here too,
most naturally the *same* new CI-facing endpoint used for auto-switch (asking for "an available
alias for a brand-new repo" is the same selection logic as "an available alias to switch to",
without needing to exclude a previous alias).

## 4. Existing label/secret conventions already established (for consistency, not new research)

- Defensive label creation: `gh label list ... | grep -qx ... || gh label create ...`, duplicated
  per the Naming Contract's documented two-point-create pattern. Not directly relevant here (no
  new label), noted only because the new KV-based pool is the same kind of "shared, mutable state
  across repos" this codebase already has one precedent for (labels), handled by direct idempotent
  creation rather than any locking — the new pool's add/remove commands should follow the same
  simple, no-lock philosophy (accepted small race risk, consistent with this codebase's documented
  tolerance for similar races elsewhere, e.g. the `/init-knowledge` slash command's own two
  best-effort pre-checks).
- `setup-online-pipeline.sh`/`vendor-skills.sh` remain the only update path for the three workflow
  YAMLs on already-onboarded repos (e.g. `sz-ops`, re-vendored three times already this session) —
  this feature's workflow changes need the exact same re-vendor step on every repo that should gain
  auto-switch, which is a manual step for the user, not something this feature can push remotely.

## Open implementation decisions left for drafting (not user decisions — technical detail)

1. Exact KV key scheme: proposed `token-pool:{alias}` → JSON `{token, addedAt}`;
   `repo-token:{owner}/{repo}` → `{alias}`; `token-cooldown:{alias}` → ISO timestamp string
   (absent/expired = available). Finalized during drafting.
2. Exact reset-time-to-absolute-timestamp conversion: `Intl.DateTimeFormat` with the captured named
   zone (`"UTC"`, `"Asia/Saigon"`, ...) to compute the next future occurrence of the parsed local
   time — Workers' V8 runtime ships full ICU data, so arbitrary IANA zone names are supported
   without a separate timezone database dependency.
3. Sealed-box library choice: `tweetnacl` (+ `tweetnacl-sealedbox-js` or a ~10-line hand-rolled
   wrapper over `tweetnacl.box`) — decided during drafting/implementation, not a user decision.
