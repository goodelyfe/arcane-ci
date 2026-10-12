# arcane-ci

Free GitHub-hosted CI for the `feat/secret-sources` branch of `goodelyfe/arcane`.
It is a separate repo so no workflow files end up in the Arcane branches or
the upstream pull request.

Every 6 hours (or from the Actions tab), it:

1. Syncs the fork's `main` with upstream (fast-forward only; it never
   overwrites commits of your own).
2. Rebases a copy of `feat/secret-sources` onto upstream `main`. Your real branch is
   never rewritten. A failed rebase fails the run and lists the conflicting files.
3. Builds the backend and runs the secrets tests, then the full backend suite
   (informational).
4. Type-checks and lints the frontend.

## Setup

There are no secrets stored in GitHub. The one credential the workflow needs
(a GitHub token that can update the fork's `main`) lives in Infisical. The
runner reaches your Infisical over your tailnet and logs in with GitHub's
OIDC identity, so nothing long-lived sits in this repo.

### 0. Find this repo's OIDC subject

Repos created on GitHub.com after July 15, 2026 use numeric IDs in the OIDC
`sub` claim. Get the exact prefix:

```bash
gh api repos/goodelyfe/arcane-ci/actions/oidc/customization/sub
```

Use its `sub_claim_prefix`, followed by `:ref:refs/heads/main`. Example shape:
`repo:goodelyfe@123/arcane-ci@456:ref:refs/heads/main`. Below, this is `<SUB>`.

### 1. Tailscale: let the runner join as `tag:ci`

1. In the tailnet policy, define the tag and allow it to reach only Infisical:

   ```jsonc
   "tagOwners": { "tag:ci": ["autogroup:admin"] },
   "grants": [
     // adjust dst to how your Infisical node is tagged or named
     { "src": ["tag:ci"], "dst": ["tag:infisical"], "ip": ["443"] }
   ]
   ```

2. Admin console → **Trust credentials** → **Credential** → **OpenID Connect**:
   issuer **GitHub Actions** (`https://token.actions.githubusercontent.com`),
   subject `<SUB>`, scope **auth_keys** (write) with tag `tag:ci`.
3. Copy the **Client ID** and **Audience**.

Runner nodes are ephemeral and disappear when the job ends.

### 2. Infisical: a machine identity that trusts this workflow

1. Store the GitHub token as `FORK_TOKEN` in an Infisical project, environment
   `prod`, path `/arcane-ci`. Make it a fine-grained token limited to
   `goodelyfe/arcane` with **Contents** and **Workflows** read/write only.
2. Create a machine identity (e.g. `github-arcane-ci`) with **OIDC Auth**:
   - OIDC Discovery URL and Issuer: `https://token.actions.githubusercontent.com`
   - Subject: `<SUB>`
   - Audiences: leave the default (`https://github.com/goodelyfe`)
3. Add the identity to the project with read access to `/arcane-ci` only.
4. Your Infisical server needs outbound access to
   `token.actions.githubusercontent.com` to verify tokens.

### 3. GitHub: repository variables (not secrets)

In **Settings → Secrets and variables → Actions → Variables**:

| Variable | Value |
| --- | --- |
| `TS_OAUTH_CLIENT_ID` | Client ID from step 1 |
| `TS_AUDIENCE` | Audience from step 1 |
| `INFISICAL_URL` | `https://infisical.heron-monitor.ts.net` |
| `INFISICAL_IDENTITY_ID` | the machine identity's ID |
| `INFISICAL_PROJECT_SLUG` | the project slug |

None of these are secret on their own. Make the repo **public** (free runners),
then run the workflow once from the Actions tab.

### Why this is safe in a public repo

- The workflow has no `pull_request` trigger, so outside contributors can't run
  it with your identity, and both Tailscale and Infisical only accept tokens
  whose subject is this repo's `main` branch.
- Only the sync job gets `id-token: write`; the test jobs have no credentials.
- The tailnet grant lets `tag:ci` reach Infisical on 443 and nothing else.

## Getting the branch to GitHub

Make one `git push` update both Forgejo and GitHub:

```bash
git remote set-url --add --push origin "$(git remote get-url origin)"
git remote set-url --add --push origin https://github.com/goodelyfe/arcane.git
git push origin feat/secret-sources
```

## Image

After each passing run, `build-images.yml` builds the branch (as it is on
GitHub, not the rebased test copy) into one all-in-one image, like the
official one, for amd64 and arm64:

- `ghcr.io/goodelyfe/arcane:secret-sources` (latest)
- `ghcr.io/goodelyfe/arcane:secret-sources-<short sha>` (fixed)

Plain upstream needs no build here; use `ghcr.io/getarcaneapp/arcane`.

## Notes

- GitHub pauses scheduled workflows in public repos after 60 days without
  activity in the repo; re-enable from the Actions tab if that happens.
- The fork's `main` only fast-forwards, so never commit to it directly.
- To rotate the GitHub token, update it in Infisical; nothing changes here.
