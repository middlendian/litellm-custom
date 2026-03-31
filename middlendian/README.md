# Middlendian Fork Management

This directory contains all files specific to the middlendian fork of [BerriAI/litellm](https://github.com/BerriAI/litellm). Everything here is isolated to avoid conflicts when rebasing onto upstream tags.

## Branch Model

```
upstream tag (e.g. v1.62.6)
        │
        └─► middlendian/v1.62.6   ← staging branch (rebase work, conflict resolution)
                    │
                    └─► middlendian/custom  ← last released state (never modified directly)
                                │
                                └─► v1.62.6-middlendian tag ← triggers Docker build → GHCR
```

- **`middlendian/custom`** — always equals the most recently released staging branch. Never commit to this directly.
- **`middlendian/v{tag}`** — one branch per upstream release. Created by the rebase workflow, kept as an audit trail.
- **`v{version}-middlendian`** — release tag. Created by the release workflow; triggers Docker publish.

## Local Setup

Add the upstream remote once:

```bash
git remote add upstream https://github.com/BerriAI/litellm.git
git fetch upstream --tags
```

## Viewing Your Customizations

To see a diff of all middlendian commits on top of the current upstream base:

```bash
git diff $(git merge-base --fork-point upstream/main HEAD)..HEAD
```

To see just the commit list:

```bash
git log $(git merge-base --fork-point upstream/main HEAD)..HEAD --oneline
```

If you are on a staging branch (`middlendian/v{tag}`), you can also diff against the upstream tag directly:

```bash
# Replace v1.62.6 with the actual tag
git diff refs/tags/v1.62.6..HEAD
git log refs/tags/v1.62.6..HEAD --oneline
```

## Common Workflows

### 1. Rebase onto a new upstream release

Trigger the **Middlendian — Rebase onto upstream release** workflow from the GitHub Actions UI:

- Go to **Actions → Middlendian — Rebase onto upstream release → Run workflow**
- This creates `middlendian/v{latest-tag}` rebased on the upstream release
- Green run: branch is ready to test
- Red run: conflicts found — see [Resolving Conflicts](#resolving-conflicts) below

### 2. Resolving conflicts after a failed rebase

When the rebase workflow fails, it prints the conflicting files and a reproduce command in the workflow log. To resolve locally:

```bash
git fetch origin
git fetch upstream --tags

# Check out the staging branch (pushed at middlendian/custom HEAD as a starting point)
git checkout middlendian/v{tag}    # e.g. middlendian/v1.62.6

# Start the rebase — git will stop at each conflict
git rebase refs/tags/v{tag}        # e.g. refs/tags/v1.62.6

# For each conflict:
#   1. Open the conflicting files and resolve the markers
#   2. Stage the resolved files:
git add <resolved-files>
#   3. Continue to the next commit:
git rebase --continue

# If you want to skip a commit that is no longer relevant:
git rebase --skip

# If you need to bail out entirely and start over:
git rebase --abort

# When the rebase completes cleanly, push the result:
git push --force origin middlendian/v{tag}
```

### 3. Release a staging branch

Once `middlendian/v{tag}` is rebased, tested, and ready to ship:

- Go to **Actions → Middlendian — Release → Run workflow**
- Enter the staging branch name (e.g. `middlendian/v1.62.6`)
- The workflow:
  1. Verifies the upstream tag hasn't been mutated
  2. Force-pushes `middlendian/custom` to the staging branch HEAD
  3. Creates the `v{version}-middlendian` tag → triggers Docker publish

### 4. Check what is in a staging branch vs upstream

```bash
git fetch origin
git fetch upstream --tags

# Commits unique to the staging branch
git log refs/tags/v{tag}..origin/middlendian/v{tag} --oneline

# Full diff
git diff refs/tags/v{tag}..origin/middlendian/v{tag}
```

### 5. Manually create a staging branch (without the workflow)

```bash
git fetch upstream --tags
git checkout -b middlendian/v{tag} origin/middlendian/custom
git rebase refs/tags/v{tag}
# resolve any conflicts (see above)
git push --force origin middlendian/v{tag}
```

## File Placement Convention

All files specific to this fork must be placed under `middlendian/`. Files that must live outside this directory (e.g. `.github/workflows/`) use the `middlendian-` filename prefix (e.g. `middlendian-rebase.yml`). This keeps fork-specific files isolated from upstream paths and minimises rebase conflicts.

## Workflows

| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| Rebase | `.github/workflows/middlendian-rebase.yml` | Manual | Create `middlendian/v{tag}` rebased on latest upstream release |
| Release | `.github/workflows/middlendian-release.yml` | Manual | Promote staging branch to `middlendian/custom`, cut release tag |
| Docker publish | `.github/workflows/middlendian_docker_publish.yml` | `v*-middlendian` tag push | Build and push image to GHCR |
