# 🚦 Merge Queue Action

A lightweight GitHub Actions merge queue for teams that don't have GitHub Enterprise. Add a label, walk away — your PRs merge themselves.

## The Problem

GitHub requires branches to be up-to-date with the base branch before merging. With multiple PRs, this creates a painful cycle:

1. Update branch → wait for CI → try to merge
2. Someone else merged first → branch is outdated again
3. Repeat until you lose the will to live

GitHub's native merge queue solves this, but it's **Enterprise Cloud only** for private repos.

## The Solution

This action processes labeled PRs one at a time, in FIFO order:

1. Developer adds the `queue` label to an approved PR
2. The action updates the branch if it's behind
3. Waits for CI to pass
4. Merges via squash and moves to the next PR

No Enterprise plan needed. No third-party services. Just a GitHub Action.

## Setup

### 1. Create a GitHub App (recommended)

Using a GitHub App token ensures branch updates trigger CI and don't dismiss approvals.

1. Go to your org's **Settings → Developer settings → GitHub Apps → New GitHub App**
2. Fill in:
   - **Name:** `Merge Queue Bot`
   - **Homepage URL:** your repo URL
   - **Webhook:** uncheck "Active"
   - **Permissions:**
     - `Contents: Read & write`
     - `Pull requests: Read & write`
     - `Checks: Read`
3. Create the app, note the **App ID**
4. Generate a **private key** (downloads a `.pem` file)
5. Install the app on your repository
6. Add repo secrets:
   - `MERGE_QUEUE_APP_ID` — the numeric App ID
   - `MERGE_QUEUE_APP_PRIVATE_KEY` — full contents of the `.pem` file

### 2. Add the workflow

Copy `.github/workflows/merge_queue.yml` into your repository, or create it with the contents below:

