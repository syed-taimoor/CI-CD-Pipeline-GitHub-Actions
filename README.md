# ⚙️ CI/CD Pipeline — 3 Environment Setup

> `develop` → Development | `release` → Staging | `main` → Production

---

## Pipeline Flow

```
Push to develop              Push to release              Push to main
      │                            │                           │
      ▼                            ▼                           ▼
┌───────────┐              ┌───────────┐              ┌───────────┐
│🔍 Quality │              │🔍 Quality │              │🔍 Quality │
└─────┬─────┘              └─────┬─────┘              └─────┬─────┘
      │                          │                           │
      ▼                          ▼                           ▼
┌───────────┐              ┌───────────┐              ┌───────────┐
│🏗️  Build  │              │🏗️  Build  │              │🏗️  Build  │
└─────┬─────┘              └─────┬─────┘              └─────┬─────┘
      │                          │                           │
      ▼                          ▼                           ▼
┌───────────┐              ┌───────────┐              ┌───────────┐
│🟡  Dev    │              │🔵 Staging │              │🟢  Prod   │
│  Server   │              │  Server   │              │  Server   │
└───────────┘              └───────────┘              └───────────┘

Pull Requests  →  Quality + Build only (deploy is blocked)
Any other branch  →  Nothing runs
```

---

## Step 1 — Add the Workflow File

In your project root:

```bash
mkdir -p .github/workflows
# Copy deploy.yml into .github/workflows/deploy.yml
```

---

## Step 2 — Create GitHub Environments

Go to your repo → **Settings → Environments → New environment**

Create these 3:

| Environment Name | Branch |
|-----------------|--------|
| `development` | `develop` |
| `staging` | `release` |
| `production` | `main` |

> For `production`, enable **Required reviewers** so someone must approve before it goes live.

---

## Step 3 — Generate SSH Keys (one per server)

On your local machine:

```bash
# Dev key
ssh-keygen -t ed25519 -C "github-actions-dev" -f ~/.ssh/deploy_dev

# Staging key
ssh-keygen -t ed25519 -C "github-actions-staging" -f ~/.ssh/deploy_staging

# Production key
ssh-keygen -t ed25519 -C "github-actions-prod" -f ~/.ssh/deploy_prod
```

Add each public key to its server:

```bash
# On Dev server
ssh-copy-id -i ~/.ssh/deploy_dev.pub root@YOUR_DEV_IP

# On Staging server
ssh-copy-id -i ~/.ssh/deploy_staging.pub root@YOUR_STAGING_IP

# On Production server
ssh-copy-id -i ~/.ssh/deploy_prod.pub root@89.167.105.136
```

---

## Step 4 — Add Secrets to GitHub

**Settings → Secrets and variables → Actions → New repository secret**

### Development Secrets

| Secret | Value |
|--------|-------|
| `DEV_HOST` | Your dev server IP |
| `DEV_USER` | `root` |
| `DEV_SSH_KEY` | Contents of `~/.ssh/deploy_dev` |
| `DEV_PORT` | `22` |

### Staging Secrets

| Secret | Value |
|--------|-------|
| `STAGING_HOST` | Your staging server IP |
| `STAGING_USER` | `root` |
| `STAGING_SSH_KEY` | Contents of `~/.ssh/deploy_staging` |
| `STAGING_PORT` | `22` |

### Production Secrets

| Secret | Value |
|--------|-------|
| `PROD_HOST` | `89.167.105.136` |
| `PROD_USER` | `root` |
| `PROD_SSH_KEY` | Contents of `~/.ssh/deploy_prod` |
| `PROD_PORT` | `22` |

> Read a private key with: `cat ~/.ssh/deploy_prod`
> Copy everything including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`

---

## Step 5 — Set Up Git Branches

```bash
# Create develop branch
git checkout -b develop
git push origin develop

# Create release branch
git checkout -b release
git push origin release

# main already exists
```

Set branch protection rules — **Settings → Branches → Add rule**:

| Branch | Recommended Rules |
|--------|------------------|
| `main` | Require PR, require CI checks to pass |
| `release` | Require PR, require CI checks to pass |
| `develop` | Require CI checks to pass |

---

## Step 6 — Commit and Push

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add 3-environment CI/CD pipeline"
git push origin develop
```

Go to **Actions** tab on GitHub — you'll see the pipeline trigger for `develop` → Dev server.

---

## Daily Git Flow

```
feature/your-feature
        │
        │ PR → merge
        ▼
    develop  ──────────►  🟡 Dev Server     (auto deploy)
        │
        │ PR → merge
        ▼
    release  ──────────►  🔵 Staging Server (auto deploy)
        │
        │ PR → merge (after QA sign-off)
        ▼
      main   ──────────►  🟢 Production     (auto deploy)
```

---

## PM2 Process Per Environment

| Environment | PM2 Name | Server |
|-------------|----------|--------|
| Development | `portfolio-dev` | Dev IP |
| Staging | `portfolio-staging` | Staging IP |
| Production | `portfolio` | `89.167.105.136` |

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Deploy job skipped | Branch name must exactly match `develop`, `release`, or `main` |
| `Permission denied` on SSH | Re-paste the full private key including header and footer lines |
| Wrong environment triggered | Check the `if:` condition in `deploy.yml` |
| Build fails in CI | Confirm Node version — workflow uses Node 20 |
| Production needs manual approval | Enable **Required reviewers** in GitHub Environment settings |
