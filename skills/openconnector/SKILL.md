---
name: openconnector
description: Open Connector workflow for third-party app tasks. Use when a user asks to discover or call an external service through Open Connector, or to connect an account for that service.
---

# Open Connector

Use the first-party `oc` CLI. This Skill targets the released `oc` 0.1.x command contract.

## Inspect-first loop

Run each stage to its completion criterion before moving to the next one.

### 1. Prepare

Run `command -v oc` and `oc --version`.

If `oc` is absent, offer this official installer and run it only after the user confirms:

```sh
curl -fsSL https://github.com/openconnector-dev/openconnector/releases/latest/download/install.sh | sh
```

Verify installation with `oc --version`. Configure an existing project API key outside the chat:

```sh
oc login --url <open-connector-url> --key "$OPEN_CONNECTOR_API_KEY" --entity <entity-id>
```

Verify configuration with `oc toolkits list --limit 1`. This stage is complete when the command returns catalog data.

### 2. Discover

Find the toolkit, then the exact tool slug:

```sh
oc toolkits list --search <service-or-task>
oc tools list <toolkit> --search <task>
```

This stage is complete when one returned tool slug matches the requested outcome. Catalog search is keyword ranking; keep the user's terms specific.

### 3. Inspect

Read the selected tool before forming arguments:

```sh
oc tools info <tool-slug>
```

This stage is complete when the required inputs, output shape, and external effect are clear.

### 4. Connect

Reuse an existing connection when it fits the toolkit and end user:

```sh
oc connected-accounts list
```

Otherwise inspect the toolkit's auth methods and existing auth configs, then create or select one:

```sh
oc toolkits info <toolkit>
oc auth-configs list --toolkit <toolkit>
oc auth-configs create <toolkit> --auth-method <auth-method-id>
oc connected-accounts link <toolkit> --auth-config <auth-config-id>
```

Explain the requested OAuth scopes or credentials before starting the link. Treat the authorization URL as a user-interactive handoff. This stage is complete when `oc connected-accounts list` returns the connection to use.

### 5. Execute

Use the inspected schema and selected connection:

```sh
oc tools execute <tool-slug> --connected-account <connection-id> -d '<json>'
```

For a different end user, add the global `--entity <entity-id>` flag. Before an external write, state the target and final arguments, then obtain explicit confirmation. This stage is complete only when the command returns its actual result or error.

## Operating rules

- Keep CLI output as JSON; use `--human` only for user-requested terminal formatting.
- Keep project keys in user-managed environment variables and local CLI configuration.
- Execute one tool per command and report the returned result verbatim enough to support the next decision.
- Use `oc <command> --help` when the installed version reports a command or option mismatch.
