## Git Workflow and Commit Attribution

### 1. Commit Ownership
- Codex must be the primary author of commits containing changes independently implemented by Codex.
- Use a Git author identity only if it is verified and recognized by GitHub.
- Never use an unverified email or impersonate another account.
- Do not add unnecessary Co-authored-by trailers.

### 2. Dynamic Commit Messages
- Analyze the actual changes before creating a commit.
- Determine the appropriate Conventional Commit type:
  - feat: New functionality
  - fix: Bug fixes
  - docs: Documentation changes
  - refactor: Code restructuring
  - test: Tests
  - chore: Maintenance
  - build: Build system changes
  - ci: CI/CD changes
  - perf: Performance improvements
- Generate a concise, meaningful commit message describing the actual changes.
- Never reuse a fixed commit message.
- Use an appropriate scope when relevant.

### 3. Commit Workflow
- Review git status and git diff before committing.
- Stage only files related to the completed task.
- Run relevant tests and validation.
- Create commits only when a meaningful unit of work is complete.
- Never create empty or artificial commits.

### 4. Git Safety
- Never modify global Git configuration.
- Never rewrite existing commit history.
- Never force-push.
- Preserve unrelated user changes.
- Push only when explicitly requested.

### 5. Attribution Verification
- Verify the author identity before creating commits.
- After committing, verify the commit author and message.
- Never claim Codex will appear in GitHub Contributors unless GitHub actually recognizes the author identity.

### 6. Codex Identity Setup
- Codex is the primary author for work it independently implements only when its GitHub identity and commit email are verified.
- Confirm the exact commit email from reliable GitHub account information, such as the account's email settings or an authenticated account source. Do not derive or assume an email from a username, account ID, or expected address format alone.
- If the exact identity cannot be verified, do not configure or use a guessed Codex email. Continue the requested project work, but explain the identity limitation before creating a commit that would claim Codex as its primary author.
- Do not change global Git identity. Configure repository-local author settings only after the exact Codex identity has been verified and the user has requested that setup.
- Do not add the user's account as a co-author unless the user contributed to that specific change.
