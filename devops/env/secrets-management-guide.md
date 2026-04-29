# Secrets Management Guide: From Development to Production

A comprehensive guide on securing environment variables and secrets across all stages of your software lifecycle.

---

## Table of Contents

1. [What Are Secrets?](#what-are-secrets)
2. [Securing `.env` in Development](#securing-env-in-development)
3. [Securing Secrets in CI/CD](#securing-secrets-in-cicd)
4. [Open-Source Secrets Managers for Production](#open-source-secrets-managers-for-production)
5. [Choosing the Right Tool](#choosing-the-right-tool)
6. [Quick Reference Checklists](#quick-reference-checklists)

---

## What Are Secrets?

Secrets are sensitive configuration values that must never be exposed publicly. They include:

- **Database credentials** — connection strings, usernames, passwords
- **API keys** — third-party service tokens (Stripe, Twilio, SendGrid, etc.)
- **JWT secrets** — signing keys for authentication tokens
- **SSH keys** — private keys for server access
- **Encryption keys** — keys used to encrypt/decrypt data at rest
- **OAuth credentials** — client IDs and secrets for OAuth flows

Exposing secrets can lead to data breaches, financial loss, unauthorized access to infrastructure, and regulatory penalties.

---

## Securing `.env` in Development

The `.env` file is the standard way to manage secrets in local development. However, it must be handled carefully to avoid accidental exposure.

### 1. Add `.env` to `.gitignore` Immediately

This is the single most important step. A leaked `.env` file committed to a public repository is one of the most common causes of security breaches.

```gitignore
# .gitignore
.env
.env.local
.env*.local
.env.development
.env.production
```

**Never commit a `.env` file to version control — even in a private repository.**

---

### 2. Use a `.env.example` Template

Commit a `.env.example` file with placeholder values. This communicates what variables are required without exposing real secrets.

```bash
# .env.example (safe to commit)
DATABASE_URL=your_database_url_here
API_KEY=your_api_key_here
JWT_SECRET=your_jwt_secret_here
PORT=3000
```

Team members clone the repo and copy this file:

```bash
cp .env.example .env
# Then fill in real values
```

---

### 3. Restrict File Permissions (Unix/macOS)

Limit who can read the `.env` file at the OS level:

```bash
# Owner can read/write; no access for group or others
chmod 600 .env

# Verify permissions
ls -la .env
# -rw------- 1 user group 256 Jan 1 .env
```

---

### 4. Never Log Environment Variables

Avoid printing secrets to the console or log files:

```js
// ❌ BAD — exposes all env vars including secrets
console.log(process.env);

// ❌ BAD — logs the actual secret value
console.log("API Key:", process.env.API_KEY);

// ✅ GOOD — log only existence, not value
console.log("API Key loaded:", !!process.env.API_KEY);
console.log("DB connected:", !!process.env.DATABASE_URL);
```

---

### 5. Validate Env Vars at Startup

Fail fast if required variables are missing or malformed. Use a validation library like `envalid` or `zod`:

**Using `envalid`:**

```js
import { cleanEnv, str, port, url } from 'envalid';

const env = cleanEnv(process.env, {
  DATABASE_URL: url(),
  API_KEY:      str(),
  PORT:         port({ default: 3000 }),
  NODE_ENV:     str({ choices: ['development', 'test', 'production'] }),
});

// env.DATABASE_URL is now type-safe and guaranteed to exist
```

**Using `zod`:**

```js
import { z } from 'zod';

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  API_KEY:      z.string().min(1),
  PORT:         z.coerce.number().default(3000),
});

export const env = envSchema.parse(process.env);
```

If a required variable is missing, the app crashes immediately at startup with a clear error — rather than failing silently in production.

---

### 6. Avoid `.env` in Docker Images

Never copy your `.env` file into a Docker image. Docker layers can be inspected, and images are often pushed to registries.

```dockerfile
# ❌ BAD — secrets baked into the image
COPY .env /app/.env
RUN source /app/.env

# ✅ GOOD — inject at runtime only
# (no .env-related commands in Dockerfile)
```

Inject secrets at runtime via `--env-file`:

```bash
docker run --env-file .env myapp
# or
docker run -e DATABASE_URL="postgres://..." myapp
```

Also add `.env` to `.dockerignore`:

```
# .dockerignore
.env
.env.*
```

---

### 7. Rotate Secrets Immediately If Exposed

If a `.env` file is ever:
- Accidentally committed to Git
- Shared in a chat, email, or screenshot
- Included in logs or error reports

**Treat every secret in that file as compromised.** Rotate all of them immediately, regardless of whether misuse has been detected.

---

### Development Security Checklist

| Step | Done? |
|------|-------|
| `.env` added to `.gitignore` | ✅ |
| `.env.example` committed with placeholders | ✅ |
| File permissions set to `600` | ✅ |
| No secrets logged to console | ✅ |
| Env vars validated at startup | ✅ |
| `.env` excluded from Docker image | ✅ |
| Secrets rotated if ever exposed | ✅ |

---

## Securing Secrets in CI/CD

In CI/CD pipelines, **never use a `.env` file directly**. Instead, use the secret management system built into your CI/CD platform.

### Core Rule

```
Development  →  .env file (local only, gitignored)
CI/CD        →  Platform secret store
Production   →  Dedicated secrets manager
```

---

### GitHub Actions

Store secrets in: **Settings → Secrets and variables → Actions**

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run tests
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          API_KEY: ${{ secrets.API_KEY }}
        run: npm test

      - name: Deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: npm run deploy
```

- Secrets are masked in logs automatically
- Secrets are scoped to environments (staging, production)
- Use **environment protection rules** to require approvals before deploying to production

---

### GitLab CI

Store secrets in: **Settings → CI/CD → Variables**

```yaml
# .gitlab-ci.yml
deploy:
  stage: deploy
  script:
    - echo "Deploying with DB=$DATABASE_URL"
    # Variables are auto-injected from CI settings
  only:
    - main
```

- Mark variables as **Masked** to hide from logs
- Mark variables as **Protected** to restrict to protected branches only
- Scope variables to specific environments

---

### CircleCI

Store secrets in: **Project Settings → Environment Variables**

```yaml
# .circleci/config.yml
version: 2.1
jobs:
  deploy:
    docker:
      - image: cimg/node:18.0
    steps:
      - checkout
      - run:
          name: Deploy application
          command: npm run deploy
          # DATABASE_URL and API_KEY are auto-injected from project settings
```

---

### Generating a `.env` File at Runtime (When Required)

Some tools require a `.env` file on disk. Generate it from secrets at runtime and delete it after use:

```yaml
# GitHub Actions example
steps:
  - name: Create .env from secrets
    run: |
      echo "DATABASE_URL=${{ secrets.DATABASE_URL }}" >> .env
      echo "API_KEY=${{ secrets.API_KEY }}" >> .env
      echo "JWT_SECRET=${{ secrets.JWT_SECRET }}" >> .env

  - name: Run application
    run: npm start

  - name: Cleanup secrets
    if: always()   # runs even if previous step fails
    run: rm -f .env
```

---

### What to Never Do in CI/CD

```yaml
# ❌ NEVER hardcode secrets in config files
env:
  API_KEY: "sk-abc123-real-secret"

# ❌ NEVER echo secret values
- run: echo "My API key is $API_KEY"

# ❌ NEVER copy .env into Docker build context
- run: docker build --build-arg SECRET=${{ secrets.SECRET }} .

# ❌ NEVER store secrets in source code or config
# ❌ NEVER commit .env to the repo, even for CI
```

---

### Scope Secrets Per Environment

Most platforms allow you to scope secrets to specific environments or branches. Use this to enforce separation:

| Environment | Database | API Key |
|-------------|----------|---------|
| Development | dev-db | dev-key (read-only) |
| Staging | staging-db | staging-key |
| Production | prod-db | prod-key (restricted access) |

This prevents a staging pipeline from accidentally connecting to a production database.

---

### CI/CD Security Checklist

| Step | Done? |
|------|-------|
| Secrets stored in platform secret store | ✅ |
| No secrets hardcoded in YAML config | ✅ |
| Secrets masked in CI logs | ✅ |
| Secrets scoped per environment/branch | ✅ |
| `.env` files deleted after ephemeral use | ✅ |
| Secrets rotated on team member offboarding | ✅ |
| Access logs reviewed periodically | ✅ |

---

## Open-Source Secrets Managers for Production

For production systems, you need a dedicated secrets manager — a centralized, auditable, access-controlled vault for all your secrets.

---

### 1. HashiCorp Vault — Industry Standard

The most widely used self-hosted secrets manager. Battle-tested at scale.

**Key Features:**
- Dynamic secrets (generates temporary DB credentials on demand)
- Fine-grained ACL policies
- Secret leasing and automatic rotation
- Audit logging for every secret access
- Multiple auth backends (LDAP, GitHub, Kubernetes, AWS IAM)
- Encryption as a service

**Installation (Docker):**

```bash
docker run -d \
  --name vault \
  -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=mytoken \
  hashicorp/vault
```

**Basic Usage:**

```bash
# Authenticate
export VAULT_ADDR="http://127.0.0.1:8200"
export VAULT_TOKEN="mytoken"

# Store a secret
vault kv put secret/myapp \
  DATABASE_URL="postgres://user:pass@db:5432/mydb" \
  API_KEY="sk-abc123"

# Read a secret
vault kv get secret/myapp
vault kv get -field=DATABASE_URL secret/myapp

# Dynamic database credentials (short-lived, auto-rotated)
vault read database/creds/my-role
```

**In Node.js:**

```js
import Vault from 'node-vault';

const vault = Vault({ endpoint: 'http://vault:8200', token: process.env.VAULT_TOKEN });
const { data } = await vault.read('secret/data/myapp');
const { DATABASE_URL, API_KEY } = data.data;
```

**Best for:** Large teams, enterprise workloads, Kubernetes environments

**Cons:** Complex to operate, requires dedicated infrastructure team

---

### 2. Infisical — Modern Developer-Friendly Alternative

Purpose-built Vault alternative with excellent developer experience. Self-hostable with Docker Compose in minutes.

**Key Features:**
- Beautiful web UI for managing secrets
- CLI that injects secrets directly as env vars
- End-to-end encryption
- Secret versioning and audit logs
- Native GitHub Actions, GitLab CI, Kubernetes integrations
- SDKs for Node.js, Python, Go, Java, and more

**Self-Host with Docker Compose:**

```bash
git clone https://github.com/Infisical/infisical
cd infisical
cp .env.example .env
docker-compose up -d
```

**CLI Usage:**

```bash
# Install CLI
npm install -g @infisical/cli

# Login and link project
infisical login
infisical init

# Run your app with secrets auto-injected
infisical run -- node server.js
infisical run -- python app.py
infisical run -- npm run dev
```

**GitHub Actions Integration:**

```yaml
- name: Inject Infisical secrets
  uses: Infisical/secrets-action@v1
  with:
    client-id: ${{ secrets.INFISICAL_CLIENT_ID }}
    client-secret: ${{ secrets.INFISICAL_CLIENT_SECRET }}
    env-slug: "production"
    project-slug: "my-project"
```

**Node.js SDK:**

```js
import { InfisicalClient } from '@infisical/sdk';

const client = new InfisicalClient({
  clientId: process.env.INFISICAL_CLIENT_ID,
  clientSecret: process.env.INFISICAL_CLIENT_SECRET,
});

const secret = await client.getSecret({
  secretName: "DATABASE_URL",
  projectId: "my-project-id",
  environment: "production",
});
```

**Best for:** Startups and mid-size teams wanting simplicity with strong security

---

### 3. OpenBao — Vault Fork (Truly Open-Source)

Forked from HashiCorp Vault after the BSL license change. Maintains API compatibility with Vault while remaining fully open-source under MPL 2.0, governed by the Linux Foundation.

**Key Features:**
- 100% API-compatible with HashiCorp Vault
- Full Vault feature set (dynamic secrets, audit logs, ACLs)
- Open governance under the Linux Foundation
- Active community development

**Installation:**

```bash
# Download binary
wget https://github.com/openbao/openbao/releases/latest/download/bao_linux_amd64.zip
unzip bao_linux_amd64.zip

# Start dev server
./bao server -dev

# Usage is identical to Vault
export BAO_ADDR="http://127.0.0.1:8200"
bao kv put secret/myapp API_KEY="abc123"
bao kv get -field=API_KEY secret/myapp
```

**Migration from Vault:** Drop-in replacement — change `vault` to `bao` in commands, update the client library, done.

**Best for:** Teams currently using or evaluating Vault who want to avoid BSL license constraints

---

### 4. Bitwarden Secrets Manager

Built on the trusted Bitwarden codebase. Integrates with Bitwarden's password manager for unified credential management.

**Key Features:**
- End-to-end encryption with zero-knowledge architecture
- Machine accounts for CI/CD and service authentication
- SDK support for multiple languages
- Familiar UI for teams already using Bitwarden

**Self-Host:**

```bash
# Uses the standard Bitwarden self-host setup
curl -Lso bitwarden.sh \
  "https://func.bitwarden.com/api/dl/?app=self-host&platform=linux"
chmod 700 bitwarden.sh
./bitwarden.sh install
./bitwarden.sh start
```

**CLI Usage:**

```bash
# Install CLI
npm install -g @bitwarden/cli

# Login
bws login

# Get a secret by ID
bws secret get <secret-uuid>

# List secrets in a project
bws secret list <project-uuid>
```

**Node.js SDK:**

```js
import { BitwardenClient, ClientSettings, DeviceType, LogLevel } from "@bitwarden/sdk-napi";

const settings = {
  apiUrl: "https://api.bitwarden.com",
  identityUrl: "https://identity.bitwarden.com",
  userAgent: "Bitwarden SDK",
  deviceType: DeviceType.SDK,
};

const client = new BitwardenClient(settings, LogLevel.Info);
await client.loginWithAccessToken(process.env.BWS_ACCESS_TOKEN);
const secret = await client.secrets().get("<secret-uuid>");
```

**Best for:** Teams already using Bitwarden for password management

---

### 5. SOPS (Secrets OPerationS) — Git-Native Encryption

Mozilla's tool for encrypting secret files and storing them safely in Git. No server required.

**Key Features:**
- Encrypts files in-place (YAML, JSON, ENV, INI formats)
- Works with AWS KMS, GCP KMS, Azure Key Vault, PGP, and `age`
- Encrypted files can live in Git
- Partial encryption — can encrypt only values, leaving keys readable
- Great for GitOps workflows

**Installation:**

```bash
# macOS
brew install sops

# Linux
wget https://github.com/mozilla/sops/releases/latest/download/sops-linux-amd64
chmod +x sops-linux-amd64 && sudo mv sops-linux-amd64 /usr/local/bin/sops
```

**Using with `age` (recommended for simplicity):**

```bash
# Install age
brew install age

# Generate a key pair
age-keygen -o key.txt
# Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p

# Encrypt a .env file
sops --encrypt \
  --age age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p \
  .env > .env.enc

# Decrypt at runtime
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
sops --decrypt .env.enc > .env

# Or inject directly without writing to disk
sops exec-env .env.enc 'node server.js'
sops exec-env .env.enc 'npm run start'
```

**Partial encryption (encrypt values only, keep keys readable):**

```yaml
# secrets.yaml (after encryption)
database:
    url: ENC[AES256_GCM,data:abc123...,type:str]   # encrypted
    name: myapp                                       # not encrypted
    port: 5432                                        # not encrypted
```

**Using with AWS KMS:**

```bash
sops --encrypt \
  --kms arn:aws:kms:us-east-1:123456789:key/mrk-abc123 \
  secrets.yaml > secrets.enc.yaml
```

**Best for:** Small teams, GitOps workflows, avoiding server infrastructure

---

## Choosing the Right Tool

### Comparison Table

| Tool | License | Self-Host | Complexity | Server Required | Best For |
|------|---------|-----------|------------|-----------------|----------|
| **Vault** | BSL 1.1 | ✅ | High | Yes | Enterprise, large teams |
| **Infisical** | MIT | ✅ | Low | Yes | Startups, modern teams |
| **OpenBao** | MPL 2.0 | ✅ | High | Yes | Vault users, BSL concerns |
| **Bitwarden SM** | GPL 3.0 | ✅ | Low-Medium | Yes | Bitwarden users |
| **SOPS** | MPL 2.0 | ✅ (no server) | Medium | No | GitOps, small teams |

---

### Decision Guide by Team Size

```
Solo developer / small team
    └── SOPS + age encryption
        (no infrastructure, secrets live in Git)

Growing startup (5–50 engineers)
    └── Infisical (self-hosted)
        (easy setup, great DX, handles multiple environments)

Mid-to-large team with Bitwarden
    └── Bitwarden Secrets Manager
        (unified password + secret management)

Large / enterprise team
    └── HashiCorp Vault or OpenBao
        (maximum control, dynamic secrets, Kubernetes integration)

GitOps / Kubernetes team
    └── SOPS + External Secrets Operator
        (syncs encrypted secrets from Git into Kubernetes secrets)
```

---

### The Full Security Lifecycle

```
┌─────────────────────────────────────────────────────┐
│                 SECRET LIFECYCLE                     │
│                                                      │
│  Developer  →  .env (local, gitignored)              │
│                                                      │
│  CI/CD      →  GitHub/GitLab/CircleCI secret store   │
│                                                      │
│  Staging    →  Secrets manager (staging environment) │
│                                                      │
│  Production →  Secrets manager (prod environment)    │
│             →  App reads secrets at runtime          │
│             →  Secrets never touch disk              │
│             →  Access is logged and auditable        │
└─────────────────────────────────────────────────────┘
```

---

## Quick Reference Checklists

### Development
- [ ] `.env` is in `.gitignore`
- [ ] `.env.example` committed with placeholders
- [ ] File permissions set to `600`
- [ ] No secrets logged to console
- [ ] Env vars validated at app startup
- [ ] `.env` excluded from Docker builds via `.dockerignore`

### CI/CD
- [ ] Secrets stored in platform secret store (not in YAML)
- [ ] No secrets hardcoded anywhere in pipeline config
- [ ] Secrets masked in CI logs
- [ ] Secrets scoped per environment and branch
- [ ] Ephemeral `.env` files deleted after use
- [ ] Secrets rotated on team member offboarding

### Production
- [ ] Dedicated secrets manager deployed and operational
- [ ] Secrets injected at runtime (not baked into images)
- [ ] Audit logging enabled for all secret access
- [ ] Automatic secret rotation configured where possible
- [ ] Access limited by principle of least privilege
- [ ] Secret access reviewed and audited regularly
- [ ] Incident response plan for secret exposure documented

---

*Last updated: April 2026*
