---
name: openconnector
description: Use the first-party `oc` CLI when an agent needs to discover tools or Trigger Types, connect an account, execute a tool, manage Trigger Instances, stream Trigger Events, or proxy a provider API through Open Connector.
---

# Open Connector

Use `oc` through its retained Human OAuth login and selected Organization, Project, and connector userId. It talks to the native `/api/v1` API. Human-readable output is the default; agents must pass `--json` whenever they parse command output.

## Default workflow

1. Run `command -v oc` and `oc --version`. If `oc` is absent, offer the official installer and run it only after the user confirms:

   ```sh
   curl -fsSL https://github.com/openconnector-dev/openconnector/releases/latest/download/install.sh | sh
   ```

   Verify the installed version, then run `oc --json doctor`. Stop and follow its structured `next` instruction when `ok` is false.
2. When exact tool slugs are known, inspect 1–20 of them with one `oc --json tools info <slug...>` call. Never serially invoke `tools info` for candidates that fit in one batch. Every response uses `{ tools, errors, summary }`; each `tools[slug]` is the complete Tool detail. Otherwise use `oc --json search <task>`; search is deterministic, not semantic.
3. For a write tool, prepare input in a file and run `oc --json execute <slug> --dry-run --data @input.json`.
4. Execute with `oc --json execute <slug> --data @input.json` only when the user authorized the action.
5. If execution reports a missing connection, create or find an auth config, run `oc --json link <toolkit>`, and retry the original command.
6. When the host needs the full Open Connector Agent Plugin, run `oc setup <agent> --no-input` only with the user's approval because it changes that agent's plugin configuration. Do not invoke interactive `oc setup` from an agent or script.
7. When the user wants a personal MCP connection, preview `oc --json mcp setup <client> --dry-run`, then apply with `--yes` only after approval. This is separate from Agent Plugin setup.
8. For realtime events, inspect `oc --json triggers list <toolkit>` and `oc --json triggers info <slug>` before creating an instance with an exact Connected Account and reviewed Trigger config.

## Setup

`oc login` uses OAuth Device Authorization. A Human completes this setup and selects an Organization, Project, and Project-local connector userId:

```sh
oc login --url https://api.openconnector.dev
oc orgs switch <organization>
oc projects switch <project>
oc projects user-id <user-id>
```

For SSH or another environment without a browser, use `oc login --no-browser` and hand the displayed device URL and code to the Human. Do not copy one-time OAuth values into files, prompts, or logs. Use `--connector-user-id` during login or `oc --user-id <user-id>` for a one-command override where appropriate.

## Discover, connect, execute

```sh
oc --json search "create a github issue" --toolkit github
oc --json tools info GITHUB_ISSUES_CREATE SLACK_CHAT_POST_MESSAGE
oc --json toolkits info github
oc --json auth-configs create github --auth-method github.oauth2
oc --json --user-id user_123 link github --auth-config ac_...
oc --json execute GITHUB_ISSUES_CREATE --connected-account conn_... \
  --dry-run --data @issue.json
oc --json execute GITHUB_ISSUES_CREATE --connected-account conn_... --data @issue.json
```

Batch `tools info` accepts 1–20 exact slugs; add `--allow-missing` only when missing slugs are expected and a successful exit is desired. Resolve any other per-tool errors before execution. Reuse a matching connection with `oc --json connected-accounts list`; otherwise inspect `oc --json auth-configs list --toolkit <toolkit>`, create/select an auth config, and link it with `oc --json link <toolkit> --auth-config <auth-config-id>`. Explain requested scopes or credential fields before the user-interactive link. `--data`, `--credentials`, and `--connection-config` accept inline JSON, `@file.json`, or `-` for stdin. Keep `--json` explicit in every agent or script command; do not rely on terminal detection or an implicit output mode.

## Realtime Triggers

Inspect the Runtime Catalog and current instances before changing provider subscriptions:

```sh
oc --json triggers list github
oc --json triggers info GITHUB_COMMIT_EVENT
oc --json triggers status --connected-account-ids conn_...
oc --json triggers create GITHUB_COMMIT_EVENT \
  --connected-account conn_... --trigger-config @trigger.json
oc --json triggers enable ti_...
oc --json triggers disable ti_... --force
oc --json triggers delete ti_... --dry-run
oc --json triggers delete ti_... --force
```

`triggers create` creates or reconfigures an instance. Its `--trigger-config`, and any follow-up `--setup-inputs` after a `setup_required` response, accept JSON, `@file`, or stdin. Treat create, disable, and delete as provider-affecting writes: review the exact Connected Account and configuration, preview deletion, and obtain authorization before changing state.

