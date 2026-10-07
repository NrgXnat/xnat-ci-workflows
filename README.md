# xnat-ci-workflows

Reusable GitHub Actions workflows shared across NrgXnat repositories
(main `NrgXnat/xnat` and the plugin family).

## Workflows

### `gradle-build-publish.yml`

Builds a Gradle project, optionally publishes to JFrog Artifactory
(`nrgxnat.jfrog.io`) and/or builds a Docker image and pushes to
GitHub Container Registry (`ghcr.io/<owner>/<image-name>:<version>`).

## Caller template

Each consuming repo writes a thin `.github/workflows/build-publish.yml`
that defines its own `on:` triggers and calls this workflow via
`workflow_call`. Roughly 20 lines per repo.

### Main XNAT (`NrgXnat/xnat`)

```yaml
name: Build and Publish

on:
  push:
    branches: [main, develop, 'releases/*']
  workflow_dispatch:
    inputs:
      publish_jfrog: { type: boolean, default: false }
      publish_ghcr:  { type: boolean, default: false }

jobs:
  build-publish:
    uses: NrgXnat/xnat-ci-workflows/.github/workflows/gradle-build-publish.yml@v2
    with:
      gradle-build-args: '-x test'              # remove once tests are restored
      artifact-glob: 'xnat-web/build/libs/*.war'
      docker-context-filename: 'xnat.war'
      logback-path-in-war: 'WEB-INF/classes/logback.xml'
      publish-jfrog: ${{ github.event_name == 'push' || inputs.publish_jfrog }}
      publish-ghcr:  ${{ github.event_name == 'push' || inputs.publish_ghcr }}
      image-name: 'xnat-web'
    secrets: inherit
```

### Plugin — JFrog only (e.g. `NrgXnat/container-service`)

```yaml
name: Build and Publish

on:
  push:
    branches: [master, main, dev, develop, 'releases/*']     # different plugins use `dev` or `develop`
  workflow_dispatch:
    inputs:
      publish_jfrog: { type: boolean, default: true }

jobs:
  build-publish:
    uses: NrgXnat/xnat-ci-workflows/.github/workflows/gradle-build-publish.yml@v2
    with:
      artifact-glob: 'build/libs/*.jar'
      publish-jfrog: ${{ github.event_name == 'push' || inputs.publish_jfrog }}
      publish-ghcr:  false
    secrets: inherit
```

### 1.9.x release (Java 8, custom Tomcat base)

```yaml
jobs:
  build-publish:
    uses: NrgXnat/xnat-ci-workflows/.github/workflows/gradle-build-publish.yml@v2
    with:
      java-version: '8'
      artifact-glob: 'xnat-web/build/libs/*.war'
      docker-context-filename: 'xnat.war'
      logback-path-in-war: 'WEB-INF/classes/logback.xml'
      publish-jfrog: true
      publish-ghcr:  true
      image-name: 'xnat-web'
      docker-build-args: |
        TOMCAT_BASE=tomcat:9.0.93-jdk8
    secrets: inherit
```

## Required secrets in consuming repos

| Secret | Required when | Scope |
|---|---|---|
| `XNAT_ARTIFACTORY_USER`  | `publish-jfrog: true` | org-level (Selected repositories) |
| `XNAT_ARTIFACTORY_TOKEN` | `publish-jfrog: true` | org-level (Selected repositories) |
| `GITHUB_TOKEN`           | `publish-ghcr: true`  | auto-injected; consuming workflow needs `permissions: { packages: write }` for transitive GHCR push |

`secrets: inherit` in the caller passes everything through.

## Tag versions

Pin to a major tag in the caller (`@v2`). Breaking changes to inputs
or behavior bump the major tag. Lightweight `vN.M` tags may be added
later if we need patch-level pinning.

`v2` changes the release workflows only; inputs are unchanged, and
`gradle-build-publish.yml` is identical to `v1`:

- **Promote** no longer pushes the finalize commit to the RC branch. The
  RC branch stays at `-RC-SNAPSHOT` and only `main`'s build publishes the
  release. `main` and the tag are pushed atomically. A conflict on the
  version line alone (the previous release's finalize commit) is resolved
  in the RC's favour; any other conflict still fails the run.
- **Promote** stops before committing if a tracked Gradle script, a
  `gradle.properties` or a version catalog still names a SNAPSHOT version,
  such as `vXnat`. In scripts only quoted values count, and `//` comment
  lines are skipped. A hit inside `/* */` or a trailing comment still
  fails the run; there is no override.
- **Cut RC** pushes the RC branch and the source-branch bump atomically.

## What's NOT in this repo (lives in the consuming repo)

- The `Dockerfile` and its `docker/*.sh` helper scripts — those are
  intentionally per-project so each consumer can tune base image,
  environment, init scripts. The reusable workflow only invokes
  `docker build` against the caller's Dockerfile.
- Project-specific gradle build logic (buildSrc conventions, version
  layouts, onlyIf HEAD-probe protection for already-published release
  artifacts, etc).

The reusable workflow is intentionally thin: it's the *orchestration*
layer, not the *project* layer.
