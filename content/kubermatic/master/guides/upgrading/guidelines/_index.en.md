+++
title = "General Guidelines"
date = 2020-10-08T11:09:15+02:00
weight = 1

+++

This section describes the recommended steps to be taken for every KKP upgrade,
including from patch release to patch release.

## Create Backups

Even though regular backups should already be in place, it's always good to be
on the safe side and ensure that backups are made prior to upgrades.

The Velero chart, if installed, includes a backup job that copies all resources
(Clusters, Seeds, Users, Projects, ...) on a regular basis. Another option is
to simply use `kubectl` to dump all KKP resources:

```bash
export KUBECONFIG=...

while IFS= read -r crd; do
  echo "Dumping $crd ..."
  kubectl get $crd -A -o yaml > $crd.yaml
done <<< "$(kubectl get crd | grep kubermatic | cut -f1 -d' ')"
```

Backups must be done on the master and should be done on all seed cluster as
well.

## Upgrade Versioning Policy

KKP only supports upgrades from one minor version to the next (i.e. from 2.13 to 2.14).
You must not omit a minor version when doing an upgrade of your KKP installation
(i.e. 2.13 to 2.15 without upgrading to 2.14 in-between is not supported).

## Prepare for Reconciliation Load

Whenever KKP is upgraded, changes that affect possibly many cluster control-planes
can be rolled out. For example when new settings on Deployments are set, Docker
image versions or registries change, etc.

These mass-upgrades can cause severe load on the seed clusters, depending on the
number of user clusters. To prevent overloads, KKP has a mechanism to limit the
number of parallel reconciliations that can be active at any time,
`MaximumParallelReconciles`. This defaults to `10` and can be tweaked by editing
the `KubermaticConfiguration` and setting `spec.seedController.maximumParallelReconciles`
accordingly. In the legacy `kubermatic` Helm chart, the setting can be overridden
via `kubermatic.maxParallelReconcile`.

It is generally good practice to lower the limit prior to performing an upgrade,
observing the seed cluster load afterwards and then resetting it again.

## Manual Upgrade Procedure

Unless otherwise noted in the upgrade notes for the given KKP minor version, the
upgrade procedure is roughly as follows. The exact paths may vary slightly
in-between minor releases, but the procedure is the same.

For KKP releases prior to 2.14, refer to https://github.com/kubermatic/kubermatic-installer
to download the Helm charts and configuration files.

For KKP releases since 2.14, download the appropriate archive from
https://github.com/kubermatic/kubermatic/releases and extract it locally on your
computer.

It's now time to perform any manual upgrade steps and update the Helm `values.yaml`
file used to install the charts (a single file for all charts is recommended, but
one `values.yaml` per chart is also possible).

Before updating the Helm charts, update all CRDs:

```bash
kubectl apply -f charts/cert-manager/crd/
kubectl apply -f charts/backup/velero/crd/
kubectl apply -f charts/kubermatic/crd/
```

Unless noted otherwise, if the `KubermaticConfiguration` or `Seed` configurations
had to be changed, they should now be updated.

In general, charts can be upgraded in any order, as long as eventually all
charts have been upgraded. Depending on your installation method, install the
`kubermatic` Helm chart for legacy setups or the `kubermatic-installer` chart for
the new KKP Operator.

**Helm 3**

```bash
helm --namespace cert-manager upgrade --install --wait --values values.yaml cert-manager charts/cert-manager/
helm --namespace nginx-ingress-controller upgrade --install --wait --values values.yaml nginx-ingress-controller charts/nginx-ingress-controller/
helm --namespace oauth upgrade --install --wait --values values.yaml oauth charts/oauth/

# either
helm --namespace kubermatic upgrade --install --wait --values values.yaml kubermatic-operator charts/kubermatic-operator/

# or
helm --namespace kubermatic upgrade --install --wait --values values.yaml kubermatic charts/kubermatic/
```

**Helm 2**

```bash
helm upgrade --install --wait --values values.yaml --namespace cert-manager cert-manager charts/cert-manager/
helm upgrade --install --wait --values values.yaml --namespace nginx-ingress-controller nginx-ingress-controller charts/nginx-ingress-controller/
helm upgrade --install --wait --values values.yaml --namespace oauth oauth charts/oauth/

# either
helm upgrade --install --wait --values values.yaml --namespace kubermatic kubermatic-operator charts/kubermatic-operator/

# or
helm upgrade --install --wait --values values.yaml --namespace kubermatic kubermatic charts/kubermatic/
```
