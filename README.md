# gwsa — multi-account wrapper for `gws`

The [Google Workspace CLI](https://github.com/googleworkspace/cli) (`gws`) is
single-account by design: one `~/.config/gws/credentials.enc`. `gwsa` is a
~70-line zsh wrapper that keeps multiple independent credential stores
side-by-side under `./credentials/<account>/` and routes each invocation to
the right one via `GOOGLE_WORKSPACE_CLI_CONFIG_DIR`.

## Layout

```
gwsa/
├── gwsa                            # wrapper script (this is what you symlink onto $PATH)
├── client_secret.json              # OAuth Desktop client JSON (gitignored)
└── credentials/
    ├── personal/
    │   ├── client_secret.json        (symlink → ../../client_secret.json)
    │   └── credentials.enc           (written by `gws auth login`)
    └── work/
        ├── client_secret.json
        └── credentials.enc
```

The folder can be renamed; the script resolves its own physical path and finds
the sibling `credentials/` directory regardless. Only the `~/.local/bin/gwsa`
symlink needs updating on rename.

## One-time Google Cloud setup

1. Create a project at <https://console.cloud.google.com>.
2. **Google Auth Platform** (formerly OAuth consent screen) → User type:
   **External**. On **Branding**, fill in the app name, support email,
   developer contact, a home page URL and a privacy policy URL. Both URLs
   must be on a domain listed under **Authorized domains** (e.g.
   `example.com`). Don't upload a logo, because a logo forces
   verification. Then go to **Audience → Publish app** (**In production**).
   Don't leave it in **Testing**: Google expires Testing refresh tokens after
   7 days. Unverified is fine for personal use. Login shows "Google hasn't
   verified this app"; click **Advanced → Go to**.
3. **Enabled APIs**: Gmail, Drive, Calendar (plus anything else you want
   `gws` to call).
4. **Credentials → Create OAuth client → Desktop app** → download the JSON →
   save it as `./client_secret.json` in this folder.
5. **IAM → Grant access** to every Google account *other than the project
   owner* that will use this OAuth client. Give them role **Editor** (or, more
   tightly, **Service Usage Consumer**). Without this, the OAuth flow still
   succeeds but every API call returns
   `403: Caller does not have required permission to use project <id>`. See
   the troubleshooting section below.

## Install

```sh
mkdir -p ~/.local/bin
ln -s "$(pwd)/gwsa" ~/.local/bin/gwsa
```

Make sure `~/.local/bin` is on `$PATH`. `gws` itself must also be installed
and on `$PATH` — on macOS: `brew install googleworkspace-cli`.

## Set up an account

```sh
gwsa setup personal -s drive,gmail,calendar
gwsa setup work     -s drive,gmail,calendar
```

Each runs `gws auth login` against its own config dir. A browser tab opens —
pick the account that matches the slot's name (`personal` → your `@gmail.com`,
`work` → your work address) and approve. `credentials/<account>/credentials.enc`
is written when the flow completes.

Account names are free-form — anything matching `credentials/<name>/`
becomes a valid first argument to `gwsa`.

## Daily usage

```sh
gwsa personal gmail +triage
gwsa work     calendar +agenda
gwsa work     gmail users messages get --params '{"userId":"me","id":"..."}'
gwsa personal gmail users threads  get --params '{"userId":"me","id":"...","format":"full"}'
```

Other subcommands:

```sh
gwsa accounts           # list configured accounts
gwsa whoami personal    # runs `gws auth status` under the personal config dir
```

Any first argument that isn't `setup`, `accounts`, or `whoami` is treated as
an account name; the rest of the args go straight to `gws`.

## Encryption / Keychain caveat

`gws` encrypts `credentials.enc` with AES-256-GCM using a key stored in the
macOS Keychain. The Keychain entry is **`("gws-cli", <os-username>)` — a single
entry shared across all `gws` invocations on this macOS user**, regardless of
`GOOGLE_WORKSPACE_CLI_CONFIG_DIR`. The key is generated only when one doesn't
already exist, so the second `auth login` reuses the first's key. Both
`credentials.enc` blobs end up decryptable with the same AES key.

Practical consequences:

- **Isolation of the OAuth tokens still holds** — each account's tokens live
  in its own `credentials.enc` under its own config dir, and `gwsa` only ever
  hands `gws` one of them at a time.
- **The folder is portable on this Mac, but not across machines** — the key
  lives in this Mac's Keychain.
- **If that Keychain entry is ever deleted or regenerated** (manual cleanup,
  a future `gws` upgrade that rotates it), *both* accounts have to be
  re-logged-in. Don't be surprised by this.

If you ever want a fully self-contained folder, set
`GOOGLE_WORKSPACE_CLI_KEYRING_BACKEND=file` — but that puts the AES key on
disk next to the ciphertext, which defeats the encryption. Not recommended.

## Troubleshooting

### `403: Caller does not have required permission to use project <id>`

OAuth login succeeded (token + scopes are fine), but the account making the
API call has no IAM role on the Cloud project that owns the OAuth client.
Google bills/quotas API usage against that project, and non-owner principals
need `serviceusage.services.use` to consume it.

Fix: open the project's **IAM** page (the error message includes the direct
URL), grant the offending account **Editor** or **Service Usage Consumer**,
wait ~30s for propagation, retry. No re-login needed — the existing token
keeps working.

This typically shows up the first time you use the `work` slot, since the
Cloud project is usually owned by your personal Google account.

### `Using keyring backend: keyring` on stderr

`gws` prints `Using keyring backend: keyring` on **stderr** before its JSON
result, so piping `gwsa ... | jq` works as long as you don't redirect stderr
into stdout (`2>&1`).

## Coexistence with the existing Claude MCP integrations

The Gmail / Drive / Calendar MCP servers in Claude (personal account) are
unaffected — they use their own OAuth flow and credential storage. `gwsa` is
purely additive: use it when you need work-account access or when you need a
`gws` feature the MCP doesn't expose (e.g. full message body of a threaded
email).
