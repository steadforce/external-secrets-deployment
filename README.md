# External Secrets Deployment

[![Helm unittest](https://github.com/steadforce/external-secrets-deployment/actions/workflows/helm-unittest.yaml/badge.svg)](https://github.com/steadforce/external-secrets-deployment/actions/workflows/helm-unittest.yaml)
[![Helm hydration](https://github.com/steadforce/external-secrets-deployment/actions/workflows/helm-hydration.yaml/badge.svg)](https://github.com/steadforce/external-secrets-deployment/actions/workflows/helm-hydration.yaml)
[![Trufflehog](https://github.com/steadforce/external-secrets-deployment/actions/workflows/trufflehog.yaml/badge.svg)](https://github.com/steadforce/external-secrets-deployment/actions/workflows/trufflehog.yaml)

Umbrella chart for deploying [`external-secrets`](https://charts.external-secrets.io) with Argo CD. This repository
packages the upstream chart, adds SteadOps-specific bootstrap resources, and defines environment-specific value
files for hydration.

> [!IMPORTANT]
> Do not install this chart manually on clusters. Argo CD is responsible for all deployments; the commands below
> are for local rendering, testing, and dependency management only.

## Repository Layout

- `Chart.yaml` — declares the upstream `external-secrets` dependency and this umbrella chart's own version.
- `helm-config.yaml` — the hydration scope: environments, allowed API versions, and value file mapping.
- `values.yaml`, `values-*.yaml` — default, local, development, and production settings, plus the
  subchart overrides shared by every environment.
- `templates/` — SteadOps-specific bootstrap secret and `ClusterSecretStore` templates.
- `charts/` — the vendored `external-secrets` dependency archive, fetched by `helm dependency update` and not
  committed.
- `tests/` — Helm unittest suites covering CRDs, resource sizing, and environment-specific behavior.
- `.github/workflows/` — CI: unit tests, manifest hydration, and secret scanning.

## Environments

Environments and their value files are declared in `helm-config.yaml`, which the hydration pipeline reads to
render manifests per cluster. Do not add `-f` overrides that aren't listed there — they will not be applied by
the pipeline.

| Environment                                    | Value Files                                                    |
| ----------------------------------------------- | ---------------------------------------------------------------- |
| `local`                                         | `values-subchart-overrides.yaml`, `values-local.yaml`             |
| `sf-k8s01-dev`, `sf-k8s02-dev`, `sf-k8s03-dev`   | `values-subchart-overrides.yaml`, `values-development.yaml`       |
| `sf-k8s01-prod`                                  | `values-subchart-overrides.yaml`, `values-production.yaml`        |

> [!TIP]
> When adding a new environment value file, register it in `helm-config.yaml` and cover it with a matching Helm
> unittest case. If the new environment reuses an existing value file set unchanged, extending the existing test
> case is usually enough.

## Configuration Overview

The chart uses AWS Systems Manager Parameter Store as the secret backend. The bootstrap credentials are passed
through these values:

- `aws.accessKeyId`
- `aws.secretAccessKey`

Set `bootstrapResources.enabled=true` to render the bootstrap secret. The secret is named `awssm-secret` and is
used by the `awssm-parameter-store` `ClusterSecretStore`.

The most important value files are:

- `values-subchart-overrides.yaml` — subchart-specific overrides shared by every hydrated environment.
- `values-local.yaml` — local resource sizing.
- `values-development.yaml` — development clusters.
- `values-production.yaml` — production clusters, including the production AWS role.

## Common Workflows

### Update Chart Dependencies

```sh
 docker run --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

### Render Manifests for an Environment

The example below renders the `local` environment. For another environment, swap in the `-f` files listed for it
in `helm-config.yaml`.

```sh
 docker run --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template external-secrets . \
   --include-crds \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml
```

### Render the Bootstrap Secret

Only needed once per cluster, to seed the AWS credentials the `ClusterSecretStore` authenticates with.

```sh
 docker run --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template external-secrets . \
   --include-crds \
   --set aws.accessKeyId="<aws-access-key-id>" \
   --set aws.secretAccessKey="<aws-secret-access-key>" \
   --set bootstrapResources.enabled=true \
   -s templates/awssm-secret.yaml
```

### Run Helm Unittest

```sh
 docker run --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

### Run Helm Unittest with JUnit Output

```sh
 docker run --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest -o test-output.xml .
```

## Testing

`tests/` covers, per Helm unittest suite:

- One suite per CRD shipped by the `external-secrets` subchart, asserting the Argo CD server-side apply
  annotation from `values-subchart-overrides.yaml` is set, plus a snapshot for broad regression detection.
- Resource sizing (CPU/memory) and container image stability for the core controller, webhook, and
  cert-controller deployments, asserted separately for local and non-local environments.
- `ServiceMonitor` labels, the `awssm-secret` bootstrap secret, and the `awssm-parameter-store`
  `ClusterSecretStore` settings per environment.

> [!NOTE]
> Snapshot files under `tests/__snapshot__/` are generated locally by Helm unittest and are gitignored — do not
> commit them.

## Continuous Integration

- **Helm unittest** (`helm-unittest.yaml`) — runs the test suite on every push.
- **Helm hydration** (`helm-hydration.yaml`) — renders manifests for every environment in `helm-config.yaml` on
  push to `main`.
- **Trufflehog** (`trufflehog.yaml`) — scans for committed secrets on push, pull request, and manual dispatch.
- **Renovate** (`renovate.json`) — keeps the `external-secrets` chart dependency and GitHub Actions up to date.
  GitHub Actions updates and chart patch updates auto-merge; chart minor updates are split into a separate
  branch per major/minor version, capped below the next major version's `.1.0` release, for manual review.

All three workflows call reusable workflows from `steadforce/steadops-workflows`.

## Local Development

To run the repository's GitHub workflows locally, start the `SteadOps-Steadies-K8s-Workplace` workbench — which
also provides `helm`, `yq`, `kubectl`, and `hetzner-k3s` for related cluster work — change into this repository,
and run:

```sh
 act
```

On the first run, `act` asks which image flavor to use. The default `medium` is a good starting point.
