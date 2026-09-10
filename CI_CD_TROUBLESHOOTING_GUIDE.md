# CI/CD Pipeline Troubleshooting Guide

## 📋 Table of Contents
1. [What Happened - The Full Story](#what-happened---the-full-story)
2. [Problem #1: Git Index Out of Sync](#problem-1-git-index-out-of-sync)
3. [Problem #2: Bundle Structure Wrong](#problem-2-bundle-structure-wrong)
4. [Problem #3: Merge Conflict](#problem-3-merge-conflict)
5. [Problem #4: Terraform Key Expiration](#problem-4-terraform-key-expiration)
6. [Key Concepts Explained](#key-concepts-explained)
7. [How to Handle Similar Issues in Future](#how-to-handle-similar-issues-in-future)

---

## What Happened - The Full Story

### Your Goal
You wanted to set up **automatic validation** of your Databricks Asset Bundle (DAB) whenever you create a Pull Request on GitHub. This is called **CI/CD** (Continuous Integration/Continuous Deployment).

### The Journey
1. ✅ You created a GitHub Actions workflow file
2. ✅ You configured M2M Service Principal authentication
3. ❌ During testing, validation failed
4. ❌ You manually deleted `.github` and `my_job_bundle` folders
5. ❌ You created multiple test branches trying to fix it
6. ❌ Git got confused about what exists and what doesn't
7. ❌ When pushing, you got conflicts and errors
8. ✅ We fixed everything step by step!

---

## Problem #1: Git Index Out of Sync

### What Does This Mean?

**Git Index** = Git's memory of what files exist and their state

Think of Git like a librarian who keeps a catalog of all books:
- When you add a file → Git writes it in the catalog
- When you delete a file → Git should remove it from the catalog
- When you delete manually → Git's catalog still thinks the file exists!

### What Went Wrong

```
Your Action                Git's Understanding
-----------------         ---------------------
Deleted .github/          Still thinks .github/ exists
Deleted my_job_bundle/    Still thinks my_job_bundle/ exists
Re-added them             "Wait, these already exist? Conflict!"
```

### The Symptoms
- "Untracked files" but Git won't add them
- Push rejected
- "Index untracking errors"

### How We Fixed It

**Step 1: Checked Git status**
```bash
git status
# Output: Untracked files: .github/, my_job_bundle/
```

**Step 2: Simply added them fresh**
```bash
git add .
git commit -m "Fix: Restructure DAB bundle"
git push origin feature
```

**Why This Worked:**
Because you were on a clean branch (`feature`) that was already synced with remote, Git treated them as genuinely new files.

---

## Problem #2: Bundle Structure Wrong

### What Is a Databricks Asset Bundle (DAB)?

A DAB is like a **project folder** containing:
- `databricks.yml` - The "table of contents" (main config file)
- `src/` - Your Python code
- `resources/` - Job and pipeline definitions
- `tests/` - Test files

### The Wrong Structure

```
my_job_bundle/
└── dev/                           ❌ Problem: dev/ should NOT be in Git
    ├── files/                     ← Your source code (SHOULD be in Git)
    │   ├── databricks.yml
    │   ├── src/
    │   └── resources/
    ├── artifacts/                 ← Build outputs (should NOT be in Git)
    └── state/                     ← Deployment state (should NOT be in Git)
```

**Why Is This Wrong?**

When you run `databricks bundle deploy`, it creates a `dev/` folder:
- `dev/files/` = copies your source code
- `dev/artifacts/` = compiled/built packages
- `dev/state/` = tracks what's deployed

You should **ONLY commit source code**, not deployment artifacts!

### The Right Structure

```
my_job_bundle/
├── databricks.yml          ✅ Config at root
├── src/                    ✅ Source code
├── resources/              ✅ Resource definitions
├── tests/                  ✅ Tests
└── .gitignore             ✅ Ignores dev/, prod/, artifacts/
```

### What GitHub Actions Expected

Your workflow had:
```yaml
working-directory: ./my_job_bundle
run: databricks bundle validate -t dev
```

It looks for `./my_job_bundle/databricks.yml` ← Must be HERE!

But your file was at `./my_job_bundle/dev/files/databricks.yml` ← WRONG!

### How We Fixed It

**Step 1: Moved files up**
```bash
mv my_job_bundle/dev/files/* my_job_bundle/
```

**Step 2: Deleted deployment folders**
```bash
rm -rf my_job_bundle/dev
```

**Step 3: Created .gitignore**
```
# Ignore deployment artifacts
my_job_bundle/dev/
my_job_bundle/prod/
my_job_bundle/staging/
```

**Why This Matters:**
Now when you run `databricks bundle deploy` again, it will recreate `dev/` folder, but Git will ignore it!

---

## Problem #3: Merge Conflict

### What Is a Merge Conflict?

Imagine two people editing the same document:
- **Person A** (main branch): Deletes page 5
- **Person B** (feature branch): Edits page 5
- **Git**: "I don't know whether to keep or delete page 5!"

### What Happened in Your Case

```
main branch:                feature branch:
.github/ doesn't exist      .github/ exists with your workflow

Git: "Should I delete it or keep it? 🤔"
```

This is called a **modify/delete conflict**.

### The Error Message

```
CONFLICT (modify/delete): .github/workflows/bundle_validate.yml 
deleted in origin/main and modified in HEAD.
```

Translation:
- `main` branch: File doesn't exist (or was deleted)
- `feature` branch: File exists and you modified it
- Git: "Can't auto-merge, YOU decide!"

### How to Resolve Deleted Directory Conflicts

#### Option 1: Keep YOUR version (feature branch)

```bash
# Step 1: Try to merge main into your branch
git merge origin/main
# Output: CONFLICT!

# Step 2: Tell Git "Keep my version"
git add .github/workflows/bundle_validate.yml

# Step 3: Commit the resolution
git commit -m "Merge main - keeping workflow file"

# Step 4: Push
git push origin feature
```

#### Option 2: Delete the file (accept main's deletion)

```bash
# Step 1: After merge conflict appears
git merge origin/main

# Step 2: Tell Git "Delete it"
git rm .github/workflows/bundle_validate.yml

# Step 3: Commit the resolution
git commit -m "Merge main - removing workflow file"

# Step 4: Push
git push origin feature
```

#### Option 3: Abort and try differently

```bash
# Cancel the merge
git merge --abort

# Alternative: Rebase instead of merge
git rebase origin/main
# Handle conflicts one by one
```

### Visual Guide to Conflict Resolution

```
Scenario: File deleted in main, modified in feature

┌─────────────────────────────────────────┐
│  What do you want?                      │
├─────────────────────────────────────────┤
│                                         │
│  ✅ KEEP the file                       │
│     → git add <file>                    │
│                                         │
│  ❌ DELETE the file                     │
│     → git rm <file>                     │
│                                         │
│  🔙 CANCEL merge                        │
│     → git merge --abort                 │
│                                         │
└─────────────────────────────────────────┘

Then: git commit -m "Resolved conflict"
      git push
```

---

## Problem #4: Terraform Key Expiration

### What Is Terraform?

**Terraform** = A tool for managing cloud infrastructure (creating servers, databases, networks, etc.)

**Important:** Databricks Asset Bundles **DO NOT NEED** Terraform for validation!

### The Error

```
Error: error downloading Terraform: unable to verify checksums signature: 
openpgp: key expired
```

### What This Means (In Simple English)

1. Your workflow used `databricks/setup-cli@v0.220.0` (old version)
2. This old version tries to download Terraform
3. Terraform files are "signed" with a digital key (for security)
4. That signing key **expired** (like a credit card expiring)
5. GitHub Actions: "I can't verify this download is safe! ❌"

### Why Did It Try to Download Terraform?

The old `setup-cli` action was designed to support:
- Databricks CLI (what you need)
- Terraform (what you DON'T need)

It tried to install BOTH by default!

### The Fix

**Before:**
```yaml
- name: Setup Databricks CLI
  uses: databricks/setup-cli@v0.220.0  ← Old version, tries to get Terraform
```

**After:**
```yaml
- name: Setup Databricks CLI
  uses: databricks/setup-cli@main      ← Latest version
  with:
    disable-terraform: true            ← Skip Terraform completely!
```

### Why This Works

1. `@main` = Always use the latest version (key is not expired)
2. `disable-terraform: true` = Don't even try to download Terraform
3. Your workflow only needs Databricks CLI anyway!

### Bonus: What If You Actually Needed Terraform?

If you were using Terraform, the fix would be different:
```yaml
- name: Setup Terraform
  uses: hashicorp/setup-terraform@v3  ← Official Terraform action
```

But for DAB validation, you don't need this!

---

## Key Concepts Explained

### 1. Git Index vs Working Directory

```
┌─────────────────────────────────────────────────────┐
│  YOUR COMPUTER                                      │
│                                                     │
│  Working Directory        Git Index       Repository│
│  (files you see)          (staging)       (commits) │
│                                                     │
│      file.txt                                       │
│         │                                           │
│         │ git add                                   │
│         └────────────────> file.txt                  │
│                              │                      │
│                              │ git commit           │
│                              └────────────> file.txt  │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Working Directory** = What you see in your file explorer
**Git Index** = Files ready to be committed (staged)
**Repository** = Committed history

### 2. Merge vs Rebase

#### Merge (What We Used)

```
main:     A─B─C───────M
               \       /
feature:        D─E─F

Result: Creates merge commit M
```

#### Rebase (Alternative)

```
main:     A─B─C
feature:        D─E─F  (moved to tip of main)

Result: Linear history, no merge commit
```

### 3. CI/CD Pipeline

```
┌──────────────────────────────────────────────────────┐
│  Continuous Integration / Continuous Deployment      │
└──────────────────────────────────────────────────────┘

1. You write code
   ↓
2. Push to GitHub
   ↓
3. GitHub Actions automatically:
   • Checks out code
   • Runs validation
   • Runs tests
   • Reports pass/fail
   ↓
4. If pass → Safe to merge!
   If fail → Fix issues first
```

**Benefits:**
- Catches errors before merging
- No manual validation needed
- Team confidence in code quality

### 4. Databricks Asset Bundle Structure

```
┌─────────────────────────────────────────────────┐
│  SOURCE CODE (commit to Git)                    │
├─────────────────────────────────────────────────┤
│  my_bundle/                                     │
│  ├── databricks.yml          ← Main config     │
│  ├── src/                    ← Your code       │
│  ├── resources/              ← Job definitions │
│  └── tests/                  ← Unit tests      │
└─────────────────────────────────────────────────┘

         ↓ databricks bundle deploy

┌─────────────────────────────────────────────────┐
│  DEPLOYMENT ARTIFACTS (do NOT commit)           │
├─────────────────────────────────────────────────┤
│  my_bundle/dev/                                 │
│  ├── files/        ← Copy of source            │
│  ├── artifacts/    ← Compiled .whl files       │
│  └── state/        ← Deployment tracking       │
└─────────────────────────────────────────────────┘
                ↓
        Deployed to Databricks workspace
```

### 5. GitHub Actions Secrets

**What Are Secrets?**
Secure storage for sensitive info (passwords, API keys)

**Your Secrets:**
```yaml
DATABRICKS_HOST: https://your-workspace.cloud.databricks.com
DATABRICKS_CLIENT_ID: abc123...
DATABRICKS_CLIENT_SECRET: xyz789...  ← Never hardcode this!
```

**How They Work:**
1. You store them in GitHub (Settings → Secrets)
2. Workflow accesses them with `${{ secrets.NAME }}`
3. GitHub hides them in logs (shows ****)

---

## How to Handle Similar Issues in Future

### Checklist: Before Pushing to GitHub

```bash
# 1. Check what files changed
git status

# 2. Review the changes
git diff

# 3. Check what will be committed
git add --dry-run --all

# 4. Make sure .gitignore is correct
cat .gitignore

# 5. Add files
git add .

# 6. Commit with clear message
git commit -m "Descriptive message"

# 7. Pull latest changes first
git pull origin main

# 8. Resolve any conflicts
# 9. Push
git push origin feature
```

### When You Get Merge Conflicts

```bash
# ✅ DO:
1. Don't panic!
2. Read the conflict message carefully
3. Decide which version you want
4. Use `git add` (keep) or `git rm` (delete)
5. Commit and push

# ❌ DON'T:
1. Delete files manually without telling Git
2. Force push without understanding why
3. Ignore conflicts and hope they go away
4. Mix up which branch has which version
```

### Bundle Development Best Practices

```bash
# 1. Always work in source directory
cd my_job_bundle

# 2. Edit source files, not deployed files
vim src/main.py          ✅ Good
vim dev/files/src/main.py ❌ Bad - this gets overwritten!

# 3. Validate before deploying
databricks bundle validate -t dev

# 4. Deploy to test environment first
databricks bundle deploy -t dev

# 5. Test it works
databricks bundle run -t dev

# 6. Only then deploy to production
databricks bundle deploy -t prod
```

### GitHub Actions Troubleshooting Steps

```
Workflow Failed?
    ↓
1. Click on failed step
    ↓
2. Read error message (bottom of logs)
    ↓
3. Common errors:
    
    ❌ "File not found: databricks.yml"
    → Check working-directory in workflow
    → Verify file is at correct path
    
    ❌ "Authentication failed"
    → Check secrets are set in GitHub
    → Verify secret names match workflow
    
    ❌ "openpgp: key expired" / Terraform errors
    → Update setup-cli version
    → Add disable-terraform: true
    
    ❌ "Bundle validation failed"
    → Run locally: databricks bundle validate
    → Fix errors in databricks.yml
    → Push fixed version
```

---

## Summary of All Fixes

### Fix #1: Git Index Sync
**Problem:** Git confused about deleted/re-added files
**Solution:** Clean add and commit from scratch

### Fix #2: Bundle Structure  
**Problem:** Files nested in dev/files/ instead of root
**Solution:** Moved files up, deleted dev/, added .gitignore

### Fix #3: Merge Conflict
**Problem:** File existed in feature but not in main
**Solution:** Merged main into feature, kept the file with `git add`

### Fix #4: Terraform Key Expiration
**Problem:** Old setup-cli tried to download Terraform with expired key
**Solution:** Updated to latest version + disabled Terraform

---

## Final Working Setup

### Repository Structure
```
DATA-ENGINEERING-ASSIGNMENTS-CYNTEXA/
├── .github/
│   └── workflows/
│       └── bundle_validate.yml    ✅ CI/CD workflow
├── .gitignore                      ✅ Ignores deployment artifacts
├── my_job_bundle/
│   ├── databricks.yml             ✅ Bundle config
│   ├── src/                       ✅ Source code
│   ├── resources/                 ✅ Job definitions
│   └── tests/                     ✅ Tests
└── (assignment folders...)
```

### GitHub Actions Workflow
```yaml
name: Validate Databricks Asset Bundle

on:
  pull_request:
    branches: [main, dev, feature]

jobs:
  validate-bundle:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: databricks/setup-cli@main
        with:
          disable-terraform: true
      - name: Run Bundle Validate
        env:
          DATABRICKS_HOST: ${{ secrets.DATABRICKS_HOST }}
          DATABRICKS_CLIENT_ID: ${{ secrets.DATABRICKS_CLIENT_ID }}
          DATABRICKS_CLIENT_SECRET: ${{ secrets.DATABRICKS_CLIENT_SECRET }}
        working-directory: ./my_job_bundle
        run: databricks bundle validate -t dev
```

### Git Workflow
```bash
# Create feature branch
git checkout -b feature/my-changes

# Make changes
vim my_job_bundle/src/main.py

# Add and commit
git add .
git commit -m "Add new feature"

# Push to GitHub
git push origin feature/my-changes

# Create Pull Request
# → GitHub Actions runs automatically
# → If pass: merge to main
# → If fail: fix and push again
```

---

## Questions?

If you encounter issues:
1. Check this guide first
2. Run `git status` to see current state
3. Read error messages carefully
4. Search GitHub Actions logs for specific errors
5. Ask for help with specific error messages

**Remember:** Every error message is trying to help you! Read it carefully.

---

**Document Purpose:** Troubleshooting reference for DAB CI/CD setup
