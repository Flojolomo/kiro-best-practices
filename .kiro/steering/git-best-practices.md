---
title: Git Best Practices
inclusion: always
---

# Git Best Practices

## Commit Messages
- Use conventional commit format: `type(scope): description`
- Types: feat, fix, docs, style, refactor, test, chore
- Keep first line under 50 characters
- Use imperative mood ("Add feature" not "Added feature")
- Include body for complex changes

## Branching
- Use feature branches for new development
- Keep main/master branch stable and deployable
- Use descriptive branch names (feature/user-auth, fix/login-bug)
- Delete merged branches to keep repository clean

## Workflow (REQUIRED)
- Pull latest changes before starting work
- **Every change MUST be committed and pushed** — no local-only work
- **If no PR is open, create one** after the first push to a feature branch
- **If a previous PR was merged or closed, create a new branch and open a new PR** — never reuse closed/merged PR branches
- Commit frequently with logical chunks
- Use interactive rebase to clean up history before merging
- Review code before merging (pull requests)

## Repository Management
- Use .gitignore to exclude build artifacts and secrets
- Keep repository size manageable (use Git LFS for large files)
- Tag releases with semantic versioning
- Document branching strategy in README

## Security
- Never commit secrets, API keys, or passwords
- Use environment variables for configuration
- Review commits for sensitive information
- Use signed commits when possible