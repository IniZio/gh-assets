---
name: github-assets-upload
description: Upload images, diagrams, videos, or files to GitHub's user-attachments store and embed the returned markdown into a PR, issue, or comment. Use when asked to attach a screenshot, embed an image, or upload a file to GitHub from the CLI. Trigger keywords — attach screenshot, embed image, upload to PR, add screenshot to issue, gh assets.
---

# GitHub asset upload (gh-assets)

Upload files to GitHub and embed them where they render inline.

## Preconditions

1. Extension installed: `gh extension list | grep -q gh-assets || gh extension install IniZio/gh-assets`
2. A valid `user_session` token is resolvable. Verify: `gh assets check-token`
   (prints the login on success). If it fails, tell the user to set one:
   `gh config set -h github.com gh-assets.token "<user_session cookie value>"`
   (DevTools → Application → Cookies → github.com → `user_session`).

## Upload and embed

```bash
# Capture the markdown reference the tool prints
REF=$(gh assets upload screenshot.png --repo <owner>/<repo>)

# Embed into a PR body / comment
gh pr comment <n> --repo <owner>/<repo> --body "$REF"
```

- Multiple files: `gh assets upload a.png b.png c.svg` — one markdown line each.
- Output is ready to paste: `![name](url)` (images), bare URL (video → inline
  player), `[name](url)` (other files).
- `--repo` sets the attachment's visibility scope; omit it inside a git
  workspace to infer from the remote.

## Rules

- Never commit screenshots into the repo to reference them — that bloats git
  history. Use `gh assets` so they live in user-attachments.
- Pass `--repo <target>` when uploading for a cross-fork PR so the attachment is
  scoped to the upstream repo the reviewers can see, not your fork.
- A `user_session` cookie is account-wide credentials — never echo the token
  value into logs, PR bodies, or commits.
