# Docker Build & Push Action

[![Unit Tests](https://github.com/mr-smithers-excellent/docker-build-push/actions/workflows/ci.yml/badge.svg)](https://github.com/mr-smithers-excellent/docker-build-push/actions/workflows/ci.yml)
[![e2e Tests](https://github.com/mr-smithers-excellent/docker-build-push/actions/workflows/nightly-e2e.yml/badge.svg)](https://github.com/mr-smithers-excellent/docker-build-push/actions/workflows/nightly-e2e.yml)
[![Maintainability](https://api.codeclimate.com/v1/badges/ac0bf06dc93ba3110cd3/maintainability)](https://codeclimate.com/github/mr-smithers-excellent/docker-build-push/maintainability)
[![Test Coverage](https://api.codeclimate.com/v1/badges/ac0bf06dc93ba3110cd3/test_coverage)](https://codeclimate.com/github/mr-smithers-excellent/docker-build-push/test_coverage)

Builds a Docker image and pushes it to the private registry of your choosing.

## Supported Docker registries

- Docker Hub
- Google Container Registry (GCR)
- AWS Elastic Container Registry (ECR)
- GitHub Docker Registry

## Features

- [Auto-tagging with GitOps](#auto-tagging-with-gitops)
- [BuildKit support](#buildkit-support)
- [Multi-platform builds](#multi-platform-builds)

## Breaking changes

If you're experiencing issues, be sure you are using the [latest stable release](https://github.com/mr-smithers-excellent/docker-build-push/releases/latest) (currently v6).

### v6
- Multi-platform builds now supported
- SSH agent forwarding

### v5
- AWS ECR [get-login command](https://docs.aws.amazon.com/cli/latest/reference/ecr/get-login.html) became deprecated, migrated to [get-login-password command](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecr/get-login-password.html)
- Support for multiple tags added
- BuildKit support added

## Basic usage

- Ensure you run the [checkout action](https://github.com/actions/checkout) before using this action
- Add the following to a workflow `.yml` file in the `/.github` directory of your repo

```yaml
steps:
  - uses: actions/checkout@v3
    name: Check out code

  - uses: mr-smithers-excellent/docker-build-push@v6
    name: Build & push Docker image
    with:
      image: repo/image
      tags: v1, latest
      registry: registry-url.io
      dockerfile: Dockerfile.ci
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

## Inputs

| Name           | Description                                                                                              | Required | Type    |
|----------------|----------------------------------------------------------------------------------------------------------|----------|---------|
| image          | Docker image name                                                                                        | Yes      | String  |
| tags           | Comma separated docker image tags (see [Auto-tagging with GitOps](#auto-tagging-with-gitops))            | No       | List    |
| appendMode     | Append tags to auto-generated GitOps tags instead of replacing them                                      | No       | Boolean |
| addLatest      | Adds the `latest` tag to the GitOps-generated tags                                                       | No       | Boolean |
| addTimestamp   | Suffixes a build timestamp to the branch-based Docker tag                                                | No       | Boolean |
| registry       | Docker registry host                                                                                     | Yes      | String  |
| dockerfile     | Location of Dockerfile (defaults to `Dockerfile`)                                                        | No       | String  |
| directory      | Directory to pass to `docker build` command, if not project root                                         | No       | String  |
| buildArgs      | Docker build arguments passed via `--build-arg`                                                          | No       | List    |
| labels         | Docker build labels passed via `--label`                                                                 | No       | List    |
| target         | Docker build target passed via `--target`                                                                | No       | String  |
| platform       | Docker build platform passed via `--platform`                                                            | No       | String  |
| username       | Docker registry username                                                                                 | No       | String  |
| password       | Docker registry password or token                                                                        | No       | String  |
| githubOrg      | GitHub organization to push image to (if not current)                                                    | No       | String  |
| enableBuildKit | Enables Docker BuildKit support                                                                          | No       | Boolean |
| multiPlatform  | Enables Docker buildx support                                                                            | No       | Boolean |
| overrideDriver | Disables setting up docker-container driver (if `true`, alternative docker driver must be set up)        | No       | Boolean |
| pushImage      | Flag for disabling the login & push steps, set to `true` by default                                      | No       | Boolean |

## Outputs

| Name          | Description                                        | Format                 |
|---------------|----------------------------------------------------|------------------------|
| imageFullName | Full name of the Docker image with registry prefix | `registry/owner/image` |
| imageName     | Name of the Docker image with owner prefix         | `owner/image`          |
| tags          | Tags for the Docker image                          | `v1,latest`            |

## Storing secrets

It is strongly recommended that you store all Docker credentials as GitHub [encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets). Secrets can be referenced in workflow files using the syntax `${{ secrets.SECRET_NAME }}`.

There is a distinction between secrets at the [repository](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository), [environment](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-environment) and [organization](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-an-organization) level. In general, you should store secrets at the repository or organization level, depending on your security posture. It is only recommended that you utilize environment-level secrets if your Docker credentials differ per environment (dev, staging, etc.).

## Examples

### Docker Hub

- Save your Docker Hub username (`DOCKER_USERNAME`) and password (`DOCKER_PASSWORD`) as secrets in your GitHub repo
- Modify sample below and include in your workflow `.github/workflows/*.yml` file

```yaml
uses: mr-smithers-excellent/docker-build-push@v6
with:
  image: docker-hub-repo/image-name
  registry: docker.io
  username: ${{ secrets.DOCKER_USERNAME }}
  password: ${{ secrets.DOCKER_PASSWORD }}
```

### Google Container Registry (GCR)

- Create a service account with the ability to push to GCR (see [configuring access control](https://cloud.google.com/container-registry/docs/access-control))
- Create and download JSON key for new service account
- Save content of `.json` file as a secret called `DOCKER_PASSWORD` in your GitHub repo
- Modify sample below and include in your workflow `.github/workflows/*.yml` file
- Ensure you set the username to `_json_key`

```yaml
uses: mr-smithers-excellent/docker-build-push@v6
with:
  image: gcp-project/image-name
  registry: gcr.io
  username: _json_key
  password: ${{ secrets.DOCKER_PASSWORD }}
```

### AWS Elastic Container Registry (ECR)

- Create an IAM user with the ability to push to ECR (see [example policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/ecr_managed_policies.html))
- Create and download access keys
- Save `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as secrets in your GitHub repo
- Ensure the repo you are trying to push to already exists, if not create with `aws ecr create-repository` before pushing
- Modify sample below and include in your workflow `.github/workflows/*.yml` file

```yaml
uses: mr-smithers-excellent/docker-build-push@v6
with:
  image: image-name
  registry: [aws-account-number].dkr.ecr.[region].amazonaws.com
env:
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### GitHub Container Registry

- GitHub recently [migrated their container registry](https://docs.github.com/en/packages/guides/migrating-to-github-container-registry-for-docker-images) from docker.pkg.github.com to ghcr.io
- It is assumed you'll be pushing the image to a repo inside your GitHub organization, unless you set `githubOrg`
- If using ghcr.io, provide the image name in `ghcr.io/OWNER/IMAGE_NAME` format
- If using docker.pkg.github.com, provide the image name in `docker.pkg.github.com/OWNER/REPOSITORY/IMAGE_NAME` format
- Provide either the `${{ github.actor }}` or an alternate username for Docker login (with associated token below)
- Pass the default GitHub Actions token or custom secret with [proper push permissions](https://docs.github.com/en/packages/guides/pushing-and-pulling-docker-images#authenticating-to-github-container-registry)

#### New ghcr.io

```yaml
uses: mr-smithers-excellent/docker-build-push@v6
with:
  image: image-name
  registry: ghcr.io
  githubOrg: override-org # optional
  username: ${{ secrets.GHCR_USERNAME }}
  password: ${{ secrets.GHCR_TOKEN }}
```

#### Legacy docker.pkg.github.com

```yaml
uses: mr-smithers-excellent/docker-build-push@v6
with:
  image: github-repo/image-name
  registry: docker.pkg.github.com
  username: ${{ github.actor }}
  password: ${{ secrets.GITHUB_TOKEN }}
```

## Auto-tagging with GitOps

By default, if you do not pass a `tags` input this action will use an algorithm based on the state of your git repo to determine the Docker image tag(s). This is designed to enable developers to more easily use [GitOps](https://www.weave.works/technologies/gitops/) in their CI/CD pipelines. Below is a table detailing how the GitHub trigger (branch or tag) determines the Docker tag(s).

| Trigger                  | Commit SHA | addLatest | addTimestamp | Docker Tag(s)                          |
|--------------------------|------------|-----------|--------------|----------------------------------------|
| /refs/tags/v1.0          | N/A        | false     | N/A          | v1.0                                   |
| /refs/tags/v1.0          | N/A        | true      | N/A          | v1.0,latest                            |
| /refs/heads/dev          | 1234567    | false     | true         | dev-1234567-2021-09-01.195027          |
| /refs/heads/dev          | 1234567    | true      | false        | dev-1234567,latest                     |
| /refs/heads/main         | 1234567    | false     | true         | main-1234567-2021-09-01.195027         |
| /refs/heads/main         | 1234567    | true      | false        | main-1234567,latest                    |
| /refs/heads/SOME-feature | 1234567    | false     | true         | some-feature-1234567-2021-09-01.195027 |
| /refs/heads/SOME-feature | 1234567    | true      | false        | some-feature-1234567,latest            |

### Adding custom tags to GitOps tags

The `appendMode` input allows you to add additional tags while keeping the auto-generated GitOps tags:

```yaml
uses: mr-smithers-excellent/docker-build-push@v6
with:
  image: repo/image
  registry: docker.io
  tags: stable,production
  appendMode: true
  username: ${{ secrets.DOCKER_USERNAME }}
  password: ${{ secrets.DOCKER_PASSWORD }}
```

For a `main` branch build with commit `1234567`, this would produce tags: `main-1234567,stable,production`

## BuildKit support

Enables [Docker BuildKit](https://docs.docker.com/build/buildkit/)

```yaml
steps:
  - uses: actions/checkout@v3
    name: Check out code

  - uses: mr-smithers-excellent/docker-build-push@v6
    name: Build & push Docker image
    with:
      image: repo/image
      registry: docker.io
      enableBuildKit: true
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

## Multi-platform builds

Enables [multi-platform builds](https://docs.docker.com/build/building/multi-platform/) with the default [docker-container driver](https://docs.docker.com/build/drivers/docker-container/)

```yaml
steps:
  - uses: actions/checkout@v3
    name: Check out code

  - uses: mr-smithers-excellent/docker-build-push@v6
    name: Build & push Docker image
    with:
      image: repo/image
      registry: docker.io
      multiPlatform: true
      platform: linux/amd64,linux/arm64,linux/arm/v7
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

Enables [multi-platform builds](https://docs.docker.com/build/building/multi-platform/) with custom driver

```yaml
steps:
  - uses: actions/checkout@v3
    name: Check out code

  # Required when overrideDriver is set to true
  - uses: docker/setup-buildx-action@v2
    name: Customize Docker driver
    with:
      driver-opts: image=moby/buildkit:v0.11.0

  - uses: mr-smithers-excellent/docker-build-push@v6
    name: Build & push Docker image
    with:
      image: repo/image
      registry: docker.io
      multiPlatform: true
      platform: linux/amd64,linux/arm64,linux/arm/v7
      overrideDriver: true
      username: ${{ secrets.DOCKER_USERNAME }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```
