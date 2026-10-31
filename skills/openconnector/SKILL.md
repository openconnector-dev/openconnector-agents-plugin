---
name: openconnector
description: Use the first-party `oc` CLI when an agent needs to discover tools or Trigger Types, connect an account, execute a tool, orchestrate several tool calls with local JavaScript, manage Trigger Instances, stream or replay Trigger Events, or proxy a provider API through Open Connector.
---

# Open Connector

Use `oc` through its selected Profile: a retained Human OAuth login, or a configured Project Key, plus the selected Organization, Project, and connector userId. It talks to the native `/api/v1` API. Human-readable output is the default; agents must pass `--json` whenever they parse command output. `--json` never prompts, opens a browser, or starts a TUI, so many state-changing commands that normally ask a Human fail until you pass an explicit `--yes`; add it only once the user has authorized that exact change.

## Default workflow

1. Run `command -v oc` and `oc --version`. If `oc` is absent, offer the official installer and run it only after the user confirms:

   ```sh
   curl -fsSL https://github.com/openconnector-dev/openconnector-community/releases/latest/download/install.sh | sh
   ```

   Verify the installed version, then run `oc --json doctor`. Stop and follow its structured `next` instruction when `ok` is false.
2. When exact tool slugs are known, inspect 1–20 of them with one `oc --json tools info <slug...>` call. Never serially invoke `tools info` for candidates that fit in one batch. Every response uses `{ tools, errors, summary }`; each `tools[slug]` is the complete Tool detail. Otherwise use `oc --json search <task>`; search is deterministic, not semantic.
3. For a write tool, prepare input in a file and run `oc --json execute <slug> --dry-run --data @input.json`.
4. Execute with `oc --json execute <slug> --data @input.json` only when the user authorized the action.
5. If execution reports a missing connection, create or find an auth config, then create a hosted link with `oc --json link <toolkit> --auth-config <id> --no-wait`. Give the returned `url` to the Human and ask them to open it and complete the connection. Then run `oc --json connected-accounts wait <connectionId> --timeout 5m`; retry the original command after the connection is `active`. If the wait times out, run `wait` again instead of creating another link.
6. When the host needs the full Open Connector Agent Plugin, preview `oc --json setup <agent> --dry-run`, then apply with `oc --json setup <agent> --yes` only with the user's approval because it changes that agent's plugin configuration. Do not invoke interactive `oc setup` from an agent or script.
7. When the user wants a personal MCP connection, preview `oc --json mcp setup <client> --dry-run`, then apply with `--yes` only after approval. This is separate from Agent Plugin setup.
8. For realtime events, inspect `oc --json triggers list <toolkit>` and `oc --json triggers info <slug>` before creating an instance with an exact Connected Account and reviewed Trigger config.

## Setup

A Human completes setup. `oc login` uses OAuth Device Authorization by default and selects an Organization, Project, and Project-local connector userId:

```sh
oc login --url https://api.openconnector.dev
oc orgs switch <organization>
oc projects switch <project>
oc projects user-id <user-id>
```

For SSH or another environment without a browser, use `oc login --no-browser` and hand the displayed device URL and code to the Human. Do not copy one-time OAuth values into files, prompts, or logs. `login` also accepts `--organization`, `--project`, and `--connector-user-id` to select context in one step. Use `oc --user-id <user-id>` for a one-command connector user override where appropriate.

For unattended automation, the Human can instead configure a Project Key from protected input: `oc configure --url <server> --key @key.json` (or `--key -` from stdin). Never place a Project Key in command arguments.

`oc profiles list|use|remove` manages named server connections; `--profile <name>` selects one for a single command without changing the default. `oc logout` revokes the selected Profile's Human OAuth grant and needs `--yes` outside a prompt.

## Discover, connect, execute

```sh
oc --json search "create a github issue" --toolkit github
oc --json tools info GITHUB_ISSUES_CREATE SLACK_CHAT_POST_MESSAGE
oc --json toolkits info github
oc --json auth-configs create github --auth-method github.oauth2
oc --json --user-id user_123 link github --auth-config ac_... --no-wait
oc --json connected-accounts wait conn_... --timeout 5m
oc --json execute GITHUB_ISSUES_CREATE --connected-account conn_... \
  --dry-run --data @issue.json
oc --json execute GITHUB_ISSUES_CREATE --connected-account conn_... --data @issue.json
```

Batch `tools info` accepts 1–20 exact slugs; add `--allow-missing` only when missing slugs are expected and a successful exit is desired. Resolve any other per-tool errors before execution. Reuse a matching connection with `oc --json connected-accounts list`; otherwise:

