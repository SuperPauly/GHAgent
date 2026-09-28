# GitHub Development Workflow

A coding-agent skill for taking repository work from a request to an accurate GitHub handoff. It uses GitHub's own issues, branches, pull requests, checks, reviews, and merge policy when they help the task, while preserving unrelated work in the user's checkout.

The skill lives at [`.github/skills/github-development-workflow/SKILL.md`](.github/skills/github-development-workflow/SKILL.md). Its [feature playbook](.github/skills/github-development-workflow/references/github-features.md) covers capabilities beyond the ordinary issue-to-PR path.

## What it does

| Stage | GitHub capabilities the agent can use |
| --- | --- |
| Plan | Repository instructions, rulesets, classic protection, issue search, issue types, sub-issues, dependencies, Projects, milestones |
| Develop | Issue-linked branches, forks, isolated worktrees, existing PR detection |
| Verify | Local checks, Actions runs and artifacts, required checks, code and secret scanning, dependency review |
| Review | Draft PRs, templates, CODEOWNERS, reviewer requests, conversations, branch updates |
| Deliver | Allowed merge methods, merge queues, auto-merge, releases, packages, deployments, and accurate completion states |

The agent selects features that fit the repository's existing workflow. It does not create an issue for every edit, change repository policy to unblock a PR, or treat an enabled auto-merge as a completed merge.

## Install

This repository already places the skill in GitHub Copilot's [project skill location](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills). To use it in another repository, copy the complete `github-development-workflow` directory into that repository's `.github/skills/` directory and commit it.

With GitHub CLI **2.90.0 or newer**, you can install the published repository skill for a local coding agent. The path is explicit because the source lives under `.github/skills/`:

```bash
gh skill preview SuperPauly/GHAgent .github/skills/github-development-workflow/SKILL.md --allow-hidden-dirs
gh skill install SuperPauly/GHAgent .github/skills/github-development-workflow/SKILL.md --allow-hidden-dirs --agent codex --scope user
```

For a repository-scoped Copilot installation, use `--agent github-copilot --scope project` instead. Review the skill before installing it. The `gh skill` command is [in preview](https://cli.github.com/manual/gh_skill_install); older CLI versions can use the directory-copy method above.

## Use

Ask the coding agent to use `github-development-workflow` for GitHub-backed tasks, for example:

```text
Use the github-development-workflow skill to fix issue #42, run the relevant checks, and open a draft PR.
```

The skill starts by determining the requested stopping point. A task may end with local work, a pushed branch, a draft or ready PR, an authorized merge, or a documented blocker. Publishing a branch does not imply permission to merge, deploy, release, or administer the repository.

It checks the installed `gh` version and command help before relying on newer flags. For example, current GitHub CLI documents an issue-linked worktree with `gh issue develop --checkout --worktree <path>`, but older versions lack `--worktree`. The skill falls back to Git worktrees or a supported API where appropriate and reports unavailable native relationships honestly. See the [GitHub CLI issue development manual](https://cli.github.com/manual/gh_issue_develop).

## Principles

- Preserve unrelated changes, credentials, branches, and worktrees.
- Read applicable repository instructions and policy before editing or publishing.
- Reuse existing issues, branches, PRs, labels, and Projects rather than creating duplicates.
- Use closing keywords only when the PR truly completes the issue.
- Associate verification with the current PR head; recheck after a push or branch update.
- Respect required reviews, security checks, deployments, and merge queues.
- Confirm a PR actually merged before cleaning up agent-owned work.

The [skill](.github/skills/github-development-workflow/SKILL.md) contains the full workflow and delivery-state definitions. GitHub's [agent skill documentation](https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills) and [CLI skill manual](https://cli.github.com/manual/gh_skill) describe host support and installation options.
