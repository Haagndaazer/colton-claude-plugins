# coltondyck — Claude Code plugin marketplace

A single Claude Code marketplace (`coltondyck`) that indexes Colton Dyck's plugins.
This repo is **index-only**: it contains just `.claude-plugin/marketplace.json`. Each
plugin's code lives in its own repository and is referenced here by URL + pinned SHA.

## Plugins

| Plugin | What it does | Source |
|--------|--------------|--------|
| [`vibe-cognition`](https://github.com/Haagndaazer/vibe-cognition) | Project knowledge graph — decisions, failures, discoveries, patterns, and reasoning chains. Fully local, no API keys. | `Haagndaazer/vibe-cognition` |
| [`teammate-comms`](https://github.com/Haagndaazer/teammate-comms) | Agent-to-agent messaging + channel idle-wake for full Claude Code instances. | `Haagndaazer/teammate-comms` |

## Install

```
/plugin marketplace add Haagndaazer/colton-claude-plugins
/plugin install vibe-cognition@coltondyck
/plugin install teammate-comms@coltondyck
```

> `teammate-comms` is a channel plugin: launch with
> `--dangerously-load-development-channels` to enable its idle-wake channel (custom
> channels are not on Anthropic's allowlist).

## Migrating from the old marketplaces

These plugins were previously published from two separate marketplaces —
`coltondyck` (sourced from the vibe-cognition repo) and `colton-comms` (sourced from
the teammate-comms repo). This repo replaces both. Because it also declares
`name: "coltondyck"`, remove the old registrations first to avoid a name clash and a
duplicate `teammate-comms` install:

```
/plugin marketplace remove coltondyck
/plugin marketplace remove colton-comms
/plugin marketplace add Haagndaazer/colton-claude-plugins
/plugin install vibe-cognition@coltondyck
/plugin install teammate-comms@coltondyck
```

Old cached plugin versions under `~/.claude/plugins/cache/` are inert and can be
deleted by hand if you want to reclaim the space.

## Releasing a new plugin version

Plugins are pinned by commit SHA. To publish an update: push the plugin's own repo,
then bump its `source.sha` in `.claude-plugin/marketplace.json` here and push.
