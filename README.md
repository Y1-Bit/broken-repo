# Git Debugging Exercise - Solution

## Overview

This repository was given with several intentional Git issues. Below is a record of what was found, how each issue was fixed, and the reasoning behind each approach.


## Task 1: Inspect the History

What was found:

- Commit 2ee27fa "quick fix": Mixed three unrelated changes in one commit - a README update, a new config.txt file, and a notes.txt status update. These should have been separate commits.
- Commit b2f0f15 "refactor: Update todo list with progress": Introduced an incorrect line in todo.txt: "This line should not be here - it was added by mistake."
- Commit 5f76142 "feat: Add configuration file": Hardcoded an API key in secrets.txt: API_KEY=sk_live_1234567890abcdefghijklmnopqrstuvwxyz

Commands used:

    git log --oneline --all --graph
    git log --stat
    git show <commit-hash>
    git log --all -p | grep -i "api_key\|secret"


## Task 2: Clean Up Commit History

Problem: Commit 2ee27fa ("quick fix") mixed three unrelated changes:
1. Added a Features section to README.md
2. Created a new config.txt file
3. Updated project status in notes.txt

Fix: Used interactive rebase to split it into three logical commits:

    git rebase -i 12516f6
    # Mark 2ee27fa as 'edit', then:
    git reset HEAD~1
    git add README.md  && git commit -m "docs: Add features section to README"
    git add config.txt && git commit -m "feat: Add project configuration file"
    git add notes.txt  && git commit -m "docs: Update project status notes"
    git rebase --continue

Result: Three clean, focused commits replacing one messy one.


## Task 3: Fix the Mistake Safely

Problem: Commit b2f0f15 added a spurious line to todo.txt:
"This line should not be here - it was added by mistake."

Fix: Used git revert to undo the commit without rewriting history:

    git revert b2f0f15 --no-edit

Why revert and not reset?

git reset removes commits from history - safe only on local, unpushed branches. Since this commit was already part of shared history, resetting would require a force push, which overwrites others' work.

git revert creates a new commit that undoes the change. History stays intact, the fix is visible, and no one's work is disrupted.


## Task 4: Resolve a Merge Conflict

Problem: main and feature/login both modified notes.txt with different content that needed to be combined.

Fix: Merged feature/login into main and manually resolved the conflict by keeping both sides:

    git merge feature/login
    # Edited notes.txt to preserve both sets of changes
    git add notes.txt
    git commit -m "merge: Resolve conflict between main and feature/login"

Both the performance/database notes (from main) and the authentication notes (from feature/login) were preserved in the final file.


## Task 5: Remove Leaked Secrets

Problem: Commit 5f76142 hardcoded a live API key in secrets.txt:
API_KEY=sk_live_1234567890abcdefghijklmnopqrstuvwxyz

Fix (for this exercise): Replaced the real key with a placeholder and committed:

    # Edited secrets.txt: API_KEY=YOUR_API_KEY_HERE
    git add secrets.txt
    git commit -m "fix: Remove hardcoded API key from configuration file"

What to do in a real production scenario:

1. Revoke the key immediately - treat it as fully compromised from the moment it was committed, even if the push was caught quickly.
2. Remove from current state - edit the file, commit the fix (done above).
3. Rewrite history to remove it from all past commits:

       git filter-repo --replace-text <(echo "sk_live_...==>REDACTED")

4. Force push the rewritten history to all remotes (coordinate with the team - everyone must re-clone).
5. Notify the team and rotate any related credentials.
6. Prevent recurrence:
   - Add secret files to .gitignore
   - Use environment variables or a secrets manager (AWS Secrets Manager, HashiCorp Vault)
   - Add a pre-commit hook with tools like git-secrets or truffleHog to scan for secrets before every commit


## Task 6: Rebase and Sync

Problem: feature/login was behind main after several commits had been added to main.

Fix:

    git checkout feature/login
    git rebase main
    git rebase --continue

Why rebase and not merge?

Rebase produces a linear, clean history with no extra merge commits just for syncing. Merge would preserve the branch structure but add an unnecessary merge commit. Rebase was the right choice here because feature/login is a local branch that has not been shared with others, so rewriting its history is safe. The resulting branch reads as if it was always built on top of the latest main.


## Task 7: Recover Lost Commit (Bonus)

Problem: A git reset --hard HEAD~1 on feature/login removed commit 104d95f ("feat: Add helper documentation"), which added helpers.txt.

Fix: Used git reflog to find the lost commit hash, then cherry-picked it back:

    git reflog
    # Found: 104d95f HEAD@{87}: commit: feat: Add helper documentation
    # Followed by: 075274b HEAD@{86}: reset: moving to HEAD~1

    git cherry-pick 104d95f

What the lost commit contained: A new file helpers.txt with formatting guidelines and style guide references.

Why reflog works: Git does not immediately delete commits when they are reset - they remain as dangling objects for 30-90 days (depending on GC settings). git reflog records every movement of HEAD, so you can trace back to commits that are no longer reachable from any branch.


## How to Avoid These Issues on a Team

- Atomic commits: Each commit should do one thing. Use git add -p to stage changes in hunks when multiple things were changed at once.
- Branch protection rules: Require PR reviews before merging to main. This catches messy commits and incorrect content before they land.
- Pre-commit hooks: Use tools like pre-commit, git-secrets, or husky to scan for secrets, lint code, and enforce commit message conventions automatically.
- Conventional commits: Enforce a commit message format (feat:, fix:, docs:, etc.) via tooling like commitlint. This keeps history readable and enables automated changelogs.
- Never commit secrets: Store secrets in environment variables or a secrets manager. Add .env and credential files to .gitignore by default.
- Code review culture: Reviewers should check not just the code but also the commit hygiene - message clarity, scope, and whether sensitive data was accidentally included.
