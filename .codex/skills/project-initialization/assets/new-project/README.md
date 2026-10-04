<!--
Scaffold for the project README. README is for humans — write the final content
in the language the user writes in. Headers below are a starting point; localize them too.
-->

# [Project name]

> **This README is for the project owner**, not for AI agents.
> Instructions for agents live in CLAUDE.md and the .claude/skills/project-knowledge/ skill.

## About

[Short description: what the project does and why it exists]

## Project structure

```
.claude/                    # Knowledge base for AI agents
├── skills/
│   └── project-knowledge/  # Project docs (architecture, patterns, etc.)
└── ...

backlog.md        # Feature ideas and bugs (what to do later)
work/             # Active features and bugs (what we're doing now)
└── [features]/   # Each planned feature has one user-spec.md

src/              # Source code
```

## Development methodology

The project uses a **spec-driven approach** with AI agents:

1. **User Spec** (user's language) → agree on the complete outcome
2. **Implementation** → a new AI-agent chat executes that user-spec directly
3. **Finalization** → project documentation is updated and the feature is archived

Future feature ideas and known bugs are tracked in `backlog.md`. Active feature and bug work lives
in the `work/` folder.

## Online Pipeline (optional)

This repo already has the GitHub Actions workflow files to run the whole feature lifecycle (issue →
interview → spec PR → approve → implement → code PR → finalize) through GitHub only, no local
terminal needed — plus a vendored snapshot, under `.claude/skills/` and `.claude/agents/`, of every
skill and agent those workflows depend on (a CI runner has no access to your machine's global
`~/.claude/skills/`, so this repo carries its own copy). All of it is inert until setup is finished:

1. Install the Claude GitHub App on this repository (run `claude` locally once and use
   `/install-github-app`, or visit https://github.com/apps/claude).
2. Create a Claude auth credential — `claude setup-token` (OAuth) or an API key from
   https://console.anthropic.com — and store it as the `CLAUDE_CODE_OAUTH_TOKEN` or
   `ANTHROPIC_API_KEY` repo secret.
3. Create a Personal Access Token (`repo` scope) at https://github.com/settings/tokens and store it
   as the `GH_PAT` repo secret (needed only so approving a spec PR can trigger the implement job).
4. (Optional) To also drive this repo's online-pipeline from Slack: set the `SLACK_RELAY_TOKEN` and
   `SLACK_WORKER_URL` repo secrets (same values as the account-wide Slack bridge Worker already
   deployed elsewhere — see that control-plane repo's own README), create a public Slack channel
   for this project, and run `/link-repo <owner>/<name>` in it once. Skip this entirely if there is
   no Slack bridge for this account yet; online-pipeline works exactly the same without it.

No script needed for this repo — the workflow files and vendored skills are already in place. Once
the secrets are set, open a new issue to start the pipeline.

The vendored skills/agents are a snapshot and will go stale as the upstream framework evolves.
Re-run `.claude/skills/online-pipeline/scripts/vendor-skills.sh` anytime to pull the latest versions
(clones the public source repo, no auth needed) — nothing is committed automatically, review the
diff first.
