---
name: jira-access
description: "Select and authenticate access for Jira operations. Use whenever an agent needs to read or mutate Jira, including when another Jira-specific skill needs a common access layer: prefer the native Jira or Atlassian MCP server, automatically fall back to ACLI when a native operation fails, and authenticate ACLI from documented environment variables when required."
---

# Jira Access

Use this skill only for access selection and authentication. Let the calling
skill or task define the Jira operation, fields, confirmation gates, and
verification requirements.

## Access order

1. Identify the target Jira Cloud site before any mutation. Use the calling
   skill's site or cloud ID when supplied; otherwise require the user or
   environment to establish the target.
2. Prefer a configured native Jira or Atlassian MCP server that exposes the
   required operation and target.
3. If a native MCP operation fails for any reason, automatically continue with
   ACLI. This includes unavailable tools, connectivity failures, expired
   authentication, permission errors, and unsupported operations. Do not ask
   the user to approve this fallback or stop merely to announce it.
4. Check `command -v acli`. If ACLI is missing, stop and report the blocker.
5. Run `acli jira auth status` and inspect its content, not only its exit code.
6. If the status reports an authenticated session, compare its `Site` with the
   target site and its `Email` with `ACLI_EMAIL` when that variable is set. Stop
   on a mismatch. Account switching is outside this workflow; require the user
   to correct or explicitly request a change to the active ACLI session.
7. If the status does not explicitly report an authenticated session, follow
   the authentication workflow below.
8. Use the equivalent `acli jira` operation only after the target and active
   session have been verified.
9. If ACLI cannot perform the equivalent operation, report the ACLI blocker and
   failed action to the user. Do not silently substitute another Jira client.

Never use the Atlassian Rovo connector. Do not use any unrelated Jira connector
as a silent substitute for the native MCP server or ACLI.

## Expected environment variables

Define these variables in the calling environment when ACLI may need to log in:

| Variable | Expected value |
| -------- | -------------- |
| `ACLI_EMAIL` | Email address for the Jira account, such as `user@example.com` |
| `ACLI_SITE` | Jira Cloud hostname without a URL scheme, such as `example.atlassian.net` |
| `ACLI_TOKEN` | Jira API token supplied through a secret manager or other protected environment source |

`ACLI_SITE` must identify a Jira Cloud tenant, normally a hostname ending in
`.atlassian.net`. A calling skill may define the target site independently, but
the two values must agree when both are present.

Treat `ACLI_TOKEN` as a secret. Never print it, include it in logs or command
arguments, write it to a file, or commit it. Do not enable shell tracing around
the login command.

## ACLI authentication

Authenticate only when the content of `acli jira auth status` does not
explicitly report an authenticated session. First require all expected
variables without printing their values:

```sh
: "${ACLI_EMAIL:?Set ACLI_EMAIL to the Jira account email}"
: "${ACLI_SITE:?Set ACLI_SITE to the Jira Cloud hostname}"
: "${ACLI_TOKEN:?Set ACLI_TOKEN to a Jira API token}"
```

Before logging in, compare `ACLI_SITE` exactly with the target site established
in Access order step 1. Stop and report the mismatch without attempting login.

Then pass the token through standard input:

```sh
(
  set +x
  printf '%s\n' "$ACLI_TOKEN" | acli jira auth login \
    --email "$ACLI_EMAIL" \
    --site "$ACLI_SITE" \
    --token
)
```

Run `acli jira auth status` again. Require its content to affirm an active
session and report the target site from Access order step 1 and the requested
email before continuing; do not rely on exact decoration, casing, or the exit
code alone. If variables are missing or login fails, report only the missing
variable names or the redacted authentication error and stop.

## ACLI operation guardrails

- Prefer `--json` wherever the command supports machine-readable output.
- Use exact issue keys for mutations.
- Require a Jira Cloud target. Stop if the resolved instance is Jira Server or
  Data Center, or if its deployment type cannot be established.
- Preserve every confirmation, assignment, and verification gate imposed by
  the calling skill or user.
- Never claim success until the requested Jira state has been read back and
  verified when a corresponding read operation is available.
- Fall back to ACLI automatically after any native MCP execution failure. Only
  stop when ACLI also cannot perform the requested action safely.
- Treat stricter guardrails in a calling skill as authoritative.
