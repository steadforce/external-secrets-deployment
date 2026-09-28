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

## Prerequisites

- [Docker](https://www.docker.com/) for the containerized commands, or
- the `SteadOps-Steadies-K8s-Workplace` workbench, which provides `helm`, `yq`, `kubectl`, `hetzner-k3s`, and
  `act` directly in its shell.

All commands run from the repository root.

## Repository Layout

| Path                         | Purpose                                                                    |
| ---------------------------- | -------------------------------------------------------------------------- |
| `Chart.yaml`                 | Upstream `external-secrets` dependency and this umbrella chart's version   |
| `Chart.lock`                 | Committed lock file pinning the resolved dependency version and digest     |
| `helm-config.yaml`           | Hydration scope: environments, allowed API versions, value file mapping    |
| `values.yaml`                | Chart defaults: the default AWS role (`aws.role`)                          |
| `values-*.yaml`              | Local, development, and production settings, plus shared subchart overrides |
| `templates/`                 | Bootstrap secret and `ClusterSecretStore` templates                        |
| `tests/`                     | Helm unittest suites                                                       |
| `charts/`                    | Dependency archive installed by `helm dependency build`; gitignored        |
| `.github/workflows/`         | CI: unit tests, manifest hydration, and secret scanning                    |
| `renovate.json`              | Renovate configuration for chart and GitHub Actions updates                |

## Environments

Environments and their value files are declared in `helm-config.yaml`, which the hydration pipeline reads to
render manifests per cluster. Do not add `-f` overrides that aren't listed there — they will not be applied by
the pipeline.

| Environment     | Value Files                                                 |
| --------------- | ----------------------------------------------------------- |
| `local`         | `values-subchart-overrides.yaml`, `values-local.yaml`       |
| `sf-k8s01-dev`  | `values-subchart-overrides.yaml`, `values-development.yaml` |
| `sf-k8s02-dev`  | `values-subchart-overrides.yaml`, `values-development.yaml` |
| `sf-k8s03-dev`  | `values-subchart-overrides.yaml`, `values-development.yaml` |
| `sf-k8s04-dev`  | `values-subchart-overrides.yaml`, `values-development.yaml` |
| `sf-k8s01-prod` | `values-subchart-overrides.yaml`, `values-production.yaml`  |

> [!TIP]
> When adding a new environment value file, register it in `helm-config.yaml` and cover it with a matching Helm
> unittest case. If the new environment reuses an existing value file set unchanged, extending the existing test
> case is usually enough.

## Configuration Overview

The chart uses AWS Systems Manager Parameter Store as the secret backend. `values.yaml` sets only `aws.role`,
the IAM role the `awssm-parameter-store` `ClusterSecretStore` assumes; `values-production.yaml` replaces it with
the production role. The bootstrap credentials are passed through these values:

- `aws.accessKeyId`
- `aws.secretAccessKey`

Set `bootstrapResources.enabled=true` to render the bootstrap secret. The secret is named `awssm-secret` and is
used by the `awssm-parameter-store` `ClusterSecretStore`.

The most important value files are:

- `values-subchart-overrides.yaml` — subchart-specific overrides shared by every hydrated environment.
- `values-local.yaml` — local resource sizing.
- `values-development.yaml` — development clusters.
- `values-production.yaml` — production clusters, including the production AWS role.

## Setup

`Chart.lock` is committed and pins the `external-secrets` subchart version. Install that version into `charts/`
after cloning and after every pull that changes `Chart.lock`. The unit tests render whatever sits in `charts/`,
so a stale archive passes the whole suite while testing the wrong subchart.

`helm dependency build` resolves HTTP(S) repositories by their registered name, so the repositories declared in
`Chart.yaml` are registered first, the same way the pipeline does.

In the workbench:

```sh
 yq 'explode(.) | .dependencies[] | select(.repository == "http*") | .name + " " + .repository' Chart.yaml |
   while read -r name repo; do helm repo add --force-update "$name" "$repo"; done
 helm dependency build .
```

With Docker:

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     yq "explode(.) | .dependencies[] | select(.repository == \"http*\") | .name + \" \" + .repository" Chart.yaml |
       while read -r name repo; do helm repo add --force-update "$name" "$repo"; done &&
     helm dependency build .
   '
```

`helm dependency build` fails when `Chart.lock` and `Chart.yaml` disagree; see
[Dependency Updates](#dependency-updates) for regenerating the lock file.

## Rendering

The example below renders the `local` environment. For another environment, swap in the `-f` files listed for it
in `helm-config.yaml`. In the workbench, run the same `helm template` arguments without the `docker run` wrapper.

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template external-secrets . \
   -a external-secrets.io/v1/ClusterSecretStore \
   -f values-subchart-overrides.yaml \
   -f values-local.yaml \
   --include-crds
```

> [!IMPORTANT]
> `-a` is not optional here. `templates/awssm-cluster-secret-store.yaml` is gated on the
> `external-secrets.io/v1/ClusterSecretStore` capability, so without it the `ClusterSecretStore` is silently
> left out of the rendered output. The hydration pipeline supplies the API versions from the `apis` list in
> `helm-config.yaml`.

### Render the Bootstrap Secret

Only needed once per cluster, to seed the AWS credentials the `ClusterSecretStore` authenticates with.

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template external-secrets . \
   --include-crds \
   -s templates/awssm-secret.yaml \
   --set aws.accessKeyId="<aws-access-key-id>" \
   --set aws.secretAccessKey="<aws-secret-access-key>" \
   --set bootstrapResources.enabled=true
```

## Testing

Run the Helm unittest suites after [Setup](#setup). Tests of the subcharts under `charts/` run as well, as in
the pipeline.

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

helm-unittest writes XUnit by default. To get the JUnit report the pipeline publishes, add
`-o test-output.xml -t JUnit`; `test-output.xml` is gitignored.

### Refresh Test Snapshots

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest -u .
```

Needed after an intentional manifest change, most often a subchart version bump. Snapshots prove nothing on
their own, since refreshing them silences a regression just as easily as it records an intended change — rely on
the direct assertions for anything that must not change.

### Test Coverage

`tests/` covers, per Helm unittest suite:

- One suite per CRD shipped by the `external-secrets` subchart, asserting the Argo CD server-side apply
  annotation from `values-subchart-overrides.yaml` is set, plus a snapshot for broad regression detection.
- Resource sizing (CPU/memory) and container image stability for the core controller, webhook, and
  cert-controller deployments, asserted separately for local and non-local environments.
- `ServiceMonitor` labels, the `awssm-secret` bootstrap secret, and the `awssm-parameter-store`
  `ClusterSecretStore` settings per environment.
- The two gates that decide whether a resource exists at all: the `ServiceMonitor` resources appear only
  because `values-subchart-overrides.yaml` enables them, and the `ClusterSecretStore` renders only once the
  `external-secrets` CRDs are present.

> [!NOTE]
> Snapshot files under `tests/__snapshot__/` are generated locally by Helm unittest and are gitignored — do not
> commit them.

## CI/CD

All workflows call reusable workflows from `steadforce/steadops-workflows`, pinned to `v4.2.0`.

- **Helm unittest** (`helm-unittest.yaml`) — runs on every push. Installs the subchart pinned by `Chart.lock`
  with `helm dependency build`, runs the suites including subchart tests, publishes a JUnit report, and runs
  `helm lint`. On `renovate/` branches it posts the result to Microsoft Teams: successes go to
  `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK`, failures to `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK`
  (a separate error channel). When the error secret is unset, failures fall back to the regular webhook; with
  neither secret set, no notification is sent.
- **Helm hydration** (`helm-hydration.yaml`) — runs on push to `main`. Builds the dependencies from
  `Chart.lock`, renders every environment in `helm-config.yaml` with `helm template`, adds Argo CD server-side
  apply and sync-wave `-1` annotations to CRDs, and opens one pull request per environment against the
  `environments/<name>` branch. Patch updates of the subchart share one pull request branch. The caller grants
  `issues: write` so the workflow can create the coloured `env: <name>` label for these pull requests.
- **Trufflehog** (`trufflehog.yaml`) — scans the pushed commit range for secrets on push and pull request to
  `main`, and on manual dispatch.

## Dependency Updates

Renovate (`renovate.json`) keeps the `external-secrets` chart dependency and GitHub Actions up to date:

- GitHub Actions updates, including major and digest updates, auto-merge.
- Chart patch updates auto-merge; chart minor and major updates need manual review.
- Chart updates get a separate branch per major and minor version, capped below the next major version's
  `.1.0` release.

To change the dependency by hand, edit `Chart.yaml`, then regenerate `Chart.lock`, and commit both files together:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

## Local Development

To run the repository's GitHub workflows locally, start the `SteadOps-Steadies-K8s-Workplace` workbench, change
into this repository, and run:

```sh
 act
```

On the first run, `act` asks which image flavor to use. The default `medium` is a good starting point.