1. Inspect `oc --json auth-configs list --toolkit <toolkit>` and create or select one auth config.
2. Run `oc --json link <toolkit> --auth-config <auth-config-id> --no-wait`.
3. Explain the requested scopes or credential fields, then give the returned hosted `url` to the Human. The hosted page handles OAuth and collects API-key, Basic-auth, and provider-specific fields without placing secrets in Agent context.
4. Run `oc --json connected-accounts wait <connectionId>` until the connection is `active`, then continue the original task.

When an existing connection has expired or lost authorization, use `oc --json connected-accounts reconnect <id>` instead of linking a new account; it reauthorizes in place and preserves the account's references, including Trigger Instances. Give any returned authorization URL to the Human.

Use direct `--credentials -` or a protected `@file` only when the Human explicitly requests a noninteractive connection and has chosen a secure local input channel; inline credential JSON is rejected. `--data` and `--connection-config` accept inline JSON, `@file.json`, or `-` for stdin. Keep `--json` explicit in every agent or script command; do not rely on terminal detection or an implicit output mode.

## Orchestrate with local JavaScript

`oc run` executes a JavaScript async function body that can call several Tools and return one JSON result. It is not a sandbox: the script runs locally in a child process with the user's OS permissions. Write the script to a file, show it to the user, and pass `--yes` only after they authorize both the code and its Tool calls:

```sh
oc --json run --file ./workflow.js --yes
oc --json run --file - --yes < ./workflow.js
```

Inside the script, call `await execute(slug, data, { connectedAccount: "conn_..." })`; every call needs an exact Connected Account. Use `return` for the final JSON value; console output goes to stderr. Calls are not a transaction, and a failed call fails the run even when caught, so writes that already succeeded stay applied and are never retried automatically. `--timeout` bounds the whole run (default `5m`). Prefer individual `execute --dry-run` checks for each write before combining them in a script.

## Realtime Triggers

Inspect the Runtime Catalog and current instances before changing provider subscriptions:

```sh
oc --json triggers list github
oc --json triggers info GITHUB_COMMIT_EVENT
oc --json triggers status --connected-account-ids conn_...
oc --json triggers create GITHUB_COMMIT_EVENT \
  --connected-account conn_... --trigger-config @trigger.json --dry-run
oc --json triggers create GITHUB_COMMIT_EVENT \
  --connected-account conn_... --trigger-config @trigger.json --yes
oc --json triggers enable ti_... --yes
oc --json triggers disable ti_... --dry-run
oc --json triggers disable ti_... --yes
oc --json triggers delete ti_... --dry-run
oc --json triggers delete ti_... --yes
```

`triggers create` creates or reconfigures an instance. Its `--trigger-config`, and any follow-up `--setup-inputs` after a `setup_required` response, accept JSON, `@file`, or stdin. Create, enable, disable, and delete are provider-affecting writes: review the exact Connected Account and configuration, preview each change with `--dry-run`, and pass `--yes` only after authorization. Outside an interactive terminal these commands fail without `--yes`. `--force` on enable, disable, and delete is a deprecated alias for `--yes`.

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

Use `--table` for compact tab-separated summaries (it cannot be combined with `--json`) or `--out <path>` to append protected canonical event views as JSONL with mode `0600`. The listener starts with new events, reconnects with an opaque cursor, and can deliver an event again after a disconnect; de-duplicate business processing by Trigger Event id. `--forward` sends the production webhook envelope with Standard Webhooks headers. Forwarding failures go to stderr and do not block or retry the stream, so retain the event id and replay it. Keep `OC_TRIGGER_FORWARD_SECRET` in the environment or another protected secret store, never in command arguments or committed files.

## Inspect and replay retained events

Retained Trigger Events can be listed and replayed to a local handler after a missed forward:

```sh
oc --json trigger-events list --toolkits github --limit 20
oc --json trigger-events replay <eventId>
oc --json trigger-events replay <eventId> --forward http://127.0.0.1:3000/webhooks/openconnector --dry-run
OC_TRIGGER_FORWARD_SECRET=whsec_... \
  oc --json trigger-events replay <eventId> --forward http://127.0.0.1:3000/webhooks/openconnector
```

`trigger-events list` filters with comma-separated `--toolkits`, `--trigger-ids`, `--connected-account-ids`, `--trigger-slugs`, and `--user-ids`, and pages with `--cursor`. Replay without `--forward` only displays the event. Forwarding re-delivers it to the local handler and, outside an interactive terminal, sends immediately without a prompt; preview with `--dry-run` and forward only after authorization.

## Safety

