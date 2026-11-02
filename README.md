# .github

Org-wide GitHub defaults and shared workflows for Cloud Four repositories.

GitHub gives a public repository named `.github` two special roles in an organization. This repo uses both, and it also hosts reusable workflows.

## Default community health files

Certain files placed here, such as a pull request template, `CONTRIBUTING.md` or `SECURITY.md`, become the default for **every** repository in the organization that doesn't have its own. That includes private repositories. A repository's own file always takes precedence, so adding one there overrides the default for just that repository.

Defaults don't appear in a repository's file list or in clones, so a contributor reading a checkout won't see them. Link to this repo when that matters.

Because a change here reaches every repository at once, keep these files general. Anything project-specific belongs in that project.

## Reusable workflows

Workflows in `.github/workflows/` that run `on: workflow_call` can be called from any repository:

```yaml
jobs:
  example:
    uses: cloudfour/.github/.github/workflows/<workflow>.yml@main
```

Callers reference `main` rather than a tag, the same choice as [cloudfour/renovate-config](https://github.com/cloudfour/renovate-config#versioning): a fix here then reaches every repository without editing each one. The trade-off is that a bad commit here breaks every caller at once, so changes go through pull requests.

### Dependency drift check

[`dependency-drift.yml`](.github/workflows/dependency-drift.yml) notices when the default branch has dependency changes that users don't have yet. It compares `dependencies`, `optionalDependencies` and `peerDependencies` in `package.json` with the version on npm's `latest` tag, which is what a plain `npm install` gets. When they differ, it opens an issue labeled `unreleased dependencies` that lists each change. It updates that one issue as more changes land, and closes it once a release brings npm up to date.

It exists because automerged dependency updates are easy to lose track of. In a package that pins its runtime dependencies, every update needs a release to reach users. In one that uses ranges, as the [`library` Renovate preset](https://github.com/cloudfour/renovate-config#library) does, only an update outside the range or a manual edit does. Lockfile-only updates never show up here, because they don't change what users install.

To use it, add this as `.github/workflows/dependency-drift.yml` in the repository, or pick **Dependency drift check** under **Actions → New workflow**:

```yaml
name: Dependency drift

on:
  # A merged Renovate PR or a manual edit changes package.json.
  push:
    branches: [main]
    paths: [package.json]
  # Publishing doesn't push anything, so a daily run is what notices a
  # release and closes the issue.
  schedule:
    - cron: '17 15 * * *'
  workflow_dispatch:

permissions:
  contents: read
  issues: write

jobs:
  drift:
    uses: cloudfour/.github/.github/workflows/dependency-drift.yml@main
```

Some details:

- **Only the default branch manages the issue.** A manual run on any other branch writes its findings to the run's summary page instead, which is handy for checking a change before merging it.
- **Packages it skips:** anything marked `private`, never published to npm, or without a `package.json` at the repository root.
- **Releases in progress.** If `package.json` already has a version that isn't on npm yet, the issue says so and closes after the publish.
- **Inactive repositories.** GitHub turns off scheduled workflows in a public repository after 60 days without activity. The push trigger still runs, but the daily close may stop until someone re-enables the workflow under **Actions**.

## Workflow templates

Files in `workflow-templates/` appear as starter workflows under **Actions → New workflow** in every repository in the organization. A template is copied into the repository when it's used, so later changes here don't reach repositories that already use it.
