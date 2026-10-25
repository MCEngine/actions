# actions

Shared GitHub Actions workflows for the MCEngine family of Minecraft plugins.
Each plugin repository keeps its own thin `publish.yml` and `release.yml`, and
those call the reusable workflows here, so the release logic lives in one place.

| Workflow | What it does |
|---|---|
| [`publish.yml`](.github/workflows/publish.yml) | Tests, publishes the library modules to GitHub Packages, and tags `v{project-version}` when that tag does not exist yet. |
| [`release.yml`](.github/workflows/release.yml) | Builds the distributed jars from the tag and attaches them to a GitHub release, with the version's changelog as the notes. |

## Using them from a plugin

`.github/workflows/publish.yml`:

```yaml
name: Publish

on:
  push:
    branches: [master]
  workflow_dispatch:

concurrency:
  group: release
  cancel-in-progress: false

permissions: {}

jobs:
  publish:
    permissions:
      contents: write
      packages: write
    uses: MCEngine/actions/.github/workflows/publish.yml@master
    secrets: inherit
```

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  workflow_run:
    workflows: [Publish]
    types: [completed]
    branches: [master]
  workflow_dispatch:
    inputs:
      tag:
        description: Tag to release, such as v1.2.0. Leave it empty for the version in gradle.properties.
        required: false
        type: string

concurrency:
  group: github-release
  cancel-in-progress: false

permissions: {}

jobs:
  release:
    if: github.event_name == 'workflow_dispatch' || github.event.workflow_run.conclusion == 'success'
    permissions:
      contents: write
      actions: write
    uses: MCEngine/actions/.github/workflows/release.yml@master
    with:
      tag: ${{ inputs.tag }}
```

## Inputs

`publish.yml`

| Input | Default | Meaning |
|---|---|---|
| `java-version` | `25` | JDK used for the build. |
| `publish-packages` | `true` | Run `publish-task`. Set `false` for a plugin that publishes no library; the run still tests and tags. |
| `build-task` | `build` | Gradle task that builds and tests. |
| `publish-task` | `publish` | Gradle task that publishes to GitHub Packages. |

`release.yml`

| Input | Default | Meaning |
|---|---|---|
| `tag` | empty | Tag to release; empty means `v{project-version}`. |
| `jars` | `platforms/engine/build/libs/*-{version}.jar` | Newline-separated globs; `{version}` is replaced, and each glob must match one jar. |
| `java-version` | `25` | JDK used for the build. |
| `assemble-task` | `assemble` | Gradle task that builds the jars. |

## Conventions the workflows rely on

* The version is `project-version` in `gradle.properties`, as `Major.Minor.Patch`.
  `0.0.0` means nothing has been released and is never tagged.
* The default branch is `master`.
* Release notes come from `wiki/logs/{Major}/{Minor}/{Patch}/CHANGELOG.md` when it
  exists in the tagged commit; otherwise GitHub lists the merged pull requests.
* Publishing needs `GITHUB_ACTOR` and `GITHUB_TOKEN` read by the Gradle
  `GitHubPackages` repository, which the workflow sets.
