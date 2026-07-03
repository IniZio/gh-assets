# gh-assets

> Upload screenshots, diagrams, and files to GitHub straight from the CLI — and
> get back a ready-to-paste markdown reference.

GitHub has **no public API** for the attachment uploads its web UI accepts via
drag-and-drop. That internal flow produces `user-attachments` URLs whose
visibility is scoped to the repository they were uploaded to. `gh-assets`
replays that flow as a `gh` CLI extension, so you can drop a screenshot — or any
GitHub-supported file (PDF, zip, log, video) — into a PR body, issue, or comment
without leaving the terminal. Images embed inline, videos render as players,
other files become download links, and uploads on private repos stay private.

It is a single, auditable Bash script — no compiled binary, no release
pipeline. `curl` + `jq` + `gh` are the only dependencies.

```console
$ gh assets upload screenshot.png
![screenshot.png](https://github.com/user-attachments/assets/…)

$ gh assets upload report.pdf
[report.pdf](https://github.com/user-attachments/files/…/report.pdf)
```

## Install

```bash
gh extension install IniZio/gh-assets
```

Requires `gh`, `curl`, and `jq` on `PATH`.

## Usage

```bash
# Repo inferred from the current git workspace
gh assets upload screenshot.png

# Multiple files at once
gh assets upload before.png after.png diagram.svg

# Target a specific repo (sets attachment visibility scope)
gh assets upload shot.png --repo oursky/hanlun-lms

# Pipe straight into a PR / issue body
gh pr comment 123 --body "Repro:

$(gh assets upload bug.png)"
```

Each successful upload prints one ready-to-paste line: `![name](url)` for
images, a bare URL for videos (GitHub renders these as an inline player when the
URL is on its own line), and `[name](url)` for everything else. If one file in a
batch fails, its error goes to stderr and the process exits non-zero — the rest
still upload.

## The upload flow

`gh-assets` performs the same four steps the browser does:

1. **uploadToken** — `GET github.com/<owner>/<repo>`, scrape `"uploadToken":"…"`
   from the page (proves write access).
2. **policy** — `POST /upload/policies/assets` with the file metadata → a
   presigned S3 form + an `asset_upload_url`.
3. **S3** — `POST` the file to the presigned bucket (no GitHub cookies).
4. **finalize** — `PUT` the `asset_upload_url` → the canonical
   `user-attachments` href.

## Authentication

The upload endpoints authenticate with a GitHub **`user_session` cookie**, not a
PAT (personal access tokens can't reach this internal endpoint). Resolution
order — first match wins:

| Priority | Source | Use for |
|---|---|---|
| 1 | `--token <value>` flag | one-off (visible in `ps` — avoid on shared hosts) |
| 2 | `GH_SESSION_TOKEN` env var | CI / headless / preferred locally |
| 3 | `gh config get -h github.com gh-assets.token` | persistent local default |
| 4 | **local browser cookie store** | zero-config local use (default) |

**Zero-config (like gh-image):** if nothing above is set, `gh-assets` reads the
`github.com` `user_session` straight from your browser — **Firefox** (plaintext
`cookies.sqlite`) or the **Chromium family** (Chrome/Brave/Edge; AES-decrypted
via the GNOME keyring, needs `python3`+`cryptography` and `secret-tool`). Set
`GH_ASSETS_NO_BROWSER=1` to disable.

> [!NOTE]
> Browser extraction only works where the browser lives. On a **headless or
> remote dev host** there is no cookie store — set `GH_SESSION_TOKEN` (or store
> it in `gh config`) instead.

To set a token explicitly, get the value from a logged-in browser (**DevTools →
Application → Cookies → `github.com` → `user_session` → Value**) and store it
once:

```bash
gh config set -h github.com gh-assets.token "<user_session-value>"
gh assets check-token            # prints your login if the token is valid
```

> [!WARNING]
> A `user_session` cookie grants **full account access** — it is not scoped like
> a PAT. Treat it like a password. If it leaks, sign out at
> <https://github.com/logout> (or revoke via Settings → Sessions). For CI on a
> shared repo, extract the token from a **dedicated bot account**, not your own.

### SAML SSO

If an org enforces SAML SSO and your session isn't authorized for it, step 1
can't find the uploadToken even with write access. Authorize once in a browser
at `https://github.com/orgs/<owner>/sso` (~24h), then retry.

## Prior art

The `user-attachments` handshake here is ported from
[drogers0/gh-image](https://github.com/drogers0/gh-image) (MIT), a Go extension
that additionally extracts the session cookie directly from your browser's
encrypted store. `gh-assets` trades that convenience for a dependency-light,
single-file Bash implementation and explicit token handling.

## License

MIT