- Do not execute write tools, run `oc run` scripts, link or reconnect accounts, create auth configs, change Trigger Instances, proxy write methods, or delete connections unless the user requested that action.
- Prefer one `tools info` batch (up to 20 slugs), then `execute --dry-run`, before a write. Batch info reports per-tool errors without discarding successful details; dry-run parses the request but does not claim server-side schema validation.
- Pass `--yes` only for the exact command the user authorized, after reviewing its `--dry-run` preview where one exists. Never add `--yes` to get past an approval error the user has not seen.
- For an interactive connection, return the hosted link to the Human. Do not ask them to paste provider secrets into chat or create a temporary credential file.
- Never put API keys, Project Keys, client secrets, or credential JSON directly in command arguments. Use stdin (`-`) or a protected file.
- Preview deletion with `oc connected-accounts delete <id> --dry-run`; only use `--force` after explicit user authorization.
- Pass `--json` to keep stdout and errors machine-readable. Read recovery commands from structured errors and retry the narrow original command.
- Do not pass provider access tokens in `oc proxy` headers; Open Connector injects provider authorization from the selected Connected Account. Preview a proxy request with `--dry-run`, which prints a redacted plan without contacting the provider.

## Resource commands

- `oc doctor`: check configuration, authentication, and server reachability.
- `oc search <query...>`: rank native catalog matches; `--toolkit` is a hard filter and `--limit` bounds results.
- `oc toolkits list|info`: discover Toolkits, inspect metadata, and find auth methods. Use `list --query` to filter.
- `oc tools list|info|execute`: filter tools with `list --query`; `info` inspects one Tool or batches up to 20 complete details in one request; then execute tools. `oc execute` is the top-level shortcut.
- `oc run`: orchestrate several Tool calls with local JavaScript; requires `--yes` and runs with the user's OS permissions.
- `oc auth-configs create|list|info`: manage auth configurations.
- `oc link` / `oc connected-accounts link|list|info|wait|reconnect|delete`: connect, wait for, reauthorize, and manage connections.
- `oc triggers list|info|status|create|enable|disable|delete`: discover Trigger Types and manage realtime Trigger Instances. Writes support `--dry-run` and need `--yes` outside an interactive terminal.
- `oc triggers listen`: receive Project Trigger Events over an authenticated realtime stream. Scope it with Toolkit, Trigger, Connected Account, Trigger slug, or connector user filters; use `--json` for NDJSON, `--table` for compact output, `--out` for protected JSONL, and `--forward` to POST the production-compatible signed envelope to a local handler. Local forward failures are reported but do not block stream progress.
- `oc trigger-events list|replay`: inspect retained Trigger Events and replay one to a local handler.
- `oc proxy <url> --connection <id>`: make a curl-like provider API request through a connected account (`-X`, `-H`, `-d`, `--data-binary @file`, `--dry-run`).
- `oc login|configure|logout|profiles`: manage Human OAuth logins, Project Key configurations, and named Profiles.
- `oc orgs` / `oc projects`: list, create, switch, and show the current Organization or Project; `oc projects user-id` sets the default connector userId. `orgs keys` and `projects keys` manage automation credentials, which print plaintext only once; use them only when the Human asks.
- `oc mcp url|setup|authorizations`: print or install the fixed Human OAuth `/mcp` endpoint, then list or revoke per-client Project grants. Setup supports `codex`, `claude`, `cursor`, and `vscode`.
- `oc setup` / `oc setup <agent...> --dry-run|--yes`: humans get detected-host selection; agents use explicit targets. Supported targets are `claude`, `codex`, `hermes`, `openclaw`, `vscode`, `cursor`, `grok-bot`, `kiro`, `github-copilot`, and `nanoclaw`; hosts without an install API return a successful `manual` result with steps. `--refresh` updates only hosts recorded by an earlier setup.
- `oc upgrade --refresh-only`: refresh every Agent Plugin installation recorded in `~/.config/oc/installations.json` through each host's plugin manager; pass `--yes` outside a prompt. Binary self-upgrade is not currently available; reinstall with the official installer to change the CLI version. Skills from the Open Connector Skill Marketplace are updated separately with `npx skills update`.

MCP setup writes only the fixed server name `openconnector` and URL; it must never copy the CLI token, a Project key, context headers, or Connector `userId` into a client configuration. In non-interactive use, pass the client explicitly and use `--yes` only with authorization. Revoke an MCP context only when the user names or confirms the exact context.

## Boundaries

The native CLI does not provide a JavaScript sandbox or Composio cloud login; `oc run` executes local code with the user's permissions. `oc search` uses deterministic native catalog ranking, not semantic search; use `oc triggers listen` for realtime listening. Use `oc <command> --help` to inspect current options when the installed version differs.
