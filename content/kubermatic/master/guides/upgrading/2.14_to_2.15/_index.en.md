+++
title = "Upgrading from 2.14 to 2.15"
date = 2020-06-09T11:09:15+02:00
weight = 90

+++

## Helm 3 Support

KKP now supports Helm 3. Previous versions required some manual intervention to get all charts
installed. With the updated CRD handling (see below), we made the switch to recommending Helm 3
for new installations.

A [migration guide]({{< ref "./helm_2to3" >}}) is provided.

## [EE] Deprecation of `kubermatic` Helm Chart

After the Kubermatic Operator has been introduced as a beta in version 2.14, it is now the recommended way of
installing and managing KKP. This means that the `kubermatic` Helm chart is considered deprecated as of
version 2.15 and all users are encouraged to prepare the migration to the Operator.

The Kubermatic Operator does not support previously deprecated features like the `datacenters.yaml`
or the full feature set of the `kubermatic` chart's customization options. Instead, datacenters
have to be converted to `Seed` resources, while the chart configuration must be converted to a
`KubermaticConfiguration`. The Kubermatic Installer offers commands to perform these conversions
automatically.

Note that the following customization options are not yet supported in the Kubermatic Operator:

* `maxParallelReconcile` (always defaults to `10`)
* Node and Pod affinities, node selectors for the KKP components
* Worker goroutine count for the KKP components

Depending on your chosen installation method, a number of upgrade paths are documented:

* [Upgrading the Operator from 2.14 to 2.15]({{< ref "./kubermatic_operator" >}})
* [Migration from Helm to the Operator]({{< ref "./chart_migration" >}})
* [Upgrading the legacy Helm Chart]({{< ref "./kubermatic_chart" >}}) (EE)

## Deprecation of `nodeport-proxy` Helm Chart

In conjunction with the operator being promoted to the recommended installation method, we also deprecate the
old `nodeport-proxy` Helm chart. The proxy is still a required component of any Kubermatic setup, but is now
managed by the Kubermatic Operator (similar to how it manages seed clusters).

The upgrade documents linked above include the necessary migration steps.

## Misc Helm Charts

### Prometheus

The Prometheus version included in Kubermatic Kubernetes Platform (KKP) 2.15 now enables WAL compression by default; our Helm chart follows this
recommendation. If the compression needs to stay disabled, the key `prometheus.tsdb.compressWAL` can be set to `false`
when upgrading the Helm chart.

### CRD Handling in cert-manager, Velero

In previous KKP releases, Helm was responsible for installing the CRDs for cert-manager and Velero. While this
made the deployment rather simple, it lead to problems in keeping the CRDs up-to-date (as Helm never updates or deletes
CRDs).

For this reason the CRD handling in KKP 2.15 was changed to require users to always manually install CRDs before
installing/updating a Helm chart. This provides much greater control over the CRD life cycle and eases integration with
other deployment mechanisms.

Upgrading existing Helm releases in a cluster is simple, as Helm does not delete CRDs. To update cert-manager, simply
install the CRDs and then run Helm as usual:

**Helm 3**

```bash
kubectl apply -f charts/cert-manager/crd/
helm --namespace cert-manager upgrade --install --values YOUR_VALUES_YAML_HERE cert-manager charts/cert-manager/
```

**Helm 2**

```bash
kubectl apply -f charts/cert-manager/crd/
helm --tiller-namespace kubermatic upgrade --install --values YOUR_VALUES_YAML_HERE --namespace cert-manager cert-manager charts/cert-manager/
```

