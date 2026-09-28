# Pull-request delivery

Read this reference for publication, CI diagnosis, review, or authorized merging.
Apply the [core skill](../SKILL.md) scope and safety rules throughout. Enter at
the section needed for the task; stop at the requested deliverable.

## 1. Commit, publish, and create a draft PR

Commit when requested. Proceed to remote writes only when publication is
authorized; a local-commit request stops before pushing.

Review the complete diff and ensure it contains no unrelated work, credentials,
private vulnerability details, or accidental artifacts.

Stage explicit intended paths. Inspect the staged diff:

```bash
git diff --cached --check
git diff --cached
```

Commit according to repository conventions. Do not impose Conventional Commits,
invent signing identities, or bypass required hooks/signatures.

Push to the verified head repository:

```bash
git push -u "$PUSH_REMOTE" "$BRANCH"
```

If denied, distinguish permissions, token scopes, rules, and repository policy.
Do not bypass restrictions or silently choose a different publication target.
Use an authorized fork only when appropriate.

After pushing, verify the remote branch points to the intended local commit.
If the request ends at a pushed branch, report that result and stop.
Do not create a PR merely to finish this reference.

Before PR creation, check again for an existing PR with the same head repository,
branch, and intended base. Update the existing PR rather than duplicating it.

Use the repository's PR template. Include:

- Purpose and scope.
- Changes made.
- Verification evidence.
- Known failures or limitations.
- Screenshots, migration, rollout, or compatibility notes when relevant.

Use closing keywords in the PR body only when the PR actually completes the
issue and GitHub's closing semantics apply, normally upon merge into the
default branch.

Use a qualified reference when helpful:

```text
Closes OWNER/REPO#42
```

For partial work, non-default targets, or contextual issues, use references
instead. Do not close a parent issue with unfinished acceptance criteria.

Create a draft with explicit repository, base, and head:

```bash
gh pr create --repo "$REPO" \
  --draft \
  --base "$BASE_BRANCH" \
  --head "$PR_HEAD" \
  --title "$TITLE" \
  --body-file "$PR_BODY_FILE"
```

`PR_HEAD` is the branch or supported fork-qualified head identifier.
Use the host/CLI's documented syntax for cross-repository PRs.

Record the PR URL, number, and head SHA.

A draft may preserve incomplete work and failure evidence when publication is
safe and authorized. A draft is not an exception to secret-handling or private
security-reporting requirements.

## 2. Observe CI, security, and review

Inspect the current PR state:

```bash
gh pr view "$PR_NUMBER" --repo "$REPO" --json \
  url,state,isDraft,baseRefName,headRefOid,mergeable,mergeStateStatus,\
  reviewDecision,statusCheckRollup

gh pr checks "$PR_NUMBER" --repo "$REPO"
```

Watch required checks only with a bounded session/tool timeout:

```bash
gh pr checks "$PR_NUMBER" --repo "$REPO" --required --watch
```

### Check-state decisions

