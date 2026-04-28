# Target Repository Dispatcher

A ClawSweeper report repository cannot receive native `issues` or
`pull_request` events from sibling repositories directly. Target repositories
should forward those events with `repository_dispatch` so ClawSweeper can run a
single-job exact one-item review, sync the durable review comment, and
immediately apply a safe close proposal for that same item.

Set these placeholders before installing the workflow:

- `CLAWSWEEPER_OWNER`: owner of your fork, for example `virtualite9-ctrl`
- `CLAWSWEEPER_REPO`: report repository name, usually `clawsweeper`

Copy this workflow into each target repository as
`.github/workflows/clawsweeper-dispatch.yml`:

```yaml
name: ClawSweeper Dispatch

on:
  issues:
    types: [opened, reopened, edited, labeled, unlabeled]
  pull_request_target:
    types: [opened, reopened, synchronize, ready_for_review, edited, labeled, unlabeled]

permissions:
  contents: read

jobs:
  dispatch:
    runs-on: ubuntu-latest
    env:
      HAS_CLAWSWEEPER_APP_PRIVATE_KEY: ${{ secrets.CLAWSWEEPER_APP_PRIVATE_KEY != '' }}
    steps:
      - name: Create ClawSweeper dispatch token
        id: token
        if: ${{ env.HAS_CLAWSWEEPER_APP_PRIVATE_KEY == 'true' }}
        uses: actions/create-github-app-token@v2
        with:
          app-id: 3306130
          private-key: ${{ secrets.CLAWSWEEPER_APP_PRIVATE_KEY }}
          owner: ${{ vars.CLAWSWEEPER_OWNER || github.repository_owner }}
          repositories: ${{ vars.CLAWSWEEPER_REPO || 'clawsweeper' }}

      - name: Dispatch exact ClawSweeper review
        env:
          GH_TOKEN: ${{ steps.token.outputs.token || secrets.OPENCLAW_GH_TOKEN }}
          TARGET_REPO: ${{ github.repository }}
          ITEM_NUMBER: ${{ github.event.issue.number || github.event.pull_request.number }}
          ITEM_KIND: ${{ github.event_name == 'pull_request_target' && 'pull_request' || 'issue' }}
        run: |
          if [ -z "$GH_TOKEN" ]; then
            echo "::notice::Skipping ClawSweeper dispatch because no dispatch credential is configured."
            exit 0
          fi
          payload="$(jq -nc \
            --arg target_repo "$TARGET_REPO" \
            --argjson item_number "$ITEM_NUMBER" \
            --arg item_kind "$ITEM_KIND" \
            '{event_type:"clawsweeper_item",client_payload:{target_repo:$target_repo,item_number:$item_number,item_kind:$item_kind}}')"
          gh api repos/${{ vars.CLAWSWEEPER_OWNER || github.repository_owner }}/${{ vars.CLAWSWEEPER_REPO || 'clawsweeper' }}/dispatches \
            --method POST \
            --input - <<< "$payload"
```

The receiver keeps the review lane proposal-only, then runs exact apply for the
selected item with only immediate-safe close reasons enabled:
`implemented_on_main` and `duplicate_or_superseded`. Normal scheduled apply
still handles the broader backlog, with `stale_insufficient_info` blocked until
the item is at least 30 days old.

If you fork ClawSweeper, also set `CLAWSWEEPER_REPORT_REPO` in the report repo's
Actions environment so generated links point at your fork instead of the
upstream `openclaw/clawsweeper`.

The event job creates only a target read token before Codex runs. The target
write token and the repository push token are introduced after Codex exits, and
the same `apply-decisions` guard path still re-fetches the item before any
comment or close mutation.
