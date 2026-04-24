### Topic: Code Repository — Level 1

> **Related:** [Level 2 — CI/CD](../level-2/ci_cd.md)

**Why it matters (Karat angle)**
This seems basic, but interviewers use it as a litmus test for real team experience. They'll ask about branching strategies, merge vs rebase, pull request workflows, and conflict resolution — not just "do you use Git." A senior dev who can't explain GitFlow or trunk-based development hasn't led a team.

**Core concept**

Git is a **distributed version control system** — every developer has a full copy of the repository with history. GitHub/Bitbucket are hosting platforms that add pull requests, code review, CI/CD integration, and access control.

**Git vs SVN (the legacy comparison):**

| | Git | SVN (Subversion) |
|--|-----|-------------------|
| Architecture | Distributed — full local repo | Centralised — one server |
| Branching | Cheap (pointer, O(1)) | Expensive (full directory copy) |
| Offline work | Full history available offline | Requires server connection |
| Speed | Fast (local operations) | Slower (network for every operation) |

**Branching strategies:**

| Strategy | How it works | Best for |
|----------|-------------|---------|
| **GitFlow** | `main` (production), `develop` (integration), `feature/*`, `release/*`, `hotfix/*` | Larger teams, release cycles |
| **Trunk-based** | Everyone commits to `main`; short-lived feature branches (< 2 days) | CI/CD, microservices, small teams |
| **GitHub Flow** | `main` + short-lived feature branches + PR | Open source, small teams |

**The daily workflow (GitHub Flow):**

```bash
# 1. Create a feature branch
git checkout -b feature/add-payment-validation

# 2. Make changes, commit frequently
git add src/main/java/PaymentValidator.java
git commit -m "Add input validation for payment amounts"

# 3. Push to remote
git push origin feature/add-payment-validation

# 4. Open a Pull Request on GitHub — code review happens here

# 5. After approval, merge to main
git checkout main
git pull origin main
git merge feature/add-payment-validation
git push origin main

# 6. Delete the feature branch
git branch -d feature/add-payment-validation
git push origin --delete feature/add-payment-validation
```

**Essential Git commands:**

| Command | What it does |
|---------|-------------|
| `git clone <url>` | Copy remote repo locally |
| `git status` | Show modified/staged files |
| `git diff` | Show unstaged changes |
| `git add -A` | Stage all changes |
| `git commit -m "msg"` | Commit staged changes |
| `git push` | Upload commits to remote |
| `git pull` | Fetch + merge from remote |
| `git fetch` | Download remote changes (don't merge) |
| `git branch -a` | List all branches (local + remote) |
| `git checkout -b <name>` | Create and switch to new branch |
| `git merge <branch>` | Merge branch into current |
| `git rebase <branch>` | Replay commits on top of branch |
| `git stash` / `git stash pop` | Temporarily shelve changes |
| `git log --oneline -10` | Show last 10 commits |
| `git reset --hard HEAD~1` | Undo last commit (destructive) |
| `git revert <hash>` | Create a new commit that undoes a previous one (safe) |

**Merge vs Rebase:**

| | Merge | Rebase |
|--|-------|--------|
| History | Preserves branch history (merge commit) | Linear history (no merge commit) |
| Safety | Non-destructive | Rewrites commit hashes — dangerous on shared branches |
| When | Merging feature → main | Keeping a feature branch up-to-date with main |
| Golden rule | Always safe | **Never rebase a shared/public branch** |

```bash
# Rebase: replay your feature commits on top of latest main
git checkout feature/payment-validation
git rebase main
# If conflicts: resolve, then git rebase --continue

# Merge: create a merge commit
git checkout main
git merge feature/payment-validation
```

**Conflict resolution:**

```bash
# When merge/rebase hits a conflict:
git status                                            # shows conflicted files

# Open file — Git marks conflicts:
<<<<<<< HEAD
BigDecimal amount = request.getAmount();               # your change
=======
BigDecimal amount = request.getTransactionAmount();    # their change
>>>>>>> feature/rename-fields

# Resolve manually, then:
git add <resolved-file>
git commit                                            # for merge
# or
git rebase --continue                                 # for rebase
```

**Working code example — .gitignore for a Spring Boot project:**

```gitignore
# File: .gitignore

# Build output
target/
build/
*.jar
*.war

# IDE files
.idea/
*.iml
.vscode/
.settings/
.classpath
.project

# Logs
*.log
logs/

# Environment-specific
application-local.properties
.env

# OS files
.DS_Store
Thumbs.db
```

**Pull request best practices (what interviewers want to hear):**
1. **Small, focused PRs** — one feature or fix per PR. Easier to review.
2. **Descriptive title + description** — what, why, how to test.
3. **At least one reviewer** — no self-merge to main.
4. **CI passes before merge** — automated tests, linting, security scan.
5. **Squash commits** on merge — clean main history.

**GitHub vs Bitbucket:**

| | GitHub | Bitbucket |
|--|--------|-----------|
| Owner | Microsoft | Atlassian |
| CI/CD | GitHub Actions | Bitbucket Pipelines |
| Integration | Excellent open-source ecosystem | Tight Jira/Confluence integration |
| Pricing | Free for public repos | Free for small teams (5 users) |
| Used at Citi | Some teams | Common (Atlassian stack) |

**What to say in the interview (4-beat answer)**
1. **Definition:** Git is a distributed VCS where every clone is a full repository. GitHub/Bitbucket add collaboration features — pull requests, code review, CI/CD hooks, access control.
2. **Why/when:** Branching strategies like GitFlow suit release-based teams; trunk-based development suits CI/CD-heavy microservice teams. The key is short-lived feature branches and mandatory code review via PRs.
3. **Example:** On my team, we use GitHub Flow — feature branches off `main`, PRs with at least one reviewer, CI must pass (unit tests + SonarQube), squash-merge to keep history clean.
4. **Gotcha/tradeoff:** Rebasing gives clean linear history but rewrites commit hashes — never rebase a branch that others have pulled. Use `git revert` (safe) instead of `git reset --hard` (destructive) to undo commits on shared branches.

**Common pitfalls**
- Committing directly to `main` — bypasses code review and CI.
- Long-lived feature branches (weeks) — massive merge conflicts, integration hell.
- Force-pushing (`git push -f`) to a shared branch — overwrites teammates' commits.
- Committing secrets (passwords, API keys, `.env` files) — they persist in Git history even after deletion. Use `.gitignore` and tools like `git-secrets`.
- Not pulling before pushing — leads to rejected pushes and unnecessary merge commits.

**Self-check question**
You're working on a feature branch and `main` has moved ahead with 10 new commits. Should you merge `main` into your branch or rebase your branch onto `main`? What's the trade-off of each?