Capture both the exit code and check details. The
[CLI check command](https://cli.github.com/manual/gh_pr_checks) uses exit code
`8` for pending checks. A nonzero exit alone does not identify a failing test.

| Observation | Next action |
| --- | --- |
| Pending or queued | Inspect the current run; wait within a bounded budget, then hand off if still pending. |
| Failed, timed out, or canceled | Inspect logs and event context; distinguish a regression, superseded run, infrastructure problem, or deliberate cancellation. |
| No checks reported | Inspect workflow triggers, draft/fork approval, required checks, and external integrations before deciding why. |
| CI genuinely not configured or applicable | Record that limitation, perform relevant local verification, and allow reviewable work to become ready. |
| Skipped or neutral | Report the actual conclusion; do not claim the test executed, even if GitHub accepts it for merge policy. |
| Permission error or unknown mergeability | Keep state unknown, investigate or re-query with bounded backoff; never infer success. |

Interpret results carefully:

- No reported checks is not automatically success.
- Pending, missing, skipped, approval-blocked, and failed checks differ.
- A required-check list is not the entirety of repository policy.
- Inspect relevant non-required failures and security findings too.
- Associate evidence with the current head or its corresponding GitHub
  test-merge/merge-group commit, not an obsolete run.
- An inaccessible policy or check remains unknown.
- GitHub mergeability can be temporarily unknown; re-query with bounded
  backoff instead of guessing.

Inspect expected check names and their configured GitHub App/source; a similarly
named check from another producer is not interchangeable. A failed external
check may have no Actions run. Follow its details link through an authorized
integration instead of repeatedly searching Actions.

For failures, inspect relevant Actions or external-check logs, diagnose,
fix, verify locally, and push. Invalidate old verification when the head changes.

### Reviewing an existing PR

For a review-only request, inspect the base, head SHA, complete diff, relevant
tests, and repository instructions. Return actionable findings with file/line
references and evidence. State when no actionable findings were found, plus
verification limits. Do not modify the contributor's branch or publish a
review/comment unless that external action is authorized.

Before posting an authorized review, re-read the head. If it changed, reassess
affected findings before attaching feedback to the new commit. Do not submit
an approval based on an obsolete diff.

### Draft-to-ready transition

- Honor a request to keep the PR draft.
- Normally keep incomplete work or unresolved implementation failures draft.
- Some workflows intentionally skip drafts or start on `ready_for_review`.
- If the implementation is reviewable and local requirements are satisfied,
  mark ready when necessary to trigger those workflows or obtain required
  human actions. Clearly report that remote verification is still pending.
- Never change workflow security or expose secrets just to run draft/fork CI.

Mark ready when appropriate:

```bash
gh pr ready "$PR_NUMBER" --repo "$REPO"
```

Allow normal CODEOWNERS routing. Request additional reviewers only when useful
or required. Optional automated review does not replace required human review.

Do not approve the agent's own work through another identity or otherwise
manufacture approval.

Address review feedback, verify again, and preserve collaborators' changes.
Resolve conversations only after the concern is addressed and repository
conventions permit doing so.

If GitHub requires a branch update, use the repository-approved merge/rebase
method, optionally `gh pr update-branch` when suitable. Check for concurrent
updates. Every changed head requires renewed verification.

Record `VALIDATED_HEAD` as the exact head whose diff was reviewed and whose
local verification requirements were satisfied. Record remote check and
review state separately.

If external approval, credentials, CI, or deployment is pending, return a
bounded handoff with the responsible next action instead of waiting forever.

## 3. Merge only within authorization

Before an immediate merge, auto-merge request, or queue enrollment:

1. Confirm that the operation is authorized.
2. Re-read PR state, target branch, policy, and head SHA.
3. Compare the current head with `VALIDATED_HEAD`.
4. If different, inspect and validate the new head; do not simply overwrite
   the recorded SHA with the latest value.
5. Confirm the PR is not draft and no known unresolved implementation or
   security problem makes the requested operation inappropriate.

Immediate merging requires GitHub's applicable requirements to be satisfied.
Authorized auto-merge or queue enrollment may wait for GitHub-managed gates
when repository policy permits.

For repositories without a required merge queue, choose the method from:

- Repository instructions and target-branch policy.
- Allowed merge methods.
- The viewer's preference, only if compatible with those constraints.

Do not hard-code squash merging. If no unambiguous allowed method can be
selected, ask rather than changing repository settings.

Set `MERGE_FLAG` to exactly one supported, permitted option:
`--merge`, `--squash`, or `--rebase`.

For an authorized immediate merge:

```bash
gh pr merge "$PR_NUMBER" --repo "$REPO" \
  "$MERGE_FLAG" \
  --match-head-commit "$VALIDATED_HEAD"
```

For authorized auto-merge, if available:

```bash
gh pr merge "$PR_NUMBER" --repo "$REPO" \
  --auto \
  "$MERGE_FLAG" \
  --match-head-commit "$VALIDATED_HEAD"
```

For a required merge queue, let GitHub manage its strategy:

```bash
gh pr merge "$PR_NUMBER" --repo "$REPO" \
  --match-head-commit "$VALIDATED_HEAD"
```

Inspect the result: queue-required behavior may enqueue the PR or enable
automatic progression while requirements are pending.

Never use `--admin` to bypass reviews, checks, deployments, or the queue.

`--match-head-commit` protects the request against a changed head. It is not
a permanent lock on future auto-merge or merge-queue execution. Later pushes
require renewed assessment; GitHub rules govern future execution.

Do not combine merging with local branch deletion. Verify the actual result:

```bash
gh pr view "$PR_NUMBER" --repo "$REPO" --json \
  state,mergedAt,mergeCommit,headRefOid,url
```

A successful CLI invocation does not necessarily mean the PR merged.
Distinguish merged, auto-merge-enabled, queued, waiting, and failed states.

## 4. Clean up only verified, agent-owned state

Do not remove the branch or worktree merely because auto-merge is enabled
or the PR is queued.

After confirming the PR is actually merged:

- Confirm the local branch has no additional unpublished commits.
- Check tracked, untracked, and ignored files for work or unique artifacts
  that must be preserved.
- Remove only the agent-owned worktree, from outside that worktree.
- Do not use forced removal.
- Delete a local topic branch only when safe. If normal deletion refuses
  after squash/rebase merging, retain it rather than escalating automatically.
- Allow configured GitHub branch deletion to operate; separately authorize
  any additional remote cleanup.
- Never switch, pull, reset, or broadly prune the user's original workspace.
- Confirm expected issue closure and authorized Project transitions.
- Continue with another sub-issue only if it is unblocked and within scope.

If the PR is closed without merging, preserve work unless disposal is
separately authorized.
