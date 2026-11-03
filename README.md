# .github

Org-wide GitHub defaults and shared workflows for Cloud Four repositories.

GitHub gives a public repository named `.github` two special roles in an organization. This repo uses both, and it also hosts reusable workflows.

## Default community health files

Certain files placed here, such as a pull request template, `CONTRIBUTING.md` or `SECURITY.md`, become the default for **every** repository in the organization that doesn't have its own. That includes private repositories. A repository's own file always takes precedence, so adding one there overrides the default for just that repository.

Defaults don't appear in a repository's file list or in clones, so a contributor reading a checkout won't see them. Link to this repo when that matters.

Because a change here reaches every repository at once, keep these files general. Anything project-specific belongs in that project.

The defaults here are:

- [`.github/pull_request_template.md`](.github/pull_request_template.md): our standard pull request template, described in the [git guide](https://github.com/cloudfour/guides/tree/main/git#pull-request-template).

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

The issue also helps plan that release:

- **Upstream change:** each row says whether the dependency's own version moved by a `major`, `minor`, `patch` or `prerelease`, or was `added` or `removed`. Exact versions and simple `^` or `~` ranges like `^17.12.0` are compared by their base version. Anything more complex, such as `^1 || ^2`, shows as `range` and is flagged for checking by hand.
- **What changed:** each row links to the package's diff on [npmdiff.dev](https://npmdiff.dev), or to its npm page when there's no pair of versions to compare.
- **Suggested release:** a removed dependency, a major upstream release, or any change to `peerDependencies` suggests starting from a major. Everything else suggests patch or minor, depending on whether users see a difference.
- **Draft changelog:** one bullet per row, ready to copy into the release notes.

The suggestion is a floor based only on version numbers. Only the repository knows how it uses each dependency. A minor `globals` release can be a patch for a config that doesn't use the changed globals, and a minor release of a shared config can add error-level rules, which makes it a major. So review the changes before choosing a release, and add a sentence to each changelog bullet saying what it means for users.

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

### Oldest Node versions

[`oldest-node-versions.yml`](.github/workflows/oldest-node-versions.yml) finds the oldest Node version each part of `engines.node` in `package.json` allows, so CI can test the range a package publishes and not just the current Node. For `^22.13.0 || >=24` it outputs `["22.13.0","24.0.0"]`, ready for a matrix.

It reads the versions at run time on purpose. A `node-version` written into a workflow is something Renovate updates, which would quietly move the test away from the oldest supported version. With this, changing `engines` is the only edit needed.

Each part of the range can be an exact version or start with `^`, `~` or `>=`. A missing minor or patch counts as `0`, so `>=22` means `22.0.0`. A range whose oldest version isn't written in it, such as `>22` or `22.x`, fails the job instead of guessing.

To use it, add a job that calls it to the repository's CI workflow, and a test job that reads its `versions` output:

```yaml
jobs:
  oldest-node-versions:
    uses: cloudfour/.github/.github/workflows/oldest-node-versions.yml@main

  test-oldest-node:
    name: test (Node ${{ matrix.node-version }})
    needs: oldest-node-versions
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node-version: ${{ fromJSON(needs.oldest-node-versions.outputs.versions) }}
    steps:
      - uses: actions/checkout@v7.0.1
      - uses: actions/setup-node@v7.1.0
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm
      - run: npm ci
      - run: npm test
```

The calling workflow needs to grant at least `contents: read`, the permission the job needs to check out the repository.

## Shared actions

A reusable workflow replaces a whole job. When what's shared is a step that has to run inside a repository's own job, it's a composite action in `actions/` instead:

```yaml
steps:
  - uses: cloudfour/.github/actions/<action>@main
```

These reference `main` too, for the same reason as the reusable workflows.

### Oldest peer dependencies

[`oldest-peer-dependencies`](actions/oldest-peer-dependencies/action.yml) replaces each installed peer dependency with the oldest published release its range in `package.json` allows. Run after `npm ci`, it lets the tests check the oldest end of each range a package claims to support. It doesn't change `package.json` or the lockfile.

It asks npm which releases match each range and doesn't work it out from the range, because the oldest version a range allows may never have been published. TypeScript went from `6.0.1-rc` to `6.0.2`, so `>=6.0.0 <6.1.0` starts at `6.0.2`. A range that matches no published release fails with npm's `E404`, and a `package.json` with no `peerDependencies` fails too, since there's nothing to test.

Use it with the [oldest Node versions](#oldest-node-versions) job to test both floors at once:

```yaml
jobs:
  oldest-node-versions:
    uses: cloudfour/.github/.github/workflows/oldest-node-versions.yml@main

  test-oldest-supported:
    name: test-oldest-supported (Node ${{ matrix.node-version }})
    needs: oldest-node-versions
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        node-version: ${{ fromJSON(needs.oldest-node-versions.outputs.versions) }}
    steps:
      - uses: actions/checkout@v7.0.1
      - uses: actions/setup-node@v7.1.0
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm
      - run: npm ci
      - uses: cloudfour/.github/actions/oldest-peer-dependencies@main
      - run: npm test
```

## Workflow templates

Files in `workflow-templates/` appear as starter workflows under **Actions → New workflow** in every repository in the organization. A template is copied into the repository when it's used, so later changes here don't reach repositories that already use it.
