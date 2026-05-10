# Vault + GitHub Actions — OIDC/JWT Setup Guide

> **Shell:** All Vault CLI commands below are written for **fish shell**.
> **Platform:** HCP Vault (Dedicated). Namespace is `admin`.
> No secrets, tokens, or cluster URLs are stored in this file — use environment variables.

---

## Overview

GitHub Actions authenticates to Vault using a short-lived OIDC JWT token issued by GitHub.
Vault validates that token against GitHub's OIDC discovery endpoint and grants access based on a role.
No static Vault tokens are stored in GitHub.

Flow:
```
GitHub Action runs
  → GitHub issues short-lived OIDC JWT (audience = Vault URL by default)
  → vault-action sends JWT to Vault's auth/jwt/login
  → Vault validates JWT against GitHub's OIDC discovery endpoint
  → Vault checks bound_claims (repo name matches)
  → Vault returns a short-lived token scoped to github-actions policy
  → Action reads secrets from Vault
```

---

## Step 1 — Set environment variables (fish shell)

```fish
set -x VAULT_ADDR "https://YOUR_VAULT_CLUSTER_URL:8200"
set -x VAULT_NAMESPACE "admin"
set -x VAULT_TOKEN "YOUR_ADMIN_TOKEN"   # only needed for initial setup; never commit this
```

Sanity check:
```fish
env | grep VAULT
vault token lookup
```

---

## Step 2 — Enable JWT auth

```fish
vault auth enable jwt
```

If already enabled you will see: `path is already in use`. That is fine.

Confirm:
```fish
vault auth list
```

---

## Step 3 — Configure Vault to trust GitHub's OIDC issuer

```fish
vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"
```

Verify:
```fish
vault read auth/jwt/config
```

---

## Step 4 — Create a policy for GitHub Actions

This policy allows GitHub Actions to read secrets under `secret/data/myapp/*`.
Adjust the path to match where your secrets actually live.

> fish does not support heredoc (`<<EOF`). Use `printf` and pipe instead:

```fish
printf 'path "secret/data/myapp/*" {\n  capabilities = ["read"]\n}\n' \
  | vault policy write github-actions -
```

Verify:
```fish
vault policy read github-actions
```

---

## Step 5 — Create the JWT role

### Why `@file` is required in fish

The vault CLI expects `bound_claims` to be a JSON map object. Passing it inline as a
string causes Vault to reject it:

```
error converting input for field "bound_claims": '' expected a map, got 'string'
```

This is a shell quoting issue — fish passes the JSON as a literal string, not a parsed map.
The fix is to write the full payload to a temp file and pass it with `@file`:

```fish
set GH_USERNAME YOUR_GH_USERNAME   # replace
set GH_REPO     YOUR_REPO          # replace

set tmp (mktemp)
printf '{
  "role_type": "jwt",
  "bound_audiences": ["https://github.com/%s"],
  "bound_claims_type": "glob",
  "bound_claims": {"sub": "repo:%s/%s:*"},
  "user_claim": "actor",
  "policies": ["github-actions"],
  "ttl": "10m"
}' $GH_USERNAME $GH_USERNAME $GH_REPO > $tmp

vault write auth/jwt/role/github-actions-role @$tmp
rm $tmp
```

Verify:
```fish
vault read auth/jwt/role/github-actions-role
```

Expected key fields:
```
bound_audiences    [https://github.com/YOUR_GH_USERNAME]
bound_claims       map[sub:repo:YOUR_GH_USERNAME/YOUR_REPO:*]
bound_claims_type  glob
policies           [github-actions]
token_ttl          10m
user_claim         actor
```

---

## Step 6 — Store a test secret

```fish
vault kv put secret/myapp/production \
  DB_PASSWORD="replace-with-real-value" \
  STRIPE_KEY="replace-with-real-value"
```

Verify:
```fish
vault kv get secret/myapp/production
```

### KV v2 path note

The CLI path `secret/myapp/production` maps to the API path `secret/data/myapp/production`.
That is why the policy uses `secret/data/myapp/*`. This is normal KV v2 behaviour.

---

## Step 7 — Add GitHub repository secrets

In your GitHub repo: **Settings → Secrets and variables → Actions**

