# Jira Access Skill

Select and authenticate a common access path for Jira operations. The skill
prefers a native Jira or Atlassian MCP server, falls back to ACLI, and can log
ACLI in with environment-provided credentials.

## Expected environment variables

When ACLI may need to authenticate, export:

| Variable | Description | Example |
| -------- | ----------- | ------- |
| `ACLI_EMAIL` | Jira account email | `user@example.com` |
| `ACLI_SITE` | Jira Cloud hostname without `https://` | `example.atlassian.net` |
| `ACLI_TOKEN` | Jira API token from a protected secret source | Do not place a real token in documentation or shell history |

`ACLI_SITE` must identify a Jira Cloud tenant, normally an `atlassian.net`
hostname. Before login, the skill requires it to match the caller's target
site. When ACLI is already authenticated, the skill compares the active site
and account with that target before reusing the session.

The token is passed to `acli jira auth login` through standard input. It must
not be printed, stored in this repository, or supplied as a command-line value.

## Contents

- `SKILL.md`: Access selection, authentication, and safety workflow.
- `agents/openai.yaml`: OpenAI and Codex interface metadata.

Install or link this repository as `jira-access` in an agent skill directory.