```yaml
name: Merge Queue

on:
  schedule:
    - cron: "*/5 * * * *"
  pull_request:
    types: [labeled]
  workflow_dispatch: {}

concurrency:
  group: merge-queue
  cancel-in-progress: false

jobs:
  check-queue:
    if: >
      github.event_name != 'pull_request' ||
      github.event.label.name == 'queue'
    runs-on: ubuntu-latest
    outputs:
      has_queued: ${{ steps.check.outputs.has_queued }}
    steps:
      - name: Check for queued PRs
        id: check
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          REPO: ${{ github.repository }}
        run: |
          COUNT=$(gh pr list --repo "$REPO" --label "queue" --state open --base master --json number --jq 'length')
          echo "has_queued=$( [ "$COUNT" -gt 0 ] && echo true || echo false )" >> "$GITHUB_OUTPUT"

  process-queue:
    needs: check-queue
    if: needs.check-queue.outputs.has_queued == 'true'
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.MERGE_QUEUE_APP_ID }}
          private-key: ${{ secrets.MERGE_QUEUE_APP_PRIVATE_KEY }}

      - name: Process merge queue
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
          REPO: ${{ github.repository }}
        run: |
          set -euo pipefail

          QUEUED_PRS=$(gh pr list --repo "$REPO" --label "queue" --state open --base master \
            --json number,createdAt --jq 'sort_by(.createdAt) | .[].number')

          [ -z "$QUEUED_PRS" ] && exit 0

          PR_NUM=$(echo "$QUEUED_PRS" | head -1)
          echo "Processing PR #$PR_NUM"

          # Must be approved
          APPROVED=$(gh pr view "$PR_NUM" --repo "$REPO" --json reviewDecision --jq '.reviewDecision')
          if [ "$APPROVED" != "APPROVED" ]; then
            echo "Not approved. Skipping."
            exit 0
          fi

          # No merge conflicts
          MERGEABLE=$(gh pr view "$PR_NUM" --repo "$REPO" --json mergeable --jq '.mergeable')
          if [ "$MERGEABLE" = "CONFLICTING" ]; then
            gh pr edit "$PR_NUM" --repo "$REPO" --remove-label "queue"
            gh pr comment "$PR_NUM" --repo "$REPO" \
              --body "❌ **Merge Queue:** Removed — merge conflicts. Resolve and re-add \`queue\` label."
            exit 0
          fi

          # Update branch if behind
          HEAD_REF=$(gh pr view "$PR_NUM" --repo "$REPO" --json headRefName --jq '.headRefName')
          ENCODED_HEAD_REF=$(printf '%s' "$HEAD_REF" | jq -sRr @uri)
          IS_BEHIND=$(gh api "/repos/$REPO/compare/master...$ENCODED_HEAD_REF" --jq '.behind_by')

          if [ "$IS_BEHIND" -gt 0 ]; then
            if ! gh api -X PUT "/repos/$REPO/pulls/$PR_NUM/update-branch" \
              --field expected_head_sha="$(gh pr view "$PR_NUM" --repo "$REPO" --json headRefOid --jq '.headRefOid')" 2>&1; then
              echo "::warning::Branch update failed. Retrying next cycle."
            else
              echo "Branch updated. Waiting for CI."
            fi
            exit 0
          fi

          # Check CI
          CI_STATE=$(gh pr checks "$PR_NUM" --repo "$REPO" --json name,state \
            --jq '[.[] | select(.name | startswith("Merge Queue") | not)] |
                  if length == 0 then "PENDING"
                  elif any(.state == "FAILURE" or .state == "ERROR" or .state == "CANCELLED" or .state == "TIMED_OUT" or .state == "STARTUP_FAILURE" or .state == "ACTION_REQUIRED") then "FAILURE"
                  elif any(.state == "PENDING" or .state == "QUEUED" or .state == "IN_PROGRESS") then "PENDING"
                  elif all(.state == "SUCCESS" or .state == "SKIPPED" or .state == "NEUTRAL") then "SUCCESS"
                  else "UNKNOWN" end')

          case "$CI_STATE" in
            SUCCESS)
              gh pr merge "$PR_NUM" --repo "$REPO" --squash --delete-branch
              echo "✅ PR #$PR_NUM merged!"
              ;;
            PENDING)
              echo "CI running. Will retry."
              ;;
            FAILURE)
              gh pr edit "$PR_NUM" --repo "$REPO" --remove-label "queue"
              gh pr comment "$PR_NUM" --repo "$REPO" \
                --body "❌ **Merge Queue:** Removed — CI failed. Fix and re-add \`queue\` label."
              ;;
            *)
              echo "Unknown CI state. Will retry."
              ;;
          esac
```

### 3. Create the label

Create a `queue` label on your repository (green `#2EA44F` recommended).

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| Base branch | `master` | Change in the workflow if you use `main` |
| Merge method | `squash` | Change `--squash` to `--merge` or `--rebase` |
| Schedule | Every 5 min | Adjust the cron expression |
| Label | `queue` | Change the label name in the workflow |

## How It Works

```
Developer adds "queue" label
         │
         ▼
┌─────────────────┐
│ Any queued PRs?  │──No──▶ Exit (saves runner minutes)
└────────┬────────┘
         │Yes
         ▼
┌─────────────────┐
│   PR approved?   │──No──▶ Skip, retry next cycle
└────────┬────────┘
         │Yes
         ▼
┌─────────────────┐
│ Merge conflicts? │──Yes─▶ Dequeue + comment
└────────┬────────┘
         │No
         ▼
┌─────────────────┐
│ Behind base?     │──Yes─▶ Update branch, wait for CI
└────────┬────────┘
         │No
         ▼
┌─────────────────┐
│   CI passing?    │──No──▶ Dequeue + comment (failure)
└────────┬────────┘        or retry (pending)
         │Yes
         ▼
    ✅ Merge!
```

## Why Not GitHub's Native Merge Queue?

GitHub's built-in merge queue requires **Enterprise Cloud** for private repos (~$21/user/month). This action works on any plan, including Free and Team.

## Why Not Mergify/Kodiak/etc?

Those are great tools, but they're paid services for private repos. This is free, runs on your own GitHub Actions minutes, and you own the code.

## License

MIT
