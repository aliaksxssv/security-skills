# security-skills

Claude Code slash commands and supporting materials for AWS security assessments using the think-cnap framework.

## Project structure

```
.claude/commands/         # Slash commands — invoke with /think-cnap-aws etc.
memory/                   # Reference docs loaded manually or cited by commands
tools/                    # Helper scripts for assessments
```

## Available slash commands

| Command | Description |
|---|---|
| `/think-cnap-aws` | Assess an AWS account against the think-cnap Detection domain (SEC04) and submit results to thinkcnap.org |

## How to add a new skill

1. Create `.claude/commands/<skill-name>.md` — this becomes `/<skill-name>` in Claude Code
2. Start the file with a one-line description (first paragraph is shown in `/help`)
3. Reference shared knowledge from `memory/` using `Read` tool calls within the skill
4. Note any AWS permissions required — use `arn:aws:iam::aws:policy/SecurityAudit` as the baseline

## Conventions

- API tokens come from thinkcnap.org → Integrations → API Token
- AWS credentials should use the AWS managed policy `arn:aws:iam::aws:policy/SecurityAudit`
- Maturity scale is defined in `memory/maturity-scale.md` — use it for all scoring
- Never store credentials; ask the user each session
- Never display credentials (tokens, keys, secrets) in terminal output — mask as `***` if they must be referenced
- **Always pass credentials via environment variables** — never hardcode tokens or keys inline in shell commands. Export them first (`export THINKCNAP_API_TOKEN=...`, `export AWS_ACCESS_KEY_ID=...`) and reference as `$VAR` in all subsequent commands

## Deployment

These are local Claude Code slash commands — no deployment needed. Copy or symlink `.claude/commands/` into any project where you want the skills available, or keep this repo checked out and open it in Claude Code.
