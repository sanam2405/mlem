# Rewriting Git Author, Sign-off & GPG Signature

A practical guide to rewriting an existing commit history so every commit carries a different identity and is re-signed with a different GPG key — while keeping commit messages and timestamps byte-for-byte intact. Based on a real migration: a repo committed from a work laptop (company email, company GPG key) re-attributed to a personal identity on a new machine.

---

## Table of Contents

1. [The Problem](#1-the-problem)
2. [The Four Places Identity Hides in a Commit](#2-the-four-places-identity-hides-in-a-commit)
3. [What "Keep Timestamps Intact" Actually Means](#3-what-keep-timestamps-intact-actually-means)
4. [Choosing a Rewrite Tool](#4-choosing-a-rewrite-tool)
5. [The Pinentry / TTY Problem](#5-the-pinentry--tty-problem)
6. [The Rewrite, Step by Step](#6-the-rewrite-step-by-step)
7. [The Sign-off Trailer Gotcha](#7-the-sign-off-trailer-gotcha)
8. [Publishing: Force-Push Safely](#8-publishing-force-push-safely)
9. [GitHub's "Verified" Badge](#9-githubs-verified-badge)
10. [Cleanup & Recovery](#10-cleanup--recovery)
11. [Preventing a Repeat](#11-preventing-a-repeat)
12. [Quick Reference Cheat Sheet](#12-quick-reference-cheat-sheet)

---

## 1. The Problem

You committed to a repo with the wrong identity, and now you want to fix the whole history. Concretely, the symptoms look like this:

```
$ git log --pretty=format:'%h | A:%ae | C:%ce | sig:%G?'
f5e5278 | A:manaspb@jurisphere.ai | C:manaspb@jurisphere.ai | sig:E
cb26ab6 | A:manaspb@jurisphere.ai | C:manaspb@jurisphere.ai | sig:E
...
```

Two things are wrong:

- **`%ae` / `%ce`** — author and committer email are the *company* address, not your personal one.
- **`sig:E`** — the signature can't be verified because it was made with a GPG key that doesn't exist on this machine (`gpg: Can't check signature: No public key`).

The goal: rewrite every commit so author, committer, sign-off, and GPG signature all use the personal identity — **without** touching commit messages or timestamps.

**Why this is fiddly**: a commit's identity is not stored in one place. There are four, and a naive rewrite fixes only two of them.

---

## 2. The Four Places Identity Hides in a Commit

This is the single most important thing to understand before rewriting anything. The same person's email can appear in **four independent locations** in one commit, and they do not update together.

| # | Location | Stored as | Shown by | Set by |
|---|----------|-----------|----------|--------|
| 1 | **Author** | `author` header | `%an <%ae>` | `--author`, `user.name/email` |
| 2 | **Committer** | `committer` header | `%cn <%ce>` | `user.name/email` at commit time |
| 3 | **Sign-off trailer** | text *inside the message body* | `git log` body | the `-s` / `--signoff` flag |
| 4 | **GPG signature** | `gpgsig` header (binary) | `%G?`, `--show-signature` | the `-S` flag + `user.signingkey` |

```
commit <sha>
  author    Name <EMAIL_1>   ← (1) author header
  committer Name <EMAIL_2>   ← (2) committer header
  gpgsig    -----BEGIN PGP SIGNATURE-----   ← (4) signed over the whole commit
            ...

      fix: rename root directory label

      Signed-off-by: Name <EMAIL_3>   ← (3) lives in the message text
```

**The trap**: the common one-liner rewrites only update headers (1) and (2). If you used `git commit -s`, location (3) is plain text *inside the message* and survives untouched — so `git log` still shows the old email in the `Signed-off-by:` line even though the headers look correct. And because re-writing the commit object invalidates the old signature, (4) must be regenerated regardless.

> **In practice**: if your team signs off with `-s` (very common in projects that enforce the [DCO](https://developercertificate.org/)), budget for a *second* rewrite pass that edits the message body. We missed this on the first pass and had to redo it.

---

## 3. What "Keep Timestamps Intact" Actually Means

There are **two** timestamps per commit, and they are not the same:

- **Author date** (`%ai` / `%aI`) — when the work was originally authored.
- **Committer date** (`%ci` / `%cI`) — when the commit object was last written.

Most of the time they're equal, but they diverge whenever a commit is amended, rebased, or cherry-picked. In our real history, one commit had:

```
A:2026-05-22T00:32:25  C:2026-05-22T00:36:34   ← authored, then re-committed 4 min later
```

"Keep timestamps intact" means preserving **both** dates on **every** commit. This is exactly where the popular approaches fall down:

| Approach | Author date | Committer date |
|----------|-------------|----------------|
| `git rebase --exec '--amend'` | preserved | **reset to now** |
| `git rebase --committer-date-is-author-date` | preserved | **collapsed onto author date** |
| `git filter-repo` | preserved | preserved, but **strips signatures** |
| `git filter-branch` + `commit-tree -S` | preserved | preserved | 

The reason `filter-branch` wins here: it exports the original `GIT_AUTHOR_DATE` and `GIT_COMMITTER_DATE` into the environment before running your filters, and `git commit-tree` honors both env vars. So the rewritten commit inherits the exact original timestamps for free.

---

## 4. Choosing a Rewrite Tool

```
Need to change email AND re-sign AND keep both timestamps?
   │
   ├── Just changing email, don't care about signing?
   │       └── git filter-repo --email-callback   (modern, fast, safe default)
   │
   ├── Re-signing too, want it in ONE pass with timestamps preserved?
   │       └── git filter-branch + --commit-filter 'git commit-tree -S "$@"'
   │
   └── Only a few commits at the tip, signatures don't matter?
           └── git rebase -i --root  (manual, error-prone for dates)
```

### Why not `git filter-repo`?

It's the officially recommended tool and it's genuinely better for the email change — but it **drops GPG signatures** (the commit content changes, so old signatures are invalid, and it does not re-sign). You'd need a second signing pass, which reintroduces the committer-date problem from §3. It also **removes the `origin` remote** by design, so you'd re-add it.

### Why `git filter-branch` here

It's deprecated and slower, but it does all three things we need in a single pass:

- `--env-filter` overwrites the author/committer identity,
- `--commit-filter 'git commit-tree -S "$@"'` re-signs with the default signing key,
- and the original timestamps come along automatically (§3).

> **In practice**: pick `filter-repo` for routine email scrubbing across a huge history. Pick `filter-branch` with `commit-tree -S` for the specific "re-attribute *and* re-sign a small/medium history with timestamps preserved" job described here.

---

## 5. The Pinentry / TTY Problem

If your GPG key is passphrase-protected (it should be), the rewrite will try to sign each commit and immediately hit this:

```
error: gpg failed to sign the data:
[GNUPG:] NEED_PASSPHRASE ...
gpg: cannot open '/dev/tty': Device not configured
could not write rewritten commit
```

The cause: the rewrite runs in a **non-interactive shell with no controlling TTY**, so `gpg` can't open a prompt. This also bites any automated/agent-driven shell, and even editor "run command" panes that capture output instead of allocating a real terminal.

**The fix is to pre-warm `gpg-agent`'s passphrase cache from a real terminal**, then run the rewrite while the cache is hot. `gpg-agent` is a per-user daemon, so an unlock in *any* real terminal is visible to the rewrite process.

1. Open a real terminal (e.g. **Terminal.app**, not a captured subshell).
2. Force a signing operation so the `pinentry` dialog appears and caches the passphrase:

   ```bash
   echo precache | gpg --clearsign -u <YOUR_KEY_ID> > /dev/null && echo CACHED
   ```

3. Enter the passphrase (tick "Save to Keychain" with `pinentry-mac` to avoid future prompts).
4. Run the rewrite within the cache window (default `default-cache-ttl` is 600s).

> **In practice**: never paste your passphrase into a script, a chat, or a command flag — it lands in shell history and logs. Unlock the agent interactively and let the cache do the rest. If even a real terminal errors on `/dev/tty`, your shell isn't exporting `GPG_TTY` — add `export GPG_TTY=$(tty)` to your `~/.zshrc`.

---

## 6. The Rewrite, Step by Step

### Step 0 — Survey the damage

```bash
git log --pretty=format:'%h | A:%an <%ae> %aI | C:%cn <%ce> %cI | sig:%G?'
git log --show-signature -1          # confirm which key signed it
git config user.email                # confirm THIS machine's target identity
git config user.signingkey           # confirm the key to re-sign with exists
gpg --list-secret-keys --keyid-format=long   # confirm the secret key is present
```

### Step 1 — Make a backup

A rewrite changes every SHA. Keep an escape hatch:

```bash
git branch backup-pre-rewrite
```

### Step 2 — Rewrite identity + re-sign (single pass)

```bash
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch -f \
  --env-filter '
    export GIT_AUTHOR_EMAIL="you@personal.com"
    export GIT_COMMITTER_EMAIL="you@personal.com"
    export GIT_AUTHOR_NAME="Your Name"
    export GIT_COMMITTER_NAME="Your Name"
  ' \
  --commit-filter 'git commit-tree -S "$@"' \
  --tag-name-filter cat -- main
```

- `--env-filter` rewrites the author/committer headers (locations 1 & 2 from §2).
- `--commit-filter 'git commit-tree -S "$@"'` replaces the default unsigned `commit-tree` with a **signed** one. `-S` uses `user.signingkey` (location 4).
- `--tag-name-filter cat` keeps tag names pointing at rewritten commits.
- `-- main` scopes the rewrite to one branch. **Avoid `-- --all`** unless you intend it — it will also rewrite your backup branch (the originals are still recoverable via `refs/original`, but it defeats the point of a clean backup).

### Step 3 — Verify everything at once

```bash
git log --pretty=format:'%h | A:%ae %aI | C:%ce %cI | sig:%G?'
```

You want, on every line: the personal email in both `%ae` and `%ce`, the original timestamps unchanged, and **`sig:G`** (good signature) instead of `sig:E`.

```
204bd0b | A:you@personal.com 2026-05-22T01:36:50 | C:you@personal.com 2026-05-22T01:36:50 | sig:G
f686329 | A:you@personal.com 2026-05-22T00:32:25 | C:you@personal.com 2026-05-22T00:36:34 | sig:G  ← split dates preserved
```

**Signature status codes** (`%G?`):

| Code | Meaning |
|------|---------|
| `G` | Good (valid) signature |
| `E` | Can't check — **missing public key** (the original symptom) |
| `B` | Bad signature |
| `U` | Good signature, unknown validity |
| `N` | No signature |

---

## 7. The Sign-off Trailer Gotcha

After §6, the headers and signature are correct — but if the project uses `git commit -s`, `git log` *still* shows the old email:

```
fix: rename root directory label

Signed-off-by: Your Name <you@jurisphere.ai>   ← still the company email!
```

That `Signed-off-by:` line is location (3) from §2 — **plain text in the message body**, which `--env-filter` never touches. Fix it with a second pass that rewrites the message *and* re-signs (the message change invalidates the signature again):

```bash
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch -f \
  --msg-filter 'sed "s/you@jurisphere\.ai/you@personal.com/g"' \
  --commit-filter 'git commit-tree -S "$@"' \
  --tag-name-filter cat -- main
```

- `--msg-filter` pipes each commit message through `sed`, replacing the old email everywhere it appears in the body.
- `--commit-filter ... -S` re-signs the now-modified commits.

Verify there are zero stragglers:

```bash
git log --format='%B' | grep -c 'you@jurisphere.ai'   # want: 0
git log --format='%h trailer:%(trailers:key=Signed-off-by,valueonly)'
```

> **In practice**: you *could* combine §6 and §7 into one pass (add `--msg-filter` to the first command). We split them only because the trailer was discovered after the fact. One combined pass is cleaner if you know up front that `-s` was used.

---

## 8. Publishing: Force-Push Safely

A rewrite means the remote and local histories have diverged with no common tip — a normal push is rejected, and you must force. Use `--force-with-lease`, never a bare `--force`:

```bash
LEASE=$(git rev-parse origin/main)   # the SHA you believe is on the server
git push --force-with-lease=main:$LEASE origin main
```

`--force-with-lease=<branch>:<expected-sha>` refuses the push if the server's tip isn't `<expected-sha>` — protecting you from clobbering work someone else pushed in the meantime. A bare `--force` has no such guard.

```
Before:  origin/main ──> f5e5278 (old, company-signed)
Local:   main ─────────> 204bd0b (rewritten, personal-signed)
Push:    + f5e5278...204bd0b main -> main (forced update)
After:   origin/main ──> 204bd0b ✓
```

> **In practice**: a prior `filter-branch` run can leave your local `refs/remotes/origin/main` pointing at a *rewritten* SHA that the server never saw, which makes `--force-with-lease` behave unexpectedly. Run `git fetch` first to resync the remote-tracking ref, or pin the lease to the actual server tip from `git ls-remote origin -h refs/heads/main`.

---

## 9. GitHub's "Verified" Badge

A local `sig:G` only means the signature verifies against *your* keyring. For GitHub to show the green **Verified** badge, two more things must be true:

1. The **committer email** is a *verified email* on your GitHub account (Settings → Emails).
2. The **public half** of your signing key is uploaded to GitHub (Settings → SSH and GPG keys).

Export the public key to upload it:

```bash
gpg --armor --export <YOUR_KEY_ID>
```

If commits show **Unverified** after the push, it's almost always one of those two — not a problem with the rewrite itself.

---

## 10. Cleanup & Recovery

`filter-branch` stashes the pre-rewrite refs under `refs/original/` as a safety net. Once you've confirmed the remote looks right, remove them and the manual backup:

```bash
git branch -D backup-pre-rewrite
git update-ref -d refs/original/refs/heads/main
# (delete any other refs/original/* entries filter-branch created)
git for-each-ref refs/original   # should print nothing when fully clean
```

### If something went wrong

As long as you haven't run garbage collection, the original commits are still reachable:

```bash
git reflog                          # find the pre-rewrite SHA
git reset --hard <old-sha>          # restore local branch
# or recover from the filter-branch backup:
git reset --hard refs/original/refs/heads/main
```

After cleanup, the orphaned old commits are eventually removed by `git gc` (or force it with `git gc --prune=now` once you're certain).

---

## 11. Preventing a Repeat

The root cause was a machine whose default git identity didn't match the repo. Two durable fixes:

### Per-repo override

```bash
git config user.email "you@personal.com"
git config user.signingkey "<YOUR_KEY_ID>"
```

### Conditional includes (global, automatic)

Route identity by directory so you never have to think about it. In `~/.gitconfig`:

```ini
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work
[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

Then `~/.gitconfig-personal`:

```ini
[user]
    email = you@personal.com
    name = Your Name
    signingkey = <YOUR_KEY_ID>
[commit]
    gpgsign = true
```

> **In practice**: `includeIf` is the cleanest long-term fix on a machine that holds both work and personal repos — the right identity and signing key are selected automatically based on where the repo lives.

---

## 12. Quick Reference Cheat Sheet

### The two-pass rewrite (identity + sign-off + re-sign, timestamps preserved)

```bash
# 0. cache the GPG passphrase in a REAL terminal first (see §5)
echo x | gpg --clearsign -u <KEY> >/dev/null

# 1. backup
git branch backup-pre-rewrite

# 2. headers + signature
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch -f \
  --env-filter 'export GIT_AUTHOR_EMAIL="new@e.com" GIT_COMMITTER_EMAIL="new@e.com" \
                       GIT_AUTHOR_NAME="Name" GIT_COMMITTER_NAME="Name"' \
  --commit-filter 'git commit-tree -S "$@"' \
  --tag-name-filter cat -- main

# 3. sign-off trailer in the message body (only if `-s` was used)
FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch -f \
  --msg-filter 'sed "s/old@e.com/new@e.com/g"' \
  --commit-filter 'git commit-tree -S "$@"' \
  --tag-name-filter cat -- main

# 4. verify
git log --pretty=format:'%h | A:%ae %aI | C:%ce %cI | sig:%G?'
git log --format='%B' | grep -c 'old@e.com'    # want 0

# 5. publish
git fetch
git push --force-with-lease=main:$(git rev-parse origin/main) origin main

# 6. cleanup once confirmed
git branch -D backup-pre-rewrite
git update-ref -d refs/original/refs/heads/main
```

### Checklist before you start

- [ ] Target machine's `user.email` / `user.name` / `user.signingkey` are correct
- [ ] The signing **secret key** is present (`gpg --list-secret-keys`)
- [ ] GPG passphrase is cached in `gpg-agent` (warmed from a real terminal)
- [ ] Backup branch created
- [ ] Know whether `-s` (sign-off) was used → decide one-pass vs two-pass
- [ ] Public key uploaded to GitHub + committer email verified there

### Red flags

| Signal | Meaning |
|--------|---------|
| `sig:E` in `git log` | Public key for the signing key is missing locally |
| `Signed-off-by:` still shows old email | You forgot the `--msg-filter` pass (§7) |
| Committer date jumped to "now" | You used `rebase --amend` instead of `commit-tree` (§3) |
| GitHub shows **Unverified** | Key not uploaded, or committer email unverified (§9) |
| `cannot open '/dev/tty'` | No TTY for `pinentry` — warm the agent cache (§5) |
| `--force-with-lease` rejected | Remote-tracking ref is stale — `git fetch` first (§8) |

---

## Further Reading

- [git-filter-branch documentation](https://git-scm.com/docs/git-filter-branch)
- [git filter-repo (the modern replacement)](https://github.com/newren/git-filter-repo)
- [Pro Git — Rewriting History](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History)
- [GitHub Docs — Signing commits & the Verified badge](https://docs.github.com/en/authentication/managing-commit-signature-verification)
- [git-commit-tree (the `-S` signing primitive)](https://git-scm.com/docs/git-commit-tree)
- [Conditional includes (`includeIf`)](https://git-scm.com/docs/git-config#_conditional_includes)
- [Developer Certificate of Origin (the `-s` sign-off)](https://developercertificate.org/)
