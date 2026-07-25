# Open Connector Agent Plugins

Installable agent plugins for Open Connector.

The first release provides a Codex plugin containing a single Skill. The Skill guides an agent to install and use the first-party `oc` CLI; it does not bundle MCP servers, hooks, apps, or runtime code.

## Local development

```sh
codex plugin marketplace add .
codex plugin add openconnector@openconnector
```

Start a new Codex task after installing or updating the plugin so its Skill is loaded.
