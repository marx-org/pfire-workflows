# pfire reusable workflows

This public repository contains the trusted GitHub Actions workflows shared by the
pfire container repositories. The caller repository owns the event trigger, source
checkout, release, GHCR image, and `GITHUB_TOKEN`; this repository owns the build
implementation.

## Multi-architecture Docker workflow

Call `.github/workflows/docker-image.yml` from a thin workflow in the caller repository:

```yaml
name: Docker Image CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  image:
    uses: marx-org/pfire-workflows/.github/workflows/docker-image.yml@COMMIT_SHA
    with:
      test_command: npm ci && npm test
      node_version: '22'
    secrets: inherit
    permissions:
      contents: write
      packages: write
```

The workflow defaults to `linux/amd64,linux/arm64`, creates a single GHCR manifest,
and uses the caller's `github.repository` for the image name. A caller can override
the Dockerfile/context, release branch, test command, build arguments, image suffix,
preview tag, cache scope, and release behavior through `workflow_call` inputs.

Develop callers set `release_branch: develop`, `publish_release: false`, and
`preview_tag: develop`. They still pass their full repository test command; the
shared workflow skips only release/tag creation and publishes the immutable SHA
tag plus the moving `develop` tag.

`package_token` is optional. The workflow falls back to the caller's automatic
`GITHUB_TOKEN`; callers that install private packages should grant `packages: read`
or `packages: write` and pass `secrets: inherit`. Docker package credentials are
provided to BuildKit as a secret named `NODE_AUTH_TOKEN`, not as a Docker build arg.
Dockerfiles must consume that secret with `RUN --mount=type=secret` and must not
interpolate credentials into a logged command.

## Release and security contract

- Pushes to the configured release branch create the next patch `v*` tag and GitHub Release.
- Required tests run before release/tag creation and image publication.
- Pull requests build without publishing; `[ci-skip]` skips push release/build work.
- Caller workflows must pin this repository to an immutable commit SHA.
- Changes to this public repository can execute with caller repository permissions, so
  branch protection, review, and SHA-pinned caller updates are mandatory.
- Release tags are managed by the reusable workflow; callers must not tag manually.

## Local validation

Run `actionlint .github/workflows/*.yml` before pushing changes. A real caller build
must verify both manifest platforms with:

```sh
docker buildx imagetools inspect ghcr.io/marx-org/<image>:<tag>
```
