# external-secrets for steadforce k8s clusters

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

## Render resource local

### local

```
 helm template \
  --include-crds \
  --output-dir _local/local \
  --release-name external-secrets \
  --skip-tests \
  -a generators.external-secrets.io/v1alpha1/GithubAccessToken \
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
  -a generators.external-secrets.io/v1alpha1/GithubAccessToken \
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
  -a generators.external-secrets.io/v1alpha1/GithubAccessToken \
  -f values-subchart-overrides.yaml \
  -f values-production.yaml \
  -n external-secrets \
  .
```
