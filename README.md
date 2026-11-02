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

## Workflow templates

Files in `workflow-templates/` appear as starter workflows under **Actions → New workflow** in every repository in the organization. A template is copied into the repository when it's used, so later changes here don't reach repositories that already use it.
