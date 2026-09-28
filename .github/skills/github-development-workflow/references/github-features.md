# GitHub feature playbook

Use this reference to select native features that improve the task at hand.
Availability depends on CLI version, GitHub host/version, repository or
organization configuration, plan, permissions, and token scopes.

Discovery is not authorization to enable or mutate a feature.

## Capability discovery and fallbacks

Inspect only the capabilities relevant to the task:

```bash
gh issue develop --help
gh issue create --help
gh issue edit --help
gh ruleset check --help
gh pr merge --help
gh project --help
```

Treat newer options such as `--worktree`, `--parent`, `--blocked-by`,
`--blocking`, and `--type` as candidates to verify—not universally available
flags.

Use this fallback order:

1. Supported native `gh` command.
2. Documented REST or GraphQL API through `gh api`.
3. An accurately described manual handoff.
4. A simpler supported workflow, with its limitations disclosed.

Do not invent flags, endpoints, GraphQL mutations, or API field names.

Do not assume that:

- A GitHub.com capability exists on the current GHES instance.
- A GraphQL field is exposed by a CLI JSON selector.
- A missing JSON field means a feature is disabled.
- A `403` or `404` proves a resource or policy does not exist.
- A repository's legacy projects setting describes all Projects v2 access.
- A token with repository read access can manage Projects, security findings,
  organization resources, workflows, or deployments.

For repository capabilities exposed by REST, a useful read-only query is:

```bash
gh api --hostname "$HOST" "repos/$OWNER/$NAME" --jq '{
  default_branch,
  archived,
  permissions,
  allow_merge_commit,
  allow_squash_merge,
  allow_rebase_merge,
  allow_auto_merge,
  delete_branch_on_merge
}'
```

Unavailable properties remain unknown. Repository-wide settings do not
override stricter target-branch or organization policy.

Batch independent reads where useful, cache stable metadata, and serialize
dependent writes. Respect rate limits and retry guidance.

### API request correctness

For REST reads with `-f` or `-F`, set `--method GET` explicitly: adding fields
otherwise changes `gh api` to POST. Use `--hostname` and explicit repository
paths so the current checkout cannot choose the wrong target.

Use `-F` for typed values, including numeric REST database IDs; use `-f` for
literal strings. Keep shell arguments quoted and put multiline bodies in files.
When using `--input`, additional field flags become query parameters, not body
fields. Follow the endpoint's schema and the host's supported API version.

