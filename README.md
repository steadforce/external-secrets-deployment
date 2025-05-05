# external-secrets for steadforce SteadOps k8s clusters

Repository containing helm chart for external-secrets installation. Never install the content of this repo on our clusters manually. This is all done by argocd.

## Dependencies

This chart pulls in `external-secrets` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

```
 helm dependency update
```

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

## Installation

Currently we use AWS Parameter Store to store the secrets externally. There are two accounts for
SteadOps with only access rights to AWS Systems Manager Parameter Store. The AWS access key and secret
for these accounts are stored in the internal password database.

These two secrets have to be passed into the helm chart with parameter keys:

```
aws.accessKeyId
aws.secretAccessKey
```

Therefrom the `awssm-secret` is created. This secret is referenced from `awssm-parameter-store` and
makes it possible to get external secrets out of the aws parameter store.

For details look at [external secrets aws parameter store](https://external-secrets.io/latest/provider/aws-parameter-store/)
documentation.

## Testing

### values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the subchart(s) used by this chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

### run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Render resource local

### local

```
 helm template \
  --include-crds \
  --output-dir _local/local \
  --release-name external-secrets \
  --skip-tests \
  -a external-secrets.io/v1beta1/ClusterSecretStore \
  -f values-subchart-overrides.yaml \
  -f values-local.yaml \
  -n external-secrets \
  .
```

### development

```
 helm template \
  --include-crds \
  --output-dir _local/dev \
  --release-name external-secrets \
  --skip-tests \
  -a external-secrets.io/v1beta1/ClusterSecretStore \
  -f values-subchart-overrides.yaml \
  -f values-development.yaml \
  -n external-secrets \
  .
```

### production

```
 helm template \
  --include-crds \
  --output-dir _local/prod \
  --release-name external-secrets \
  --skip-tests \
  -a external-secrets.io/v1beta1/ClusterSecretStore \
  -f values-subchart-overrides.yaml \
  -f values-production.yaml \
  -n external-secrets \
  .
```
