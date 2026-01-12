---
title: Pull Request Workflow
description: Submitting changes to db-yard.
---

# Pull Request Workflow

Submitting changes to db-yard.

## Before You Start

1. **Check existing issues** - Is someone already working on this?
2. **Open an issue first** - For significant changes, discuss the approach
3. **Fork the repository** - Work in your own fork

## Creating a Pull Request

### Step 1: Create a Branch

```bash
# Sync with upstream
git checkout main
git pull origin main

# Create feature branch
git checkout -b feature/my-feature
```

Branch naming:
- `feature/description` - New features
- `fix/description` - Bug fixes
- `docs/description` - Documentation
- `refactor/description` - Code refactoring

### Step 2: Make Changes

Follow the [Code Guidelines](./code-guidelines.md).

### Step 3: Write Tests

Add tests for your changes:

```bash
# Run tests
deno test

# Check coverage
deno test --coverage=coverage
deno coverage coverage
```

### Step 4: Format and Lint

```bash
deno fmt
deno lint
```

### Step 5: Commit

Use conventional commits:

```bash
git add .
git commit -m "feat: add port range configuration"
```

Commit types:
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation
- `test:` - Tests
- `refactor:` - Refactoring
- `chore:` - Maintenance

### Step 6: Push

```bash
git push origin feature/my-feature
```

### Step 7: Open PR

1. Go to GitHub
2. Click "Compare & pull request"
3. Fill in the template

## PR Template

```markdown
## Summary

Brief description of what this PR does.

## Changes

- Added X
- Fixed Y
- Updated Z

## Testing

How was this tested?

- [ ] Unit tests added/updated
- [ ] Manual testing performed
- [ ] All tests pass

## Checklist

- [ ] Code follows project style
- [ ] Tests added for new functionality
- [ ] Documentation updated if needed
- [ ] Commit messages follow conventions
```

## Review Process

### What Reviewers Look For

1. **Correctness** - Does it work?
2. **Tests** - Is it tested?
3. **Style** - Does it follow guidelines?
4. **Documentation** - Is it documented?
5. **Simplicity** - Is it simple?

### Responding to Feedback

- Address all comments
- Push new commits (don't force-push during review)
- Mark conversations resolved when fixed
- Ask for clarification if needed

### After Approval

Once approved:
1. Squash commits if requested
2. Maintainer will merge

## PR Tips

### Keep PRs Small

- Focus on one thing
- Easier to review
- Faster to merge

### Write Good Descriptions

- Explain the "why"
- Link to related issues
- Include screenshots for UI changes

### Test Thoroughly

```bash
# Run all tests
deno test

# Test your specific changes
deno test lib/your-module_test.ts

# Manual testing
./bin/yard.ts start
```

### Update Documentation

If your change affects:
- CLI commands → Update `docs/reference/cli.md`
- Configuration → Update `docs/reference/configuration.md`
- New features → Add appropriate docs

## Common Issues

### Merge Conflicts

```bash
# Sync with main
git fetch origin
git rebase origin/main

# Resolve conflicts
# Edit files to resolve
git add .
git rebase --continue

# Force push (only during rebase)
git push --force-with-lease
```

### Failed CI

Check the CI logs:
1. Tests passing?
2. Lint passing?
3. Format correct?

Fix issues and push again.

### Stale PR

If your PR hasn't been reviewed:
- Ping maintainers politely
- Check if more information is needed
- Consider if the PR is too large

## After Merge

1. Delete your branch
2. Sync your fork
3. Celebrate!

```bash
git checkout main
git pull origin main
git branch -d feature/my-feature
```

---

Thank you for contributing to db-yard!