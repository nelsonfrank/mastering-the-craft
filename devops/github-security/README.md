**GitHub's third-party access features** — like **Authorized OAuth Apps**, **Installed GitHub Apps**, and related integrations — let external services connect to your account or repositories for useful automation (e.g., CI/CD tools, project managers, or code scanners). However, they introduce real security risks if not managed carefully. Let's break this down clearly: what each is, how they differ, how to manage them, the risks involved, and best practices to stay secure.

### 1. What Are Authorized OAuth Apps?
These are classic third-party applications that use **OAuth 2.0** to act *on your behalf*. When you log in to a third-party tool (like an old CI service or a code analysis app) with "Sign in with GitHub," you're authorizing an OAuth App.

- It gets an **access token** tied to your identity.
- The token has **broad scopes** (e.g., the `repo` scope gives full read/write access to *all* your repositories — public and private).
- The token **never expires** until you manually revoke it.
- You’ll see these listed in your account under **Settings → Applications → Authorized OAuth Apps**.

OAuth Apps were the original integration method and are still common, but they’re less secure by design.

### 2. What Are Installed GitHub Apps (and Authorized GitHub Apps)?
**GitHub Apps** are GitHub’s modern, recommended integration type. They are “installed” rather than just authorized.

- **Installation**: The app is installed on specific repositories, your personal account, or an organization. You choose exactly which repos it can touch.
- **Permissions**: These are **fine-grained** (e.g., “Issues: Read” only, or “Pull Requests: Write” without touching code). No broad “repo” scope required.
- **Tokens**: Short-lived (expire after 1 hour) with automatic refresh — much safer than permanent tokens. The app can also act independently (as a bot like `@myapp-bot`) without needing a user logged in.
- There are two tabs in settings:
  - **Installed GitHub Apps** → Controls where the app is installed and which repos it can access.
  - **Authorized GitHub Apps** → For cases where the app needs to act *as you* (user-to-app authorization).

GitHub strongly prefers GitHub Apps over OAuth Apps for new integrations because they give you (and org owners) far more control.

### 3. Key Differences (OAuth Apps vs. GitHub Apps)

| Aspect                  | OAuth Apps                              | GitHub Apps (preferred)                          |
|-------------------------|-----------------------------------------|--------------------------------------------------|
| **Permissions**        | Broad scopes (e.g., `repo` = everything) | Fine-grained (e.g., only “Issues” or “Contents: Read”) |
| **Repository Access**  | All repos you can see                   | Only the repos you explicitly choose during install |
| **Tokens**             | Permanent until revoked                 | Short-lived (1 hour) + refresh tokens            |
| **Acts as**            | You (shows up as your username)         | Bot account (independent of any user)            |
| **Best for**           | Simple personal tools                   | Organizations, long-running automations, security |
| **Security**           | Higher risk if compromised              | Lower risk (least privilege + expiring tokens)   |
| **Org Controls**       | Subject to OAuth approval policies      | Not subject to those policies; more flexible     |

GitHub Apps also have built-in webhooks, better rate limits, and survive if the person who installed them leaves the company.

### 4. Security Risks to Your Account
These integrations are convenient but can be a major attack vector:

- **Compromised or malicious app**: If the third-party service gets hacked (or is malicious from the start), the attacker gets whatever access you granted — potentially full repo access, ability to push malicious code, exfiltrate private data, or even create backdoors.
- **Permanent tokens (OAuth)**: A leaked token stays valid forever unless you revoke it. No expiration means long windows for abuse.
- **Overly broad permissions**: Many legacy OAuth Apps request way more access than they need.
- **Supply-chain attacks**: Attackers have targeted popular OAuth/GitHub Apps in the past to steal tokens or inject malware into thousands of repos at once.
- **Organization risk**: A single member can authorize a risky app that affects the whole org’s private repos.
- **Persistence**: Even if you leave a company, an installed GitHub App can keep running (good for continuity, bad if you forget to uninstall).

In short: these are like giving a third party a key to your house. If the key is copied or the service is breached, your entire GitHub presence is at risk.

### 5. How to Manage and Revoke Access (Do This Regularly!)
GitHub makes it easy — you should review these **at least every few months**.

**For Authorized OAuth Apps**:
1. Go to your profile picture → **Settings** → **Applications**.
2. Click **Authorized OAuth Apps** tab.
3. Review the list. Look for anything unfamiliar, old, or with broad permissions (especially private repo access).
4. Click the revoke icon → **Revoke** (or **Revoke all**).

**For GitHub Apps (Authorized + Installed)**:
1. Same path: **Settings** → **Applications**.
2. **Authorized GitHub Apps** tab → Review & revoke user-level authorization.
3. **Installed GitHub Apps** tab → Click **Configure** on any app:
   - Change repository access (All → Only select repositories).
   - Suspend (temporary block) or **Uninstall** (permanent removal).
4. For organizations: Go to the org → **Settings** → **Third-party Access** → **GitHub Apps**.

**Pro tip**: Organizations can enable **OAuth App Access Restrictions** (under org Settings → Third-party Access) so members must get owner approval before authorizing apps that touch org resources. Highly recommended.

### 6. Best Practices to Minimize Risk
- **Prefer GitHub Apps** over OAuth Apps whenever possible (most modern tools support them).
- **Least privilege**: Always grant the minimum permissions and only the specific repos needed.
- **Review regularly**: Make it a habit — check Authorized/Installed apps every quarter.
- **Trust but verify**: Only authorize apps from reputable developers. Check their GitHub repo, privacy policy, and recent security incidents.
- **For organizations**: Enable OAuth restrictions, require approval, and use audit logs to monitor app activity.
- **Monitor your security log**: Go to Settings → Security log and filter for `oauth_authorization` or app-related events.
- **Remove access immediately** when you stop using a tool.

