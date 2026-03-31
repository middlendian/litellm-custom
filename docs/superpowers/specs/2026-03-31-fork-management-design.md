# Fork Management Design: Middlendian LiteLLM Customizations

**Date:** 2026-03-31
**Status:** Approved

## Overview

Manage a set of custom patches on top of upstream LiteLLM releases using a branch-per-upstream-tag strategy. Two `workflow_dispatch` GitHub Actions workflows handle rebasing onto new upstream releases and cutting releases.

## Branch Model

```
upstream tag (e.g. v1.82.6)
        │
        └─► middlendian/v1.82.6       ← rebase staging branch (work in progress)
                    │
                    └─► middlendian/custom   ← last released state (updated only at release)
                                │
                                └─► v1.82.6-middlendian tag ← triggers Docker build
```

### Rules

- `middlendian/custom` is **never touched by the rebase workflow**. It always equals the HEAD of the most recently released `middlendian/v*` branch.
- `middlendian/v{tag}` branches are ephemeral staging branches. They accumulate the rebased patch set for a given upstream version. They persist in the repo as an audit trail.
- A new `middlendian/v{tag}` branch is created from `middlendian/custom` (not from upstream directly), so the starting point always includes the current patch set.

## Workflow 1: `middlendian-rebase.yml`

**Trigger:** `workflow_dispatch` (manual, from GitHub Actions UI)

**Purpose:** Create a new version branch rebased onto the latest upstream release.

### Steps

1. Call `GET /repos/BerriAI/litellm/releases/latest` via GitHub API to get the tag name and commit SHA.
2. Check whether `middlendian/v{tag}` already exists on origin. If it does, fail with a clear message (avoids overwriting in-progress conflict resolution).
3. Check out `middlendian/custom` and create a new local branch `middlendian/v{tag}` from it.
4. Run `git rebase {upstream-tag-sha}`.
5. **On success:** Force-push `middlendian/v{tag}` to origin. Print the tag name and SHA. Exit 0.
6. **On conflict:**
   - Run `git rebase --abort` to restore the branch to a clean state (same HEAD as `middlendian/custom`).
   - Push `middlendian/v{tag}` to origin at that clean state (gives a local starting point).
   - Print: conflicting files, failing commit SHA and message.
   - Print the exact local reproduce-and-continue command:
     ```
     git fetch origin
     git checkout middlendian/v{tag}
     git rebase {upstream-tag-sha}
     # resolve conflicts, then: git rebase --continue
     # when done: git push --force origin middlendian/v{tag}
     ```
   - Exit 1 (workflow shows as failed — visible signal in GitHub UI).

### What is NOT done

- Does not touch `middlendian/custom`.
- Does not create any release tag.
- Does not trigger Docker builds.

## Workflow 2: `middlendian-release.yml`

**Trigger:** `workflow_dispatch` (manual, from GitHub Actions UI)

**Input:** `branch` — the `middlendian/v{tag}` branch to release (e.g. `middlendian/v1.82.6`). No default; must be supplied explicitly to avoid accidental releases.

**Purpose:** Promote a staging branch to `middlendian/custom` and cut a release tag.

### Steps

1. Parse the upstream version from the input branch name (e.g. `1.82.6` from `middlendian/v1.82.6`).
2. Fetch the upstream tag's current commit SHA via GitHub API: `GET /repos/BerriAI/litellm/git/refs/tags/v{version}`.
3. Verify that the upstream tag SHA matches the merge-base of the input branch and `upstream/v{version}`. This guards against upstream tag mutation (force-pushed tags).
4. Force-push `middlendian/custom` to the HEAD of the input branch.
5. Create and push tag `v{version}-middlendian`. This triggers the existing `middlendian_docker_publish.yml` workflow.

### Safety checks

- Fails if the input branch does not exist.
- Fails if the upstream tag SHA cannot be verified (step 3).
- Prints the tag SHA and branch HEAD SHA to the workflow log for auditability.

## Existing Workflow: `middlendian_docker_publish.yml`

Unchanged. Already triggers on `v*-middlendian` tags and publishes to GHCR as `middlendian/litellm-custom`. The release workflow feeds into this automatically.

## Typical Usage Flow

```
1. New upstream release appears (e.g. v1.82.6)
2. Trigger middlendian-rebase.yml
   → Creates middlendian/v1.82.6 rebased on upstream tag
   → Green: branch is ready to test/review
   → Red: check logs, resolve conflicts locally, push branch manually
3. Test/validate middlendian/v1.82.6 (deploy to staging, etc.)
4. Trigger middlendian-release.yml with branch=middlendian/v1.82.6
   → Updates middlendian/custom
   → Creates v1.82.6-middlendian tag
   → Docker build triggers automatically
```

## Security Considerations

- The release workflow verifies the upstream tag SHA has not changed since the branch was created. This prevents releasing against a silently mutated upstream tag.
- Both workflows require `contents: write` permission (scoped to the repo). No external secrets beyond `GITHUB_TOKEN` are needed for branch/tag operations.
- Docker publish credentials (GHCR) remain in the existing workflow, not these new ones.
