# Fork Management Workflows Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement two `workflow_dispatch` GitHub Actions workflows — one to rebase the fork's patch set onto the latest upstream LiteLLM release, and one to promote a staging branch to `middlendian/custom` and cut a versioned release tag.

**Architecture:** Two standalone workflow files in `.github/workflows/`. The rebase workflow creates `middlendian/v{tag}` branches from `middlendian/custom` and rebases them onto upstream tags. The release workflow promotes a staging branch to `middlendian/custom` and creates a `v{version}-middlendian` tag, which triggers the existing Docker publish workflow.

**Tech Stack:** GitHub Actions, bash, GitHub REST API (`/releases/latest`, `/git/refs/tags/`), git

**Spec:** `middlendian/docs/superpowers/specs/2026-03-31-fork-management-design.md`

---

## Chunk 1: Rebase Workflow

### Task 1: `middlendian-rebase.yml`

**Files:**
- Create: `.github/workflows/middlendian-rebase.yml`

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/middlendian-rebase.yml` with the following content:

```yaml
name: Middlendian — Rebase onto upstream release

on:
  workflow_dispatch:

jobs:
  rebase:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout repo (full history)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure git identity
        run: |
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git config user.name "github-actions[bot]"

      - name: Fetch upstream remote and tags
        run: |
          git remote add upstream https://github.com/BerriAI/litellm.git
          git fetch upstream --tags

      - name: Get latest upstream release
        id: release
        run: |
          RESPONSE=$(curl -sf \
            -H "Authorization: Bearer ${{ secrets.GITHUB_TOKEN }}" \
            -H "Accept: application/vnd.github+json" \
            https://api.github.com/repos/BerriAI/litellm/releases/latest)
          TAG_NAME=$(echo "$RESPONSE" | jq -r '.tag_name')
          if [ -z "$TAG_NAME" ] || [ "$TAG_NAME" = "null" ]; then
            echo "::error::Failed to fetch latest upstream release tag"
            exit 1
          fi
          echo "tag=$TAG_NAME" >> "$GITHUB_OUTPUT"
          echo "Latest upstream release: $TAG_NAME"

      - name: Check branch does not already exist
        run: |
          TAG="${{ steps.release.outputs.tag }}"
          if git ls-remote --exit-code origin "refs/heads/middlendian/$TAG" > /dev/null 2>&1; then
            echo "::error::Branch middlendian/$TAG already exists on origin. Delete it first if you need to re-run."
            exit 1
          fi
          echo "Branch middlendian/$TAG does not yet exist — proceeding."

      - name: Create staging branch from middlendian/custom
        run: |
          TAG="${{ steps.release.outputs.tag }}"
          git fetch origin middlendian/custom
          git checkout -b "middlendian/$TAG" "origin/middlendian/custom"

      - name: Attempt rebase onto upstream tag
        id: rebase
        run: |
          TAG="${{ steps.release.outputs.tag }}"
          set +e
          git rebase "refs/tags/$TAG"
          REBASE_EXIT=$?
          set -e

          if [ $REBASE_EXIT -eq 0 ]; then
            echo "result=success" >> "$GITHUB_OUTPUT"
          else
            # Capture conflict info BEFORE aborting (state is lost after abort)
            CONFLICTING=$(git diff --name-only --diff-filter=U | tr '\n' ' ')
            STOPPED_SHA=$(cat .git/rebase-merge/stopped-sha 2>/dev/null || echo "unknown")
            STOPPED_MSG=$(git log --format="%s" -1 "$STOPPED_SHA" 2>/dev/null || echo "unknown")

            echo "conflicting_files=$CONFLICTING" >> "$GITHUB_OUTPUT"
            echo "stopped_sha=$STOPPED_SHA" >> "$GITHUB_OUTPUT"
            # Escape newlines/special chars for GITHUB_OUTPUT
            echo "stopped_msg=$(echo "$STOPPED_MSG" | head -1)" >> "$GITHUB_OUTPUT"
            echo "result=conflict" >> "$GITHUB_OUTPUT"

            git rebase --abort
          fi

      - name: Push branch — success
        if: steps.rebase.outputs.result == 'success'
        run: |
          TAG="${{ steps.release.outputs.tag }}"
          git push --force origin "middlendian/$TAG"
          echo "✅ Rebase succeeded. Branch middlendian/$TAG is ready."
          echo "   Upstream tag:  $TAG"
          echo "   Branch HEAD:   $(git rev-parse HEAD)"

      - name: Push clean branch and report conflicts — conflict
        if: steps.rebase.outputs.result == 'conflict'
        run: |
          TAG="${{ steps.release.outputs.tag }}"
          # Push at middlendian/custom HEAD — gives you a local starting point
          git push origin "middlendian/$TAG"
          echo ""
          echo "❌ Rebase conflict. Branch middlendian/$TAG pushed at middlendian/custom HEAD."
          echo ""
          echo "Conflicting files:"
          echo "  ${{ steps.rebase.outputs.conflicting_files }}"
          echo ""
          echo "Stopped at commit: ${{ steps.rebase.outputs.stopped_sha }}"
          echo "Commit message:    ${{ steps.rebase.outputs.stopped_msg }}"
          echo ""
          echo "To reproduce and resolve locally:"
          echo "  git fetch origin"
          echo "  git remote add upstream https://github.com/BerriAI/litellm.git  # skip if already set"
          echo "  git fetch upstream --tags"
          echo "  git checkout middlendian/$TAG"
          echo "  git rebase refs/tags/$TAG"
          echo "  # for each conflict: resolve, then 'git rebase --continue'"
          echo "  # when fully resolved: git push --force origin middlendian/$TAG"
          exit 1
```

- [ ] **Step 2: Validate the YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/middlendian-rebase.yml')); print('YAML OK')"
```

Expected output: `YAML OK`

If `actionlint` is available (`brew install actionlint` / download binary), also run:
```bash
actionlint .github/workflows/middlendian-rebase.yml
```
Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/middlendian-rebase.yml
git commit -m "feat: add middlendian-rebase workflow"
```

- [ ] **Step 4: Push and verify on GitHub**

```bash
git push origin middlendian/custom
```

Navigate to the repo on GitHub → **Actions** tab → **"Middlendian — Rebase onto upstream release"** → **Run workflow** → Run.

**Verify green path:**
- Workflow completes successfully (green checkmark)
- A new branch `middlendian/v{latest-tag}` appears under **Code → Branches**
- The branch HEAD is ahead of `middlendian/custom` by the rebase commits

**Verify idempotency guard:**
- Run the workflow a second time without deleting the branch
- Workflow fails with: `Branch middlendian/v{tag} already exists on origin`

---

## Chunk 2: Release Workflow

### Task 2: `middlendian-release.yml`

**Files:**
- Create: `.github/workflows/middlendian-release.yml`

**Pre-requisite:** Confirm that `middlendian/custom` does NOT have branch protection rules that would block a force-push from `GITHUB_TOKEN`. If it does, add an exception for GitHub Actions in the branch protection settings before testing this workflow.

- [ ] **Step 1: Create the workflow file**

Create `.github/workflows/middlendian-release.yml` with the following content:

```yaml
name: Middlendian — Release

on:
  workflow_dispatch:
    inputs:
      branch:
        description: "Staging branch to release (e.g. middlendian/v1.82.6)"
        required: true
        type: string

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout repo (full history)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure git identity
        run: |
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git config user.name "github-actions[bot]"

      - name: Parse version from branch name
        id: version
        run: |
          BRANCH="${{ github.event.inputs.branch }}"
          # Expect format: middlendian/v1.2.3
          VERSION="${BRANCH#middlendian/v}"
          if [ "$VERSION" = "$BRANCH" ]; then
            echo "::error::Branch name must match pattern 'middlendian/v*' (got: $BRANCH)"
            exit 1
          fi
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "release_tag=v${VERSION}-middlendian" >> "$GITHUB_OUTPUT"
          echo "Branch: $BRANCH → Version: $VERSION → Release tag: v${VERSION}-middlendian"

      - name: Check release tag does not already exist
        run: |
          RELEASE_TAG="${{ steps.version.outputs.release_tag }}"
          if git ls-remote --exit-code origin "refs/tags/$RELEASE_TAG" > /dev/null 2>&1; then
            echo "::error::Tag $RELEASE_TAG already exists on origin. Cannot re-release."
            exit 1
          fi
          echo "Tag $RELEASE_TAG does not exist — proceeding."

      - name: Fetch upstream remote, tags, and staging branch
        run: |
          git remote add upstream https://github.com/BerriAI/litellm.git
          git fetch upstream --tags
          git fetch origin "${{ github.event.inputs.branch }}"

      - name: Verify upstream tag is ancestor of staging branch
        id: verify
        run: |
          VERSION="${{ steps.version.outputs.version }}"
          BRANCH="${{ github.event.inputs.branch }}"

          # Dereference annotated tag to commit SHA
          UPSTREAM_TAG_SHA=$(git rev-parse "refs/tags/v${VERSION}^{commit}" 2>/dev/null || true)
          if [ -z "$UPSTREAM_TAG_SHA" ]; then
            echo "::error::Upstream tag v${VERSION} not found after fetching upstream"
            exit 1
          fi

          BRANCH_HEAD=$(git rev-parse "origin/${BRANCH}")

          if ! git merge-base --is-ancestor "$UPSTREAM_TAG_SHA" "$BRANCH_HEAD"; then
            echo "::error::Security check failed: upstream tag v${VERSION} ($UPSTREAM_TAG_SHA) is NOT an ancestor of $BRANCH ($BRANCH_HEAD)"
            echo "::error::The branch may not have been rebased onto this tag, or the upstream tag was mutated."
            exit 1
          fi

          echo "branch_head=$BRANCH_HEAD" >> "$GITHUB_OUTPUT"
          echo "upstream_tag_sha=$UPSTREAM_TAG_SHA" >> "$GITHUB_OUTPUT"
          echo "✅ Verified: upstream tag v${VERSION} ($UPSTREAM_TAG_SHA) is an ancestor of $BRANCH ($BRANCH_HEAD)"

      - name: Force-push middlendian/custom to staging branch HEAD
        run: |
          BRANCH_HEAD="${{ steps.verify.outputs.branch_head }}"
          git push --force origin "${BRANCH_HEAD}:refs/heads/middlendian/custom"
          echo "✅ middlendian/custom updated to $BRANCH_HEAD"

      - name: Create and push release tag
        run: |
          RELEASE_TAG="${{ steps.version.outputs.release_tag }}"
          BRANCH_HEAD="${{ steps.verify.outputs.branch_head }}"
          git tag "$RELEASE_TAG" "$BRANCH_HEAD"
          git push origin "$RELEASE_TAG"
          echo "✅ Tag $RELEASE_TAG created and pushed → Docker publish workflow triggered"
          echo ""
          echo "Release summary:"
          echo "  Branch:        ${{ github.event.inputs.branch }}"
          echo "  Branch HEAD:   $BRANCH_HEAD"
          echo "  Upstream tag:  ${{ steps.verify.outputs.upstream_tag_sha }}"
          echo "  Release tag:   $RELEASE_TAG"
```

- [ ] **Step 2: Validate the YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/middlendian-release.yml')); print('YAML OK')"
```

Expected: `YAML OK`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/middlendian-release.yml
git commit -m "feat: add middlendian-release workflow"
```

- [ ] **Step 4: Push and verify on GitHub**

```bash
git push origin middlendian/custom
```

Navigate to **Actions** → **"Middlendian — Release"** → **Run workflow**.

**To test without triggering a real Docker publish:**
- Pick a staging branch created by the rebase workflow (e.g. `middlendian/v1.82.6`) that you are NOT yet ready to release
- Run the workflow with that branch as input
- Confirm all verification steps pass in the logs
- **Cancel** the workflow before the "Create and push release tag" step if you don't want to actually publish

**To do a full end-to-end test:**
- Ensure the staging branch (`middlendian/v{tag}`) exists and is properly rebased
- Run the release workflow with that branch
- Verify:
  - `middlendian/custom` branch now points to the staging branch HEAD
  - Tag `v{version}-middlendian` exists on origin
  - `middlendian_docker_publish.yml` workflow triggered automatically

**Verify idempotency guard:**
- Run the release workflow again with the same branch
- Workflow fails with: `Tag v{version}-middlendian already exists on origin`
