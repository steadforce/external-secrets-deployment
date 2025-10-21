# Deployment for external-secrets

Repository containing helm chart for external-secrets installation.
Never install the content of this repo on our clusters manually. This is all done by argocd.

## Dependencies

This chart refers `external-secrets` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

## Installation

We use AWS Systems Manager's Parameter Store to externally store secrets.
There are two AWS accounts for SteadOps (development and production) and two AWS IAM users with access to these accounts.
The users have restricted permissions that only allows access to the Parameter Store.
The credentials for these users are stored in the internal password database.

These two secrets have to be passed into the helm chart with parameter keys:

```
aws.accessKeyId
aws.secretAccessKey
```

You have to enable the secret generation with the parameter `bootstrapResources.enabled`, since
it only needs to be created on bootstrapping the external secrets operator.

From the `aws.accessKeyId` and the `aws.secretAccessKey` the `awssm-secret` is created. 
This secret is referenced in `awssm-parameter-store` and
makes it possible to get external secrets out of the AWS parameter store.

For details, look at [external secrets aws parameter store](https://external-secrets.io/latest/provider/aws-parameter-store/)
documentation.

## Update awssm-secret

If an update for the aws-secret is needed, the chart supports re-rendering and update by executing the
following command.

```shell
 NS=$(yq 'explode(.) | .namespace // ""' helm-config.yaml) \
 helm dependency update && \
 helm template \
   --release-name $(yq 'explode(.) | .releaseName // ""' helm-config.yaml) \
   --set aws.accessKeyId="<-- new aws key id for the stage -->" \
   --set aws.secretAccessKey="<--new aws secret access key for the stage -->" \
   --set bootstrapResources.enabled=true \
   --skip-tests \
   -n "$NS" \
   -s templates/awssm-secret.yaml \
   . | \
    kubectl -n "$NS" apply -f -
```

# Testing

## Usage of values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the subchart(s) used by this chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

## Run helm unittests

```shell
 docker run -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Render all manifests locally

```shell
 helm dependency update && \
 for cluster in $(yq '.environments | keys[]' helm-config.yaml); do
    helm template \
      -a "$(cluster=$cluster yq '.environments.[env(cluster)].apis | @csv' helm-config.yaml)" \
      -f "$(cluster=$cluster yq '.environments.[env(cluster)].valueFiles | @csv' helm-config.yaml)" \
      -n $(yq 'explode(.) | .namespace // ""' helm-config.yaml) \
      --output-dir _local/$cluster \
      --include-crds \
      --release-name $(yq 'explode(.) | .releaseName // ""' helm-config.yaml) \
      --skip-tests \
      .
 done
```

## Run GitHub workflows locally

To run all workflows in a local environment, start up the workbench, cd into the folder containing this
`README.md` and execute the following command:

```shell
  act
```

On first execution you're asked which flavour of the act image should be used. Using the default `medium`
is a good starting point.
