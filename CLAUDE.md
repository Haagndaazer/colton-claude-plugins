# colton-claude-plugins

This repo is a **Claude Code plugin marketplace**. The only meaningful file is
`.claude-plugin/marketplace.json`, which **pins** each plugin to a commit in its own repo via
a full 40-char `source.sha`. That SHA is the release pointer — there is no version field and
no git tag.

## Repo invariant

**Only pin + push to `main` when Colton explicitly says go.** Editing and committing a pin is
fine; the push to the marketplace is what changes what every user installs, so it is gated on
Colton's explicit word — never self-initiated.

## Releasing a new plugin version

Follow the full procedure in **[`./RUNBOOK.md`](./RUNBOOK.md)** — verify the real HEAD, confirm
the version across the plugin's authoritative files, swap the `source.sha`, validate the JSON,
commit, hold for Colton's go, push, verify `local == remote`, and close the loop.