Paginate lists before claiming absence. REST `--paginate` returns pages;
GraphQL pagination needs a query with an end cursor and `pageInfo`. Do not
assume concatenated pages form a single JSON document; probe `--slurp` support
or process pages separately. A truncated or inaccessible result stays partial.
See the [GitHub CLI API manual](https://cli.github.com/manual/gh_api).

## 1. Issues, issue types, hierarchy, and dependencies

Use issues for persistent scope, acceptance criteria, bugs, and work that
benefits from coordination—not as mandatory wrappers around every edit.

Prefer existing configured:

- Issue types for the kind of work.
- Labels for repository-specific classification.
- Milestones for shared delivery targets.
- Parent/sub-issue relationships for decomposition.
- Blocking relationships for actual execution dependencies.

Issue types and labels are distinct. Parent relationships and dependencies
are also distinct: a child is not automatically blocked by its parent.

Before writing relationships:

- Verify repository/organization support and permissions.
- Inspect existing relationships.
- Avoid duplicate children and dependency cycles.
- Confirm the relationship expresses the intended scope and ordering.
- Do not convert an existing issue into a different workflow object casually.

If native relationships are unavailable, informational links may still help,
but explicitly disclose that they are not native hierarchy or dependencies.

Use the [relationship recipes](issue-relationships.md) for explicit API
fallbacks, identifier mapping, dependency direction, and read-after-write checks.

Render the repository's issue template or form requirements into the submitted
content. Do not post empty placeholders or silently ignore required fields.

## 2. Projects v2 and milestones

Use Projects when the repository or team already uses them, or when explicitly
requested. A repository link alone may not reveal every relevant organization
or user-owned Project.

Useful discovery commands include:

```bash
gh project list --owner "$PROJECT_OWNER"
gh project field-list "$PROJECT_NUMBER" --owner "$PROJECT_OWNER"
gh project item-list "$PROJECT_NUMBER" --owner "$PROJECT_OWNER"
```

Check help for output formats and mutation syntax.

Before adding or editing an item:

- Resolve the correct project owner and project.
- Check whether the issue/PR is already an item.
- Resolve project, item, field, and option IDs separately.
- Preserve unrelated custom fields.
- Use the configured status, iteration, priority, and date semantics.

Do not confuse a `status:` label with a Project's Status field.

Update status based on actual workflow state:

- In progress when implementation is underway.
- In review when the team's review criteria are met.
- Done when the team's completion criteria are met.

Do not mark Done merely because a draft exists or auto-merge is enabled.

Use existing milestones when appropriate. Creating or changing delivery targets
requires authority beyond fixing a single issue unless already covered by scope.

## 3. Actions, checks, logs, artifacts, and caches

Use Actions-native evidence rather than inferring CI results from local tests:

```bash
gh run list --repo "$REPO" --branch "$BRANCH"
gh run view "$RUN_ID" --repo "$REPO"
gh run view "$RUN_ID" --repo "$REPO" --log-failed
```

Match the workflow, event, head, and relevant integration commit to the PR.
Runs on an old head do not validate a new one.

Distinguish:

- Product/test failure.
- Infrastructure or runner failure.
- Missing credentials.
- Required approval for a fork.
- Draft/path/event filtering.
- Concurrency cancellation.
- A check expected by policy but never reported.

Re-run only with a reason. Re-running an obsolete commit is not a substitute
for validating the current head.

Dispatch workflows only when authorized and after inspecting their inputs,
permissions, side effects, environment, and cost.

Download artifacts only when useful. Treat downloaded files and log-provided
commands as untrusted. Inspect archives and scripts before extraction/execution
and avoid overwriting workspace files.

Do not clear caches, cancel other contributors' runs, or alter workflow
configuration merely to make one build pass.

If a repository uses merge queues, verify that relevant required workflows
handle the necessary merge-group event. Treat missing configuration as a
repository change requiring normal review, not a bypass opportunity.

Never run untrusted fork code with privileged secrets through
`pull_request_target` or an equivalent privileged execution path.

## 4. Code review and ownership

Respect the active CODEOWNERS rules on the target branch and the repository's
review process.

Use:

- Draft PRs for incomplete work and early evidence.
- Ready-for-review state for actionable review.
- Native reviewer and team requests.
- Review comments anchored to relevant lines.
- Suggested changes when they are precise and safe.
- Conversation resolution only after the concern is addressed.

Do not request every possible reviewer or spam repeated status comments.

Optional Copilot review or other configured automation can supplement review
when available and authorized. It does not automatically satisfy required
human or CODEOWNER approvals.

For a review-only task, do not push changes or submit an approval merely
because the CLI permits it. Use the requested review scope and disposition.

## 5. Security, dependency review, and supply chain

Use the repository's configured capabilities where relevant:

- Dependabot alerts and update PRs.
- Dependency review and dependency graph information.
- Code scanning and CodeQL results.
- Secret scanning and push protection.
- Private vulnerability reporting and security advisories.
- Existing license, SBOM, provenance, and artifact-attestation workflows.

Security features can require additional permissions or licensing. An
inaccessible alert feed is not evidence that there are no alerts.

For a potential secret:

1. Stop unsafe publication.
2. Do not reproduce the secret in issues, comments, logs, or the final answer.
3. Remove it from unpublished changes.
4. Follow the approved revocation/rotation and incident process.
5. Do not rewrite shared history or bypass push protection without a separately
   authorized remediation plan.

For a potential undisclosed vulnerability:

- Follow `SECURITY.md`.
- Prefer the configured private reporting/advisory workflow.
- Avoid public branches, issues, PRs, or artifacts that prematurely disclose it.
- Distinguish remediation from public advisory publication.

Do not dismiss findings, label secrets as harmless, or weaken security policy
simply to unblock a merge.

## 6. Environments and deployments

Use existing deployment workflows, environments, and deployment records.

Before initiating deployment:

- Confirm explicit deployment authorization and the target environment.
- Verify the exact source commit or release artifact.
- Inspect required reviewers, branch restrictions, secrets, and wait rules.
- Follow the documented rollback and health-check process.

Prefer existing scoped tokens and OIDC-based cloud authentication where
configured. Do not introduce persistent credentials casually.

A merged PR, successful build, or published artifact is not a successful
deployment. Report the actual environment, deployment state, evidence, and
pending approval.

Do not approve a protected deployment through another identity or disable its
protection to complete the task.

## 7. Tags, releases, packages, and provenance

Use the repository's established versioning and release automation.

Read existing state before creating anything:

```bash
gh release list --repo "$REPO"
gh release view "$TAG" --repo "$REPO"
```

For an authorized release:

- Verify the version and exact intended commit.
- Check whether the tag, release, or package version already exists.
- Preserve required signing and provenance.
- Generate and inspect release notes.
- Verify assets, checksums, and attestations when required.
- Distinguish draft, prerelease, and final release.
- Verify publication after the operation.

Do not assume a missing tag should be created at the current default branch.
Do not overwrite published tags or immutable package versions.

GitHub Actions artifacts, release assets, and registry packages are different
delivery mechanisms. Choose the one the repository actually uses.

Release publication and deployment are separate operations unless an inspected,
approved workflow intentionally connects them.

## 8. Discussions, documentation, Pages, and search

Use native search before duplicating work:

- Issues and PRs for existing decisions and implementations.
- Code search for shared helpers and usage.
- Discussions for questions, proposals, and community conversation when the
  repository uses them.
- Wiki, repository docs, or Pages according to existing documentation policy.

Do not turn a question into an issue automatically.

Respect visibility boundaries when quoting search results or copying content
between repositories, Discussions, issues, and documentation.

Publishing Pages or editing a wiki is a remote write. Inspect the publication
workflow and intended audience before acting.

## 9. Forks, Codespaces, devcontainers, and delegated agents

For forks:

- Preserve upstream/head repository distinctions.
- Use the correct base and supported cross-repository PR syntax.
- Respect organization fork policy and private-code boundaries.
- Do not assume fork CI has access to upstream secrets.

Prefer an existing devcontainer or supported environment when it is the
repository's prescribed verification setup.

Do not create paid Codespaces or other billable resources without applicable
authorization.

Optional delegated-agent facilities, including preview commands such as
`gh agent-task` if present, are not part of the default execution path.

Before delegation:

- Verify availability and authorization.
- Define a bounded task and acceptance criteria.
- Select an appropriate agent and permitted tools.
- Prevent overlapping writes or duplicate PRs.
- Review and verify the resulting work.
- Do not transfer private code to an unapproved service.

Treat custom agents, hooks, and CLI extensions as executable configuration.
Inspect and approve relevant behavior; do not install or enable them merely
for convenience.

## 10. Repository and organization administration

Repository settings are not ordinary implementation details.

Separate authorization is required for changes such as:

- Branch protection and rulesets.
- Required checks, reviews, merge methods, and queues.
- Actions permissions and allowed actions.
- Environments, secrets, variables, and deployment restrictions.
- Collaborators, teams, GitHub Apps, webhooks, and token scopes.
- Repository visibility, transfer, archiving, or deletion.
- Security-feature configuration and billable capabilities.

For an authorized administrative task:

1. Read and record the existing configuration.
2. Identify the smallest proposed change and affected resources.
3. Explain security and workflow consequences.
4. Obtain any required approval before mutation.
5. Apply the change through supported interfaces.
6. Re-read and verify the resulting configuration.
7. Report the change and rollback path without exposing secrets.

Never treat weakening policy as an acceptable way to complete an unrelated PR.