## Listen and forward events locally

`triggers listen` receives new Project Trigger Events over an authenticated realtime stream. Narrow the stream with `--toolkits` (comma-separated) or repeatable `--trigger-id`, `--connected-account-id`, `--trigger-slug`, and event-filter `--user-id` options:

```sh
# Bounded NDJSON smoke test
oc --json triggers listen \
  --toolkits github \
  --trigger-slug GITHUB_COMMIT_EVENT \
  --connected-account-id conn_... \
  --max-events 10

# Forward the production-compatible signed envelope to a local handler
OC_TRIGGER_FORWARD_SECRET=whsec_... \
  oc triggers listen \
    --toolkits github \
    --trigger-slug GITHUB_COMMIT_EVENT \
    --forward http://127.0.0.1:3000/webhooks/openconnector
```

Use `--table` for compact tab-separated summaries (it cannot be combined with `--json`) or `--out <path>` to append protected canonical event views as JSONL with mode `0600`. The listener starts with new events, reconnects with an opaque cursor, and can deliver an event again after a disconnect; de-duplicate business processing by Trigger Event id. `--forward` sends the production webhook envelope with Standard Webhooks headers. Forwarding failures go to stderr and do not block or retry the stream, so retain the event id for a separate replay path. Keep `OC_TRIGGER_FORWARD_SECRET` in the environment or another protected secret store, never in command arguments or committed files.

## Safety

- Do not execute write tools, link accounts, create auth configs, proxy write methods, or delete connections unless the user requested that action.
- Prefer one `tools info` batch (up to 20 slugs), then `execute --dry-run`, before a write. Batch info reports per-tool errors without discarding successful details; dry-run parses the request but does not claim server-side schema validation.
- Never put API keys, client secrets, or credential JSON directly in command arguments. Use stdin (`-`) or a protected file.
- Preview deletion with `oc connected-accounts delete <id> --dry-run`; only use `--force` after explicit user authorization.
- Pass `--json` to keep stdout and errors machine-readable. Read recovery commands from structured errors and retry the narrow original command.
- Do not pass provider access tokens in `oc proxy` headers; Open Connector injects provider authorization from the selected Connected Account.

## Resource commands

- `oc toolkits list|info`: discover Toolkits, inspect metadata, and find auth methods. Use `list --query` to filter.
- `oc tools list|info|execute`: filter tools with `list --query`; `info` inspects one Tool or batches up to 20 complete details in one request; then execute tools.
- `oc auth-configs create|list|info`: manage auth configurations.
- `oc connected-accounts link|list|info|delete`: manage connections.
- `oc triggers list|info|status|create|enable|disable|delete`: discover Trigger Types and manage realtime Trigger Instances. Create accepts JSON, `@file`, or stdin for config/setup inputs.
- `oc triggers listen`: receive Project Trigger Events over an authenticated realtime stream. Scope it with Toolkit, Trigger, Connected Account, Trigger slug, or connector user filters; use `--json` for NDJSON, `--table` for compact output, `--out` for protected JSONL, and `--forward` to POST the production-compatible signed envelope to a local handler. Local forward failures are reported but do not block stream progress; replay a missed event separately when needed.
- `oc proxy <url> --connection <id>`: make a direct provider API request through a connected account.
- `oc mcp url|setup|authorizations`: print or install the fixed Human OAuth `/mcp` endpoint, then list or revoke per-client Project grants. Setup supports `codex`, `claude`, `cursor`, and `vscode`.
- `oc setup` / `oc setup <agent...> --no-input`: humans get detected-host selection; agents use explicit targets. Claude uses its marketplace adapter; compatible clients use the Agent Plugins v1 package through their host-specific install surface.
- `oc upgrade`: replace the CLI binary and refresh every Skill and Agent Plugin installation recorded in `~/.config/oc/installations.json`.

MCP setup writes only the fixed server name `openconnector` and URL; it must never copy the CLI token, a Project key, context headers, or Connector `userId` into a client configuration. In non-interactive use, pass the client explicitly and use `--yes` only with authorization. Revoke an MCP context only when the user names or confirms the exact context.

## Boundaries

The native CLI does not provide a JavaScript sandbox or Composio cloud login. `oc search` uses deterministic native catalog ranking, not semantic search; use `oc triggers listen` for realtime listening. Use `oc <command> --help` to inspect current options when the installed version differs.
