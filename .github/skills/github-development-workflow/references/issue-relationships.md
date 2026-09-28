# Native issue relationships

Use this reference when authorized work needs hierarchy or ordering. Prefer
supported native CLI flags; probe `gh issue create --help` and
`gh issue edit --help`. These API recipes support older CLI installations only
when the target host supports the endpoints. Follow the [core skill](../SKILL.md)
scope rules and [API conventions](github-features.md#api-request-correctness).

## Resolve identities first

Set `HOST` explicitly. Repository variables below are `OWNER/NAME`, without a
host prefix. Resolve each issue in its own repository. An issue number, REST
database `id`, and GraphQL `node_id` are different identifiers.

Inspect the issue response, including its URL and whether it has a
`pull_request` property; do not accidentally treat a PR as a work-item issue.
Validate numbers and returned IDs before constructing a mutation.

## Attach a child to a parent

Resolve `PARENT_REPO`, `PARENT_NUMBER`, `CHILD_REPO`, and `CHILD_NUMBER`.
Inspect the child's current parent and the parent's children first:

```bash
gh api --hostname "$HOST" --method GET \
  "repos/$CHILD_REPO/issues/$CHILD_NUMBER/parent"
gh api --hostname "$HOST" --method GET --paginate \
  "repos/$PARENT_REPO/issues/$PARENT_NUMBER/sub_issues" -F per_page=100
```

A parent lookup returning 404 requires checking issue visibility and endpoint
support before treating the child as unparented. Reuse an existing relationship.
Do not silently reparent a child or create a hierarchy cycle.

After resolving those checks, obtain the child's REST ID and attach it:

```bash
CHILD_ID=$(gh api --hostname "$HOST" --method GET \
  "repos/$CHILD_REPO/issues/$CHILD_NUMBER" --jq .id)
gh api --hostname "$HOST" --method POST \
  "repos/$PARENT_REPO/issues/$PARENT_NUMBER/sub_issues" \
  -F sub_issue_id="$CHILD_ID"
```

Read the parent and children again to verify identity. The documented endpoint
requires the same repository owner for parent and child. Do not use
`replace_parent=true` without authority to move existing work.
See [GitHub's sub-issue API](https://docs.github.com/en/rest/issues/sub-issues).

## Make A depend on B

`DEPENDENT_REPO` / `DEPENDENT_NUMBER` identify A, which must wait.
`BLOCKER_REPO` / `BLOCKER_NUMBER` identify B, which must finish first.
Inspect existing edges, including B's dependencies, to avoid cycles:

```bash
gh api --hostname "$HOST" --method GET --paginate \
  "repos/$DEPENDENT_REPO/issues/$DEPENDENT_NUMBER/dependencies/blocked_by" \
  -F per_page=100
gh api --hostname "$HOST" --method GET --paginate \
  "repos/$BLOCKER_REPO/issues/$BLOCKER_NUMBER/dependencies/blocked_by" \
  -F per_page=100
```

Follow relevant dependency paths when needed; inspecting immediate neighbors
alone cannot exclude an indirect cycle. If the intended edge already exists,
do not post it again.

```bash
BLOCKER_ID=$(gh api --hostname "$HOST" --method GET \
  "repos/$BLOCKER_REPO/issues/$BLOCKER_NUMBER" --jq .id)
gh api --hostname "$HOST" --method POST \
  "repos/$DEPENDENT_REPO/issues/$DEPENDENT_NUMBER/dependencies/blocked_by" \
  -F issue_id="$BLOCKER_ID"
```

Re-list A's `blocked_by` relationships and verify B's identity. This direction
means **A is blocked by B**, regardless of their parent/child relationship.
See [GitHub's dependency API](https://docs.github.com/en/rest/issues/issue-dependencies).

## Recover without duplicating writes

After a timeout, inspect the native relationship before retrying. For a 403,
404, or 422, diagnose permissions, host support, identities, relationship
constraints, and rate limits. Do not repeatedly create duplicate issues or
silently replace native relationships with prose. Report a concrete limitation
and the remaining manual action when an authorized native write is unavailable.
