
# sbom-operator

> Catalogue all images of a Kubernetes cluster to multiple targets with Syft.

[![test](https://github.com/ckotzbauer/sbom-operator/actions/workflows/test.yml/badge.svg)](https://github.com/ckotzbauer/sbom-operator/actions/workflows/test.yml)

## Overview

This operator maintains a central place to track all packages and software used in all those images in a Kubernetes cluster. For this a Software Bill of 
Materials (SBOM) is generated from each image with Syft. They are all stored in one or more targets. Currently Git, Dependency Track and OCI-Registry are supported. 
With this it is possible to do further analysis, [vulnerability scans](https://github.com/ckotzbauer/vulnerability-operator) and much more in a single place. 
To prevent scans of images that have already been analyzed pods are annotated with the imageID of the already processed image.

## Kubernetes Compatibility

The image contains versions of `k8s.io/client-go`. Kubernetes aims to provide forwards & backwards compatibility of one minor version between client and server:

| sbom-operator   | k8s.io/{api,apimachinery,client-go} | expected kubernetes compatibility |
|-----------------|-------------------------------------|-----------------------------------|
| main            | v0.24.2                             | 1.23.x, 1.24.x, 1.25.x            |
| 0.15.0          | v0.24.4                             | 1.23.x, 1.24.x, 1.25.x            |
| 0.14.0          | v0.24.3                             | 1.23.x, 1.24.x, 1.25.x            |
| 0.13.0          | v0.24.2                             | 1.23.x, 1.24.x, 1.25.x            |
| 0.12.0          | v0.24.1                             | 1.23.x, 1.24.x, 1.25.x            |
| 0.11.0          | v0.24.0                             | 1.23.x, 1.24.x, 1.25.x            |
| 0.10.0          | v0.23.6                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.9.0           | v0.23.5                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.8.0           | v0.23.5                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.7.0           | v0.23.4                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.6.0           | v0.23.4                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.5.0           | v0.23.4                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.4.1           | v0.23.3                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.3.1           | v0.23.3                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.2.0           | v0.23.2                             | 1.22.x, 1.23.x, 1.24.x            |
| 0.1.0           | v0.23.2                             | 1.22.x, 1.23.x, 1.24.x            |

However, the operator will work with more versions of Kubernetes in general.

## Container Registry Support

The operator relies on the [go-containeregistry](https://github.com/google/go-containerregistry) library to download images. It should work with most registries. 
These are officially tested (with authentication):
- ACR (Azure Container Registry) (currently not unit-tested)
- ECR (Amazon Elastic Container Registry)
- GAR (Google Artifact Registry)
- GCR (Google Container Registry)
- GHCR (GitHub Container Registry)
- DockerHub


## Installation

#### Manifests

```
kubectl apply -f deploy/standard/
```

#### Helm-Chart

Create a YAML file first with the required configurations or use helm-flags instead.

```
helm repo add ckotzbauer https://ckotzbauer.github.io/helm-charts
helm install ckotzbauer/sbom-operator -f your-values.yaml
```


## Configuration

All parameters are cli-flags. The flags can be configured as args or as environment-variables prefixed with `SBOM_` to inject sensitive configs as secret values.

### Common parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `verbosity` | `false` | `info` | Log-level (debug, info, warn, error, fatal, panic) |
| `cron` | `false` | `""` | Backround-Service interval (CRON). See [Trigger](#analysis-trigger) for details. |
| `ignore-annotations` | `false` | `false` | Force analyzing of all images, including those from annotated pods. |
| `format` | `false` | `json` | SBOM-Format. (One of `json`, `syftjson`, `cyclonedxjson`, `spdxjson`, `github`, `githubjson`, `cyclonedx`, `cyclone`, `cyclonedxxml`, `spdx`, `spdxtv`, `spdxtagvalue`, `text`, `table`) |
| `targets` | `false` | `git` | Comma-delimited list of targets to sent the generated SBOMs to. Possible targets `git`, `dtrack`, `oci`. Ignored with a `job-image` |
| `pod-label-selector` | `false` | `""` | Kubernetes Label-Selector for pods. |
| `namespace-label-selector` | `false` | `""` | Kubernetes Label-Selector for namespaces. |
| `fallback-image-pull-secret` | `false` | `""` | Kubernetes Pull-Secret Name to load as a fallback when all others fail (must be in the same namespace as the sbom-operator) |


### Example Helm-Config

```yaml
args:
  targets: git
  git-author-email: XXX
  git-author-name: XXX
  git-repository: https://github.com/XXX/XXX
  git-path: dev-cluster/sboms
  verbosity: debug
  cron: "0 30 * * * *"

envVars:
  - name: SBOM_GIT_ACCESS_TOKEN
    valueFrom:
      secretKeyRef:
        name: "sbom-operator"
        key: "accessToken"
```

## Analysis-Trigger

### Cron

With the `cron` flag set, the operator runs with a specified interval and checks for changed images in your cluster.
All options from [github.com/robfig/cron](https://github.com/robfig/cron) are allowed as cron-syntax.

### Real-Time

When you omit the `cron` flag, the operator uses a Cache-Informer to process changed pods immediately. In this mode there's also
a one-time analysis at startup to sync the targets with the actual cluster-state. If you configured a job-image there's no initial
startup sync.


## Targets

It is possible to store the generated SBOMs to different targets (even multple at once). All targets are using Syft as analyzer.
If you want to use another tool to analyze your images, then have a look at the [Job image](#job-images) section. Images which are
not present in the cluster anymore are removed from the configured targets (except for the OCI-Target).

### Dependency Track

#### Dependency Track Parameter

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `dtrack-base-url` | `true` when `dtrack` target is used | `""` | Dependency-Track base URL, e.g. 'https://dtrack.example.com' |
| `dtrack-api-key` | `true` when `dtrack` target is used | `""` | Dependency-Track API key |
| `kubernetes-cluster-id` | `false` | `"default"` | Kubernetes Cluster ID (to be used in Dependency-Track or Job-Images) |

Each image in the cluster is created as project with the full-image name (registry and image-path without tag) and the image-tag as project-version.
When there's no image-tag, but a digest, the digest is used as project-version.
The `autoCreate` option of DT is used. You have to set the `--format` flag to `cyclonedx` with this target.


### Git

#### Git Parameter

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `git-workingtree` | `false` | `/work` | Directory to place the git-repo. |
| `git-repository` | `true` when `git` target is used. | `""` | Git-Repository-URL (HTTPS). |
| `git-branch` | `false` | `main` | Git-Branch to checkout. |
| `git-path` | `false` | `""` | Folder-Path inside the Git-Repository. |
| `git-access-token` | `true` when `git` target is used. | `""` | Git-Personal-Access-Token with write-permissions. |
| `git-author-name` | `true` when `git` target is used. | `""` | Author name to use for Git-Commits. |
| `git-author-email` | `true` when `git` target is used. | `""` | Author email to use for Git-Commits. |

The operator will save all files with a specific folder structure as described below. When a `git-path` is configured, all folders above this path are not touched
from the application. Assuming that `git-path` is set to `dev-cluster/sboms`. When no `git-path` is given, the structure below is directly in the repository-root. 
The structure is basically `<git-path>/<registry-server>/<image-path>/<image-digest>/sbom.json`. The file-extension may differ when another output-format is configured. A token-based authentication to the git-repository is used (PAT).

```
dev-cluster
│
└───sboms
    │
    └───docker.io
    |   │
    |   └───library
    |       │
    |       └───busybox
    |           │
    |           └───sha256_ae39a6f5...
    |               │   sbom.json
    |
    └───ghcr.io
        │
        └───kyverno
            │
            └───kyverno
            |   │
            |   └───sha256_9e3f14e5...
            |       │   sbom.json
            |
            └───kyvernopre
                │
                └───sha256_e48f87fd...
                    │   sbom.json
            |
            └───policy-reporter
                │
                └───sha256_b70caa7a...
                    │   sbom.json
```

### OCI-Registry

#### OCI-Registry Parameter

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `oci-registry` | `true` when `oci` target is used | `""` | OCI-Registry |
| `oci-user` | `true` when `oci` target is used | `""` | OCI-User |
| `oci-token` | `true` when `oci` target is used | `""` | OCI-Token |

In this mode the operator will generate a SBOM and store it into an OCI-Registry. The SBOM then can be processed by cosign, Kyverno
or any other tool. E.g.:
```bash
COSIGN_REPOSITORY=<yourregistry> cosign download sbom <your full image digest>
```

The operator needs the Registry-URL, a user and a token as password to authenticate to the registry. Write-permissions are needed.


## Job-Images

#### Job-Image Parameter

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `job-image` | `false` | `""` | Job-Image to process images with instead of Syft |
| `job-image-pull-secret` | `false` | `""` | Pre-existing pull-secret-name for private job-images |
| `job-timeout` | `false` | `3600` | Job-Timeout in seconds (`activeDeadlineSeconds`) |
| `kubernetes-cluster-id` | `false` | `"default"` | Kubernetes Cluster ID (to be used in Dependency-Track or Job-Images) |

If you don't want to use Syft to analyze your images, you can give the Job-Image feature a try. The operator creates a Kubernetes-Job
which does the analysis with any possible tool inside. There's no target-handling done by the operator, the tool from the job has to process
the SBOMs on its own. Currently there are two possible integrations:

| Tool | Description |
| ---- | ----------- |
| [Codenotary CAS](job-images/cas/README.md) | The Community Attestation Service from Codenotary can notarize your images in the Codenotary Cloud. (free) |
| [Codenotary VCN](job-images/vcn/README.md) | The VCN-Tool from Codenotary can notarize your images in the Codenotary Cloud. (chargeable) |

This feature is built as generic approach. Any image which follows [these specs](job-images/SPEC.md) can be used as job-image.

e.g. Manifest (`deploy/job-image`):
```yaml
--job-image=ghcr.io/ckotzbauer/sbom-operator/cas:<TAG>
```

e.g. Helm:
```yaml
jobImageMode: true

envVars:
  - name: SBOM_JOB_CAS_API_KEY
    value: "<KEY>"
```

All operator-environment variables prefixed with `SBOM_JOB_` are passed to the Kubernetes job.


## Security

The docker-image is based on `scratch` to reduce the attack-surface and keep the image small. Furthermore the image and release-artifacts are signed 
with [cosign](https://github.com/sigstore/cosign) and attested with provenance-files. The release-process satisfies SLSA Level 2. All of those "metadata files" are 
also stored in a dedicated repository `ghcr.io/ckotzbauer/sbom-operator-metadata`.
When discovering security issues please refer to the [Security process](https://github.com/ckotzbauer/.github/blob/main/SECURITY.md).

### Signature verification

```bash
COSIGN_EXPERIMENTAL=1 COSIGN_REPOSITORY=ghcr.io/ckotzbauer/sbom-operator-metadata cosign verify ghcr.io/ckotzbauer/sbom-operator:<tag-to-verify> --certificate-github-workflow-name create-release --certificate-github-workflow-repository ckotzbauer/sbom-operator
```

### Attestation verification

```bash
COSIGN_EXPERIMENTAL=1 COSIGN_REPOSITORY=ghcr.io/ckotzbauer/sbom-operator-metadata cosign verify-attestation ghcr.io/ckotzbauer/sbom-operator:<tag-to-verify> --certificate-github-workflow-name create-release --certificate-github-workflow-repository ckotzbauer/sbom-operator
```

### Download attestation

```bash
COSIGN_REPOSITORY=ghcr.io/ckotzbauer/sbom-operator-metadata cosign download attestation ghcr.io/ckotzbauer/sbom-operator:<tag-to-verify> | jq -r '.payload' | base64 -d
```

### Download SBOM

```bash
COSIGN_REPOSITORY=ghcr.io/ckotzbauer/sbom-operator-metadata cosign download sbom ghcr.io/ckotzbauer/sbom-operator:<tag-to-verify> | jq -r '.payload' | base64 -d
```


[License](https://github.com/ckotzbauer/sbom-operator/blob/master/LICENSE)
--------
[Changelog](https://github.com/ckotzbauer/sbom-operator/blob/master/CHANGELOG.md)
--------


## Contributing

Please refer to the [Contribution guildelines](https://github.com/ckotzbauer/.github/blob/main/CONTRIBUTING.md).

## Code of conduct

Please refer to the [Conduct guildelines](https://github.com/ckotzbauer/.github/blob/main/CODE_OF_CONDUCT.md).