{{% notice note %}}
Older versions of `kubectl` can potentially have issues applying the CRD manifests. If you encounter problems, please update
your `kubectl` to the latest stable version and refer to the
[cert-manager documentation](https://cert-manager.io/docs/installation/upgrading/upgrading-0.15-0.16/#issue-with-older-versions-of-kubectl)
for more information.
{{% /notice %}}

Note that on the first `kubectl apply` you will receive warnings because now "kubectl takes control over previously
Helm-owned resources", which can be safely ignored.

Perform the same steps for Velero:

**Helm 3**

```bash
kubectl apply -f charts/backup/velero/crd/
helm --namespace velero upgrade --install --values YOUR_VALUES_YAML_HERE velero charts/backup/velero/
```

**Helm 2**

```bash
kubectl apply -f charts/backup/velero/crd/
helm --tiller-namespace kubermatic upgrade --install --values YOUR_VALUES_YAML_HERE --namespace velero velero charts/backup/velero/
```

### Promtail

The labeling for the Promtail DaemonSet has changed, requiring administrators to re-install the Helm chart. As a clean
upgrade is not possible, we advise to delete and re-install the chart.

**Helm 3**

```bash
helm --namespace logging delete promtail
helm --namespace logging upgrade --install --values YOUR_VALUES_YAML_HERE promtail charts/logging/promtail/
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic delete promtail
helm --tiller-namespace kubermatic upgrade --install --values YOUR_VALUES_YAML_HERE --namespace logging promtail charts/logging/promtail/
```

Promtail pods are stateless, so no data is lost during this migration.

### Removed Default Credentials

To prevent insecure misconfigurations, the default credentials for Grafana and Minio have been removed. They must be
set explicitly when installing the charts. Additionally, the base64 encoding for Grafana credentials has been removed,
so the plain text values are put into the Helm `values.yaml`.

When upgrading the charts, make sure your `values.yaml` contains at least these keys:

```yaml
grafana:
  # Remember to un-base64-encode the username if you have set a custom value.
  user: admin

  # generate random password, keep it plaintext as well
  password: ExamplePassword

minio:
  credentials:
    accessKey: # generate a random, alphanumeric 32 byte secret
    secretKey: # generate a random, alphanumeric 64 byte secret
```

### Identity-Aware Proxy (IAP)

Previous KKP versions used Keycloak-Proxy for securing access to cluster services like Prometheus or Grafana.
The project was then renamed to [Louketo](https://github.com/louketo/louketo-proxy) and then shortly thereafter
[deprecated](https://github.com/louketo/louketo-proxy/issues/683) and users are encouraged to move to
[OAuth2-Proxy](https://github.com/oauth2-proxy/oauth2-proxy).

KKP 2.15 therefore switches to OAuth2-Proxy, which covers most of Keycloak's functionality but with a slightly
different syntax. Please refer to the [official documentation](https://github.com/oauth2-proxy/oauth2-proxy/blob/master/docs/configuration/configuration.md)
for the available settings, in addition to these changes:

* `iap.disocvery_url` in Helm values was renamed to `iap.oidc_issuer_url`, pointing to the OIDC provider's base
  URL (i.e. if you had this configured as `https://example.com/dex/.well-known/openid-configuration` before, this must
  now be `https://example.com/dex`).
* The `passthrough` option for each IAP deployment has been removed. Instead paths that are **not** supposed to be
  secured by the proxy are now configured via `config.skip_auth_regex`.
* The `config.scopes` option for each IAP deployment is now `config.scope`, a single string that must (for Dex)
  be space-separated.
* The `config.resources` mechanism for granting access based on user groups/roles has been removed. Instead the
  required organizations/teams are now configured via explicit config variables like `config.github_org` and
  `config.github_team`.
* `email_domains` must be configured for each IAP deployment. In most cases it can be set to `["*"]`.

A few examples can be found in the relevant [code change in KKP](https://github.com/kubermatic/kubermatic/pull/5777/files).

To prevent issues with Helm re-using IAP deployment config values from a previous release, it can be helpful to purge and
reinstall the chart:

**Helm 3**

```bash
helm --namespace iap delete iap
helm --namespace iap upgrade --install --values YOUR_VALUES_YAML_HERE iap charts/iap/
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic delete iap
helm --tiller-namespace kubermatic upgrade --install --values YOUR_VALUES_YAML_HERE --namespace iap iap charts/iap/
```

Be advised that during the re-installation the secured services (Prometheus, Grafana, ...) will not be accessible.
