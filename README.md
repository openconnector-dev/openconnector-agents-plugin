# Open Connector Agent Plugins

Installable agent plugins for Open Connector.

The first release is an [Agent Plugins v1](https://agent-plugins.org/) skills-only package. Its single Skill guides an agent to install and use the first-party `oc` CLI; it does not bundle MCP servers, hooks, apps, or runtime code.

## Package design

The repository root is the portable package root:

- `plugin.json` is the Agent Plugins v1 manifest.
- `skills/openconnector/` is the single source of the Open Connector workflow.
- `.codex-plugin/plugin.json` is a thin Codex marketplace manifest. It supplies Codex install-surface metadata and points at the same `skills/` directory; it does not duplicate the Skill.
- `.claude-plugin/plugin.json` is the Claude Code manifest and likewise points at the same `skills/` directory.

This keeps the portable capability in the Agent Plugins layout while retaining the marketplace manifests required by Codex and Claude Code. No platform wrapper forks the Skill.

Because the Agent Plugins manifest is at the repository root, Hermes can install this repository as a portable package. OpenClaw can install it as an Agent Plugins, Codex, or Claude bundle; the shared Skill is the same in every case.

## Install

The package contains instructions only. It does not install, configure, or run
the `oc` CLI until an agent needs Open Connector and asks for the required
confirmation.

### Codex

From a local checkout of this repository:

```sh
codex plugin marketplace add .
codex plugin add openconnector@openconnector
```

Start a new Codex task after installing or updating the plugin so its Skill is loaded.

### Claude Code

From a local checkout of this repository:

```sh
claude plugin marketplace add .
claude plugin install openconnector@openconnector
```

Claude Code exposes the Skill as `/openconnector:openconnector`. Run `/reload-plugins` when Claude Code asks for it after installation or a local update.

### Hermes

Hermes `v0.20.0` was verified to support the root `plugin.json` directly
(`v0.17.0` did not). Install the published repository and explicitly enable the
package:

```sh
hermes plugins install openconnector-dev/openconnector-agents-plugin --no-enable
hermes plugins enable openconnector
```

For a local checkout, Hermes accepts its Git URL too:

```sh
hermes plugins install "file://$PWD" --no-enable
hermes plugins enable openconnector
```

Hermes exposes the packaged Skill through a deterministic `agent-plugin-*`
namespace. Use `skills_list` to discover the fully qualified name, then load it
with `skill_view`. Start a new session, or restart the gateway, after enabling
the package.

### OpenClaw

For local development, install the repository root as a bundle:

```sh
openclaw plugins install .
openclaw gateway restart
```

For the published repository, browse the included Claude marketplace and install
`openconnector` from it:

```sh
openclaw plugins marketplace list openconnector-dev/openconnector-agents-plugin
openclaw plugins install openconnector --marketplace openconnector-dev/openconnector-agents-plugin
openclaw gateway restart
```

### Other Agent Plugins v1 clients

`plugin.json` and `skills/openconnector/SKILL.md` at the repository root form a
portable Agent Plugins v1 package. Clients that support this specification can
install this repository with their own package command; the protocol does not
define one universal installation CLI. The package is skills-only: no MCP
servers, hooks, apps, or executable runtime are included.

## After installation

Ask the agent to use Open Connector for an external-service task. The Skill
will first inspect whether `oc` is available, request confirmation before
installing it when needed, and then guide discovery, account connection, tool
execution, and realtime Trigger workflows such as forwarding events to a local
handler. It does not ask you to place credentials in the plugin files.

## Validation

Validate the Codex wrapper:

```sh
python3 /Users/jelf/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py .
```

The portable manifest targets `https://agent-plugins.org/schemas/1.0.0/plugin.schema.json`. Its Skill can be checked with the Agent Skills validator used by Codex:

```sh
python3 /Users/jelf/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/openconnector
```

Validate the Claude Code marketplace and plugin:

```sh
claude plugin validate . --strict
```
