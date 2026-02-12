# 🚦 Merge Queue Action

A lightweight GitHub Actions merge queue for teams that don't have GitHub Enterprise. Add a label, walk away — your PRs merge themselves.

## The Problem

GitHub requires branches to be up-to-date with the base branch before merging. With multiple PRs, this creates a painful cycle:

1. Update branch → wait for CI → try to merge
2. Someone else merged first → branch is outdated again
3. Repeat until you lose the will to live

GitHub's native merge queue solves this, but it's **Enterprise Cloud only** for private repos.

## Quick Start

Add this workflow to your repo at `.github/workflows/merge_queue.yml`:

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
  merge-queue:
    if: >
      github.event_name != 'pull_request' ||
      github.event.label.name == 'queue'
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.MERGE_QUEUE_APP_ID }}
          private-key: ${{ secrets.MERGE_QUEUE_APP_PRIVATE_KEY }}

      - name: Run merge queue
        uses: linuxlewis/merge-queue-action@v1
        with:
          github-token: ${{ steps.app-token.outputs.token }}
```

That's it. Create a `queue` label on your repo, and start labeling approved PRs.

## Setup

### 1. Create a GitHub App

Using a GitHub App token ensures branch updates trigger CI and don't dismiss approvals.

1. Go to your org's **Settings → Developer settings → GitHub Apps → New GitHub App**
2. Fill in:
   - **Name:** `Merge Queue Bot`
   - **Homepage URL:** your repo URL
   - **Webhook:** uncheck "Active"
   - **Permissions:**
     - `Actions: Read`
     - `Contents: Read & write`
     - `Pull requests: Read & write`
     - `Checks: Read`
3. Create the app, note the **App ID**
4. Generate a **private key** (downloads a `.pem` file)
5. Install the app on your repository
6. Add repo secrets:
   - `MERGE_QUEUE_APP_ID` — the numeric App ID
   - `MERGE_QUEUE_APP_PRIVATE_KEY` — full contents of the `.pem` file

### 2. Create the label

Create a `queue` label on your repository (green `#2EA44F` recommended).

## Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `github-token` | *required* | GitHub token (use a GitHub App token) |
| `base-branch` | `master` | Base branch to merge into |
| `label` | `queue` | Label that marks PRs for the queue |
| `merge-method` | `squash` | Merge method: `squash`, `merge`, or `rebase` |

### Example with custom options

```yaml
      - name: Run merge queue
        uses: linuxlewis/merge-queue-action@v1
        with:
          github-token: ${{ steps.app-token.outputs.token }}
          base-branch: main
          label: ready-to-merge
          merge-method: rebase
```

## How It Works

```
Developer adds "queue" label
         │
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

PRs are processed one at a time, oldest first (FIFO). This avoids the exact problem of multiple PRs racing to merge and invalidating each other.

## Why Not GitHub's Native Merge Queue?

GitHub's built-in merge queue requires **Enterprise Cloud** for private repos (~$21/user/month). This action works on any plan, including Free and Team.

## Why Not Mergify/Kodiak/etc?

Those are great tools, but they're paid services for private repos. This is free, runs on your own GitHub Actions minutes, and you own the code.

## License

MIT
