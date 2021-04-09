+++
title = "Upgrading from 2.10 to 2.11"
date = 2019-05-03T12:07:15+02:00
publishDate = 2019-07-12T00:00:00+00:00
weight = 50

+++

## Expose Strategy

Kubermatic Kubernetes Platform (KKP) 2.11 adds support to expose user clusters by creating one service of type `LoadBalancer` per user
cluster. Check out the [Expose Strategy documentation]({{< ref "../../concepts/expose-strategy" >}}) for more details.

## Helm Charts

### KKP: Service Account Tokens Structure

A new flag `service-account-signing-key` was added to the KKP API. It is used to sign service account tokens via
HMAC. It should be unique per KKP installation and can be generated with the command: `base64 -w0 /dev/urandom | head -c 100`
The value for this flag must be stored in `auth` section for `kubermatic`

For example:

```yaml
kubermatic:
  auth:
    serviceAccountKey: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

### Elasticsearch 7.0

KKP 2.11 ships with Elasticsearch 7.0. Version 7 can do rolling upgrades only if the Elasticsearch cluster is
already running on version 6.7 (the version shipped in KKP 2.10).

If you are already running KKP 2.10, you can simply deploy the updated Helm chart and the nodes will update one
by one to Elasticsearch 7.

If you are running an older KKP release, you must either upgrade to KKP 2.10 first or shutdown the entire
Elasticsearch cluster before doing the upgrade (e.g. by purging the chart before installing the new chart).

{{% notice warning %}}
Note that because Elasticsearch 7.0 changed its database fields, you cannot downgrade to any previous release after
the upgrade.
{{% /notice %}}

The backwards compatibility for the renamed configuration keys (`storageSize`, `replicas` etc.) introduced in KKP
2.10 has been removed, so make sure to update your `values.yaml` to use the new config keys as documented in the 2.10
release notes.

### Prometheus & Alertmanager

Both charts have had their naming scheme refactored in order to seamlessly install multiple copies along each other. This
lead to the hostnames and PersistentVolumeClaim names changing.

* `prometheus-kubermatic` service becomes `prometheus`.
* `alertmanager-kubermatic` service becomes `alertmanager`, but a service with the old name is installed to give user
  clusters time to reconcile. The fallback service will be removed in a later bugfix release.

In order to migrate existing data you must deploy both charts with the `migration.enabled` flag set to `true`:

```yaml
prometheus:
  migration:
    enabled: true

alertmanager:
  migration:
    enabled: true
```

This will update the resources and deploy a one-time job to copy the existing data into the new volumes.

* `prometheus-kubermatic-db-prometheus-kubermatic-N` becomes `db-prometheus-N`.
* `alertmanager-kubermatic-db-alertmanager-kubermatic-N` becomes `db-alertmanager-N`.

Wait until the job's pod has completed and then deploy the charts again, this time disabling the migration again by
removing the `migration` configuration key from your `values.yaml`. This will remove the one-time job and instead
deploy the new StatefulSet.

Once you have verified that both applications work, you can safely remove the old PVCs:

```bash
kubectl -n monitoring delete pvc \
  prometheus-kubermatic-db-prometheus-kubermatic-0 \
  prometheus-kubermatic-db-prometheus-kubermatic-1 \
  alertmanager-kubermatic-db-alertmanager-kubermatic-0 \
  alertmanager-kubermatic-db-alertmanager-kubermatic-1 \
  alertmanager-kubermatic-db-alertmanager-kubermatic-2
```

#### Alertmanager 0.17

Alertmanager 0.17 changed the included `amtool` to be incompatible with previous versions. This only affects you
if you ever exported silences and wanted to re-import them. Please consult the
[release notes](https://github.com/prometheus/alertmanager/releases/tag/v0.17.0) for more information.

The `alertmanager.version` field in the Chart's `values.yaml` was deprecated and replaced with the more common
`alertmanager.image.tag` field. Update your Helm values if you have ever set the Alertmanager's version explicitly.
The `migrate-values` command of the KKP-Installer can automate this migration step for you.

Likewise, the `alertmanager.resources.storage` has been renamed to `alertmanager.storageSize` and the old key has
been deprecated. Please update your `values.yaml` if you've ever increased the volume size for Alertmanager.

### NodePort-Proxy

The top-level key in the `values.yaml` has been fixed and is now called `nodePortProxy`. Please update your Helm
values as the old name (`nodePortPoxy`) is now deprecated.

### cert-manager 0.8

KKP 2.11 ships with cert-manager version 0.8, which slightly changed how certificates and issuers are
configured. The 0.8 release is backwards compatible, but this is going to be removed until version 1.0. It's
recommended that KKP customers upgrade their certificates already to be future-proof.

Please consult the [upgrade notes](https://docs.cert-manager.io/en/release-0.8/tasks/upgrading/upgrading-0.7-0.8.html)
for more information. In most cases it should be enough to simply remove the `spec.acme` field from all
`Certificate` resources. As noted in the documentation, you can use `kubectl get certificates --all-namespaces`
to see which are still using the pre-0.8 syntax.

{{% notice warning %}}
**Action required:** Because Helm cannot update CRDs, you must manually update the CRDs for this release by running
`kubectl apply -f config/cert-manager/templates/crd.yaml`. Failure to do so will break certificate renewal because
of spec validation failures. If you notice that the cert-manager is logging that orders are invalid, make sure the
CRDs are up-to-date and you restarted the cert-manager pods.
{{% /notice %}}
