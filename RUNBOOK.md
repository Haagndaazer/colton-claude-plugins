# Marketplace release runbook — pinning a plugin version

This repo is a **Claude Code plugin marketplace**. It does not cut releases of its own;
it **pins** other repos' releases. The single source of truth is
[`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json): each entry in the
`plugins` array has a `source.url` (the plugin repo's `.git`) and a `source.sha` — a full
**40-char commit SHA**. That SHA **is** the release pointer. There is no version field and
no git tag; the version is whatever the plugin repo declares at that commit. A bad or partial
SHA here breaks installs for **every** user, so the verify steps below are not optional.

## Roles

- **Plugin maintainers** cut versions on their own plugin repos (bump version, regenerate
  lockfile, green tests, push to the plugin repo's `main`).
- **The marketplace owner** does the pin: edits `marketplace.json` and pushes it here.

## Auth prerequisites

- A `gh` CLI authenticated with a token that has **`repo` scope on the marketplace remote**
  (push access to `origin/main`), used both for pushing and for reading plugin repos via the
  GitHub API.

## Standing rule

**Only pin + push to `main` when Colton explicitly says go.** Never self-initiate a publish.
Editing and committing the pin is fine; the *push* is the gated action.

## Runbook

**(a) Get the verified HEAD — never trust a relayed short SHA.**
A teammate relaying "the head is `abc1234`" has been wrong before (the repo moved one commit
further). Resolve it yourself and match the full 40-char value to the default branch:

```
git ls-remote https://github.com/<owner>/<plugin>.git refs/heads/main
```

Use the SHA on the `refs/heads/main` line — not a tag, not a PR ref.

**(b) Confirm the NEW version at that ref, across the plugin's authoritative version files.**
The point is to prove the maintainer actually bumped (the version at HEAD should be the *new*
version). Read each file at the exact SHA (the `raw` Accept header skips the base64 decode):

```
gh api -H "Accept: application/vnd.github.raw" repos/<owner>/<plugin>/contents/<path>?ref=<sha> | grep -i version
```

Which files are authoritative differs per plugin — do **not** treat a mismatch in a
non-authoritative file as a blocker:

| Plugin          | Authoritative version files                                          |
|-----------------|----------------------------------------------------------------------|
| vibe-cognition  | `.claude-plugin/plugin.json` + `pyproject.toml`. (Its `src/vibe_cognition/__init__.py` is a static `0.1.0` placeholder — **ignore it**.) |
| teammate-comms  | `.claude-plugin/plugin.json` + `pyproject.toml` + `src/teammate_comms/__init__.py` (all three must agree). |

`plugin.json` is the version Claude Code surfaces to users, so it is always authoritative.
Optionally sanity-check the diff is a clean fast-forward off the currently-pinned SHA:

```
gh api repos/<owner>/<plugin>/compare/<currentlyPinnedSha>...<headSha> --jq '{ahead:.ahead_by, behind:.behind_by}'
```

Want `behind = 0` (head is a clean continuation, not divergent).

**(c) Edit `marketplace.json`** — swap only that plugin's `source.sha` to the full 40-char HEAD.

**(d) Validate the JSON** before committing (it's hand-edited — a trailing comma or truncated
SHA is the most likely real failure):

```
python -c "import json,sys; json.load(open('.claude-plugin/marketplace.json')); print('ok')"
```

**(e) Stage only the one file and commit — then HOLD.** Do not `git add -A` (the working tree
may carry untracked files like `.mcp.json`):

```
git add .claude-plugin/marketplace.json
git commit -m "Pin <plugin> v<x.y.z> (<short-sha>) — <one-line of what's in it>"
```

**One pin per commit / push.** Never bundle two plugins' pins into a single commit or push —
keep each publish isolated so it can be reverted on its own. (If a second plugin needs pinning,
it's a separate commit and, after a separate go, a separate push.)

Stop here. **Wait for Colton's explicit go** before pushing.

**(f) After the go: push and verify `local == remote`.**

```
git push origin main
git rev-parse HEAD
git ls-remote origin -h refs/heads/main   # the two SHAs must match
```

(A CRLF/LF warning on Windows is benign.)

**(g) Close the loop.** Reply to whoever requested the pin with the pinned SHA + the marketplace
commit SHA, and confirm it's live.

## After the push — how users receive it, and the known failure mode

Users pick up the new pin the next time they **update the plugin**; the pin itself is live the
moment the push verifies. The known failure mode is on the user's side, on Windows: updating a
plugin **while its MCP server is still running** throws an `EPERM` rename in the plugin cache
(the running process holds a lock on the cache dir). Unblock:

1. Kill the plugin's server process(es).
2. Delete the stale plugin cache dir and any leftover `temp_git_*` dirs.
3. Reinstall / re-update the plugin.

This is a user-update hazard, not a pin defect — but it's the first thing to suggest if someone
reports a pinned update "won't install."

## Why each guard exists

- **(a)** Relayed short SHAs have been stale before — the marketplace must point at the real HEAD.
- **(b)** The version table prevents the obvious next-agent mistake of stopping on a benign
  `__init__.py` mismatch, and confirms the maintainer bumped before you pin.
- **(d)** `marketplace.json` is the only file that gates what users install; malformed JSON or a
  truncated SHA breaks installs for everyone.
- **(f)** A push isn't done until the remote actually carries it — verify, don't assume.
