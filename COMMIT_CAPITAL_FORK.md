# Commit Capital Paperclip Fork

This is Commit Capital's internal fork of [paperclipai/paperclip](https://github.com/paperclipai/paperclip).

We maintain this fork to cherry-pick community PRs that haven't been merged upstream yet.

## Branch model

- **`master`** — tracks `paperclipai/paperclip` master (via the `upstream` remote)
- **`commit-capital`** — our deployed branch, based on a stable upstream tag with community PRs cherry-picked on top

## Currently applied community PRs

Base: `v2026.512.0` (rebased from `v2026.428.0` on 2026-05-13)

| Upstream PR | Title | Reason |
|---|---|---|
| [#4836](https://github.com/paperclipai/paperclip/pull/4836) | Report agent_home truthfully when project workspaces are unavailable (PF-3) | Root cause of "No conversation found with session ID" errors — fallback workspace mislabeled as `project_primary`, breaking session resume |
| [#5295](https://github.com/paperclipai/paperclip/pull/5295) | Fix timer wake task-session workspace fallback | Forces fresh sessions on timer wakes; prevents stale session_display_id from persisting across runs |
| [#3781](https://github.com/paperclipai/paperclip/pull/3781) | Wake requester on reject and revision | Approval `revision_requested` and `rejected` states now wake the agent so it can respond — closes [#3780](https://github.com/paperclipai/paperclip/issues/3780) |
| [#3448](https://github.com/paperclipai/paperclip/pull/3448) | Queue follow-up run on approval_approved when mid-run | Ensures the wake from #3781 isn't swallowed by run coalescing when the agent is already running |
| [#5466](https://github.com/paperclipai/paperclip/pull/5466) | Fix workspace resolver to use projectWorkspaceId when projectId is null (BRA-500) | Root cause of "fallback workspace" warning — resolver was returning empty when projectId was null even though projectWorkspaceId was populated. Adds query path that resolves by workspace ID. |
| [#5749](https://github.com/paperclipai/paperclip/pull/5749) | Prevent invite page crash caused by React Query cache collision | Invite page crashed with `Te.some is not a function` because CompanyProvider stored a wrapper object under the same cache key InviteLanding expects to be an array. Affected new team members trying to accept invites. |

## How this gets deployed

The Commit Capital VM (`paperclip` in `commit-paperclip` GCP project) clones this fork to `/home/paperclip/paperclip-src` and runs from source (via `tsx`) on every deploy.

See `commit-capital/commit-paperclip` repo for the deploy scripts.

## Workflow: tracking upstream releases

When `paperclipai/paperclip` ships a new release we want to adopt:

```bash
# In a local clone:
git fetch upstream --tags
git checkout commit-capital

# Easier approach: squash + replay onto new tag
# (Replaying individual commits often conflicts heavily against fast-moving upstream)
git branch commit-capital-pre-rebase   # safety net

# Generate a single combined patch of our changes vs the old base
git diff <OLD_TAG> commit-capital -- . ':!COMMIT_CAPITAL_FORK.md' > /tmp/our-patches.diff

# Reset to new tag, apply patch with 3-way merge
git reset --hard <NEW_TAG>
git apply --3way /tmp/our-patches.diff

# Resolve any conflicts (likely in heartbeat.ts), then commit
git add -A
git commit -m "Replay commit-capital patches onto <NEW_TAG>"
git push --force-with-lease origin commit-capital
```

Then on the VM: `bash deploy/remote.sh` (which does `git pull && pnpm install`).

## Workflow: adding a new community PR

```bash
git fetch upstream pull/<PR_NUMBER>/head:pr-<PR_NUMBER>
git cherry-pick pr-<PR_NUMBER>
git push origin commit-capital
```

If the PR doesn't apply cleanly, try `git format-patch` + `git apply --3way` instead.

Update the table above. On the VM, redeploy.

## When upstream merges one of our PRs

When a community PR we cherry-picked finally lands in upstream:
1. Note it in the table above
2. Next rebase onto upstream will naturally include it
3. The corresponding cherry-pick on our branch will dedup or auto-skip

## Don't merge to `master`

Our changes live only on the `commit-capital` branch. `master` stays clean for syncing from upstream.
