# `gwsa`

Thin zsh wrapper that gives the single-account `gws` CLI multi-account
support, by routing each invocation through a per-account
`GOOGLE_WORKSPACE_CLI_CONFIG_DIR`. See `README.md` for full setup.

## Layout to keep in mind

- `gwsa` — the script. Symlinked from `~/.local/bin/gwsa`. Resolves its own
  physical path (`${(%):-%x}:A:h`), so renaming this folder doesn't break it —
  only the symlink target would need updating.
- `credentials/<account>/` — one per logged-in account (currently `personal`
  and `work`). Each holds a symlink to `../../client_secret.json` plus the
  `credentials.enc` blob written by `gws auth login`.
- `client_secret.json` — OAuth Desktop client JSON. **Gitignored. Treat as
  secret.** All accounts share this single file.

## Command surface

```
gwsa <account> <gws args...>     # forwards to gws under that account's config dir
gwsa setup <account> [args...]   # mkdir, symlink client_secret, run `gws auth login`
gwsa accounts                    # list directories under credentials/
gwsa whoami <account>            # `gws auth status` under that config dir
```

Account names are free-form; anything matching `credentials/<name>/` works.

## Don'ts

- **Never** `cat`, `Read`, or otherwise surface the contents of
  `client_secret.json`, `credentials/*/credentials.enc`, or
  `credentials/*/token_cache.json` into the conversation. They are secrets.
- **Never** commit them — `.gitignore` already covers all four patterns, but
  double-check before any `git add`.
- Don't pipe `gws` output through `2>&1 | jq` — `gws` writes `Using keyring
  backend: keyring` on stderr, which corrupts the JSON. Stderr goes to the
  terminal, stdout is clean JSON.

## Known footguns (saw these during setup, 2026-05-12)

- **`403: Caller does not have required permission to use project <id>`** on
  the first API call from any account that doesn't own the Cloud project.
  OAuth login succeeds, the API call fails. Fix: grant that account
  **Editor** or **Service Usage Consumer** on the project's IAM page (the
  error includes the direct URL). No re-login needed. This is why the README's
  Google Cloud setup includes step 5.
- **Keychain key is shared across `gws` invocations on this Mac user.** The
  entry is `("gws-cli", <os-username>)`, not per-config-dir. Isolation of
  *tokens* holds (each `credentials.enc` only holds one account's refresh
  token), but if that single keychain entry is deleted/rotated, *both*
  accounts have to re-login.

## Verified working (2026-05-12)

End-to-end `gmail users getProfile` works for both configured accounts, and
`gmail users threads get format=full` returns full message bodies — which is
the gap that motivated this project versus the Gmail MCP server.
