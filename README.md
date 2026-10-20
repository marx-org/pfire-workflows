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
cache scope, and release behavior through `workflow_call` inputs.

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

## Node base-image mirror

`base-image-mirror.yml` copies the official multi-architecture `node:22-alpine` manifest
to `ghcr.io/marx-org/base-node:22-alpine` without rebuilding or changing its layers. It
adds the OCI source annotation needed to link the GHCR package to this repository, so the
destination index digest can differ from the source index digest while every manifest
descriptor remains identical. It runs nightly and can be dispatched manually:

```sh
gh workflow run base-image-mirror.yml \
  --repo marx-org/pfire-workflows \
  -f source_ref=node:22-alpine \
  -f destination_tag=22-alpine
```

The workflow resolves the source digest before copying, copies from that immutable digest,
and then checks that the source tag did not move, that the destination manifest descriptors
are identical, and that both required Linux platforms are present. It prints both index
digests for the fleet audit. The default source can be replaced by an explicit
`node:22-alpine@sha256:<digest>` when recovering a known-good version.

The first successful run creates the `base-node` GHCR package and links it to
`marx-org/pfire-workflows`; an organization owner must make the package public in its
settings before consumer migrations are merged. Keep consumer `FROM` lines pinned to the
verified destination digest; the mutable `22-alpine` tag is only for mirror maintenance.
If upstream is down, leave consumers on their last-known-good digest and do not repoint the
tag until a source digest has been verified.