GitHub has improved these systems a lot (especially pushing GitHub Apps with fine-grained permissions and short-lived tokens), but the biggest risk is still **you** — forgetting to revoke old access. Treat every authorized app like a standing invitation into your account.



**GitHub Developer Settings** is the central hub where you manage programmatic access methods for your account. These tools are powerful for automation, scripts, CI/CD, and building integrations, but they are also among the **highest security risks** on GitHub because they often grant long-lived, high-privilege access directly to your account or repositories.

You can find it by clicking your profile picture → **Settings** → **Developer settings** (direct link: `https://github.com/settings/developers`).

### Main Sections in Developer Settings

1. **Personal Access Tokens (PATs)**  
   - **Classic PATs**: Old-style tokens with broad scopes (e.g., full `repo` scope = read/write to **all** your repos, public + private).  
   - **Fine-grained PATs** (recommended): Newer, more secure option with granular permissions and repository-specific access.

2. **OAuth Apps**  
   - Where you create and manage OAuth Apps that you build yourself.

3. **GitHub Apps**  
   - Where you create and manage modern GitHub Apps (preferred for most use cases).

4. **SSH and GPG keys**  
   - SSH keys for Git operations, GPG for commit signing.

5. **Other** (sometimes listed): Deploy keys, Codespaces secrets, etc.

### Security Risks of Developer Settings Features

These are like giving out master keys to your GitHub account. If compromised, attackers can steal code, inject malware, exfiltrate secrets, or take over your entire presence.

| Feature                  | Key Security Risks                                                                 | Risk Level | Why It's Dangerous |
|--------------------------|------------------------------------------------------------------------------------|------------|--------------------|
| **Classic PATs**        | Long-lived (no forced expiration), broad scopes, acts as **you** (full account access) | Very High | One leaked token = full repo access + ability to create more tokens, install apps, etc. GitHub auto-removes after 1 year of inactivity, but that's not enough. |
| **Fine-grained PATs**   | Still long-lived by default, can be over-permissioned, limited support for some old APIs | High (but lower than classic) | Better scoping (specific repos + permissions), but still powerful if leaked. Some limitations exist. |
| **OAuth Apps (you create)** | Client secrets, broad access possible, permanent tokens | High      | If your app's secret leaks or the app is maliciously used, it can affect all users who authorized it. |
| **GitHub Apps (you create)** | Private keys for installation tokens, client secrets for user tokens | Medium-High | Short-lived tokens are safer, but private key compromise is catastrophic. |
| **SSH Keys**            | Full Git push/pull access without password; no granular control | High      | Compromised machine = attacker can push malicious code to any repo you have write access to. |
| **Deploy Keys**         | Repo-specific SSH keys (read or read+write) | Medium    | Limited to one repo, but write-enabled ones still allow code injection. |

**Common attack scenarios**:
- Token leakage in CI logs, `.env` files, public repos, or compromised build servers.
- Supply-chain attacks (e.g., malicious GitHub Actions or third-party tools stealing your PAT).
- Phishing or malware on your dev machine stealing SSH keys or tokens from keychain.
- Overly permissive tokens used in scripts/shared with teams.

Recent supply-chain incidents (2025-2026) frequently involve stolen PATs or GitHub tokens from compromised Actions/workflows, leading to secret exfiltration and further downstream attacks.

### Best Practices to Minimize Risks

**1. Prefer Fine-grained PATs over Classic**
- Always choose fine-grained when possible.
- Limit to **specific repositories** (not "All repositories").
- Grant **only the exact permissions** needed (e.g., `Contents: Read` only, never full `repo`).
- Set expiration dates (e.g., 30-90 days) and rotate regularly.
- GitHub now pushes fine-grained heavily for new tokens.

**2. Use GitHub Apps Instead of PATs Whenever Possible**
- GitHub Apps use fine-grained permissions, short-lived tokens (1 hour), and can act as a bot (not as you).
- Ideal for CI/CD, bots, and long-running automations.
- Much safer for organizations.

**3. SSH & Keys Management**
- Use SSH keys with passphrases.
- Regularly review and remove unused keys.
- Prefer deploy keys (per-repo) over personal SSH keys for automation.
- Sign commits with GPG for verification.

**4. General Hygiene (Do This Regularly)**
- Go to **Developer settings → Personal access tokens** and review/revoke unused tokens.
- Check **Authorized OAuth Apps** and **GitHub Apps** (as discussed earlier).
- Enable **OAuth App restrictions** at organization level.
- Never hardcode tokens — use GitHub Secrets, environment variables, or secret managers.
- Monitor your **Security log** (Settings → Security log) for token creation/usage.
- Use the principle of **least privilege** everywhere.

**5. For Organizations**
- Set PAT policies to restrict or require approval for classic tokens.
- Use GitHub Apps for shared automations.
- Enable secret scanning and Dependabot.

### Quick How-To: Review & Secure
1. Go to **Settings → Developer settings**.
2. Check **Personal access tokens** (both tabs).
3. For any token: Note the last used date, scopes, and repos — revoke if suspicious or unused.
4. When creating new: Always fine-grained + expiration + minimal access.

**Bottom line**: Developer settings features are incredibly useful but represent a **standing high-privilege access** that attackers love to target. The shift from classic PATs/OAuth to fine-grained + GitHub Apps has improved things significantly, but poor management still causes major breaches.
