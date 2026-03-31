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

### Runner Setup

The checkout step must use `persist-credentials: true` (default for `actions/checkout@v4`) so that subsequent `git push` commands authenticate via `GITHUB_TOKEN`. After checkout, add and fetch the upstream remote:

```bash
git remote add upstream https://github.com/BerriAI/litellm.git
git fetch upstream --tags
```

This makes the upstream tag available locally by name (e.g. `refs/tags/v1.82.6`) so the rebase target does not need to be passed as a raw SHA.

### Steps

1. Call `GET /repos/BerriAI/litellm/releases/latest` via GitHub API to get the tag name (e.g. `v1.82.6`) and its commit SHA.
2. Check whether `middlendian/v{tag}` already exists on origin. If it does, fail with a clear message (avoids overwriting in-progress conflict resolution).
3. Check out `middlendian/custom` and create a new local branch `middlendian/v{tag}` from it.
4. Run `git rebase refs/tags/v{tag}` — `git fetch upstream --tags` places upstream tags into the local `refs/tags/` namespace, not `refs/remotes/upstream/`, so `refs/tags/v{tag}` is the correct named ref to use.
5. **On success:** Force-push `middlendian/v{tag}` to origin. Print the tag name and SHA. Exit 0.
6. **On conflict:**
   - Run `git rebase --abort` to restore the branch to a clean state (same HEAD as `middlendian/custom`).
   - Push `middlendian/v{tag}` to origin at that clean state (gives a local starting point).
   - Print: conflicting files, failing commit SHA and message.
   - Print the exact local reproduce-and-continue command:
     ```
     git fetch origin
     git fetch upstream --tags   # if you have the upstream remote configured
     git checkout middlendian/v{tag}
     git rebase upstream/v{tag}
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

### Runner Setup

Same as Workflow 1: `actions/checkout@v4` with `persist-credentials: true`, then fetch the upstream remote and its tags.

### Steps

1. Parse the upstream version from the input branch name (e.g. `1.82.6` from `middlendian/v1.82.6`).
2. Fail if the release tag `v{version}-middlendian` already exists on origin. This prevents re-triggering the Docker build for an already-released version.
3. Fetch the upstream tag's current commit SHA via GitHub API: `GET /repos/BerriAI/litellm/git/refs/tags/v{version}`.
4. Verify the upstream tag has not been mutated since the branch was created: check that `{upstream-tag-sha}` is an ancestor of the input branch HEAD using `git merge-base --is-ancestor {upstream-tag-sha} {branch-head-sha}`. This confirms the branch was genuinely rebased on top of that exact upstream commit. Fail if not.
5. Force-push `middlendian/custom` to the HEAD of the input branch.
6. Create and push tag `v{version}-middlendian`. This triggers the existing `middlendian_docker_publish.yml` workflow.

### Safety checks

- Fails if the input branch does not exist.
- Fails if `v{version}-middlendian` tag already exists (step 2).
- Fails if the upstream tag SHA cannot be verified (step 4).
- Prints the upstream tag SHA and branch HEAD SHA to the workflow log for auditability.

## Existing Workflow: `middlendian_docker_publish.yml`

Unchanged. Already triggers on `v*-middlendian` tags and publishes to GHCR as `middlendian/litellm-custom`. The release workflow feeds into this automatically.

**Note on Docker image tags:** The existing workflow uses `type=semver,pattern={{version}}` in the metadata action. The tag `v1.82.6-middlendian` has a non-semver suffix, so Docker's metadata action will strip it and produce `1.82.6` as the image tag. This is pre-existing behavior and not changed by this design, but worth being aware of.

## GitHub Permissions

Both workflows require `contents: write` on the repository (to push branches and create tags).

**Important:** If `middlendian/custom` has branch protection rules enabled, a `GITHUB_TOKEN`-based force-push will be blocked regardless of permissions. Either:
- Disable branch protection on `middlendian/custom`, or
- Add a branch protection exception for GitHub Actions

Similarly, if tags matching `v*-middlendian` are protected, the release workflow will fail at tag creation. Ensure no tag protection rules cover this pattern.

## Typical Usage Flow

```
1. New upstream release appears (e.g. v1.82.6)
2. Trigger middlendian-rebase.yml
   → Creates middlendian/v1.82.6 rebased on upstream tag
   → Green: branch is ready to test/review
   → Red: check logs, resolve conflicts locally, push branch manually
3. Test/validate middlendian/v1.82.6 (deploy to staging, etc.)
4. Trigger middlendian-release.yml with branch=middlendian/v1.82.6
   → Verifies upstream tag integrity
   → Updates middlendian/custom
   → Creates v1.82.6-middlendian tag
   → Docker build triggers automatically
```