| Secret name       | Value                                   |
|-------------------|-----------------------------------------|
| `VAULT_ADDR`      | your HCP Vault cluster URL (port 8200)  |
| `VAULT_NAMESPACE` | `admin`                                 |

> Do **not** store your admin Vault token in GitHub. JWT auth is the point — no static tokens.

---

## Step 8 — GitHub workflow

```yaml
name: Deploy with Vault

on:
  push:
    branches: [main]

permissions:
  id-token: write   # REQUIRED — without this GitHub will not issue the OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Fetch secrets from Vault
        uses: hashicorp/vault-action@v3
        with:
          url: ${{ secrets.VAULT_ADDR }}
          namespace: ${{ secrets.VAULT_NAMESPACE }}   # REQUIRED for HCP Vault — see lessons learned
          method: jwt
          role: github-actions-role
          secrets: |
            secret/data/myapp/production DB_PASSWORD | DB_PASSWORD ;
            secret/data/myapp/production STRIPE_KEY | STRIPE_KEY

      - name: Verify secrets loaded
        run: |
          echo "DB_PASSWORD present: ${DB_PASSWORD:+yes}"
          echo "STRIPE_KEY present: ${STRIPE_KEY:+yes}"
```

---

## Lessons learned (from actual debugging)

### 1. Missing `namespace` in workflow → misleading "role not found" 400

**Symptom:**
```
failed to retrieve vault token. code: ERR_NON_2XX_3XX_RESPONSE,
message: Response code 400 (Bad Request),
vaultResponse: {"errors":["role \"github-actions-role\" could not be found"]}
```

**Actual cause:** `namespace` was missing from the `hashicorp/vault-action` step.
Without it, the request hits the `root` namespace which has no JWT auth or role configured.
HCP Vault always requires `namespace: admin`.

**Fix:** Add `namespace: ${{ secrets.VAULT_NAMESPACE }}` to the vault-action step.

---

### 2. `bound_claims` passed as string → 400 "expected a map, got string"

**Symptom:**
```
Error writing data to auth/jwt/role/github-actions-role: Code: 400.
error converting input for field "bound_claims": '' expected a map, got 'string'
```

**Cause:** Inline quoting of JSON in fish shell passes `bound_claims` as a string literal,
not a JSON object. The vault CLI does not parse inline JSON strings for map fields.

**Fix:** Use `@file` with a temp file (see Step 5 above).

---

### 3. Audience mismatch → JWT login fails after role is found

`hashicorp/vault-action@v3` requests the GitHub OIDC token with the **Vault URL** as the
audience by default, not `https://github.com/YOUR_GH_USERNAME`.

The role's `bound_audiences` must match what the action sends.

**Option A (recommended):** Set `bound_audiences` to your Vault URL when creating the role:
```json
"bound_audiences": ["https://YOUR_VAULT_CLUSTER_URL:8200"]
```

**Option B:** Keep `bound_audiences` as `https://github.com/YOUR_GH_USERNAME` and add
`jwtGithubAudience` in the workflow to override what GitHub issues:
```yaml
jwtGithubAudience: "https://github.com/YOUR_GH_USERNAME"
```

---

## Helpful verification commands

```fish
vault auth list                                  # confirm jwt/ is enabled
vault read auth/jwt/config                       # confirm GitHub issuer is configured
vault read auth/jwt/role/github-actions-role     # confirm role exists and fields are correct
vault policy read github-actions                 # confirm policy exists
vault kv get secret/myapp/production             # confirm secret exists
```

---

## Final checklist

- [ ] `vault auth list` shows `jwt/`
- [ ] JWT config points to `https://token.actions.githubusercontent.com`
- [ ] Role exists with correct `bound_claims` (repo) and `bound_audiences` (Vault URL)
- [ ] Policy `github-actions` grants read on the correct `secret/data/...` path
- [ ] Secret exists at the expected path
- [ ] GitHub workflow has `id-token: write` permission
- [ ] GitHub repo secret `VAULT_ADDR` is set
- [ ] GitHub repo secret `VAULT_NAMESPACE` is set to `admin`
- [ ] `namespace:` field is present in the `hashicorp/vault-action` step
