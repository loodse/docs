+++
title = "Migrating to the Operator"
date = 2020-06-09T11:09:15+02:00
weight = 30

+++

This chapter outlines a migration plan for the Kubermatic Operator, starting from having KKP installed
using the old `kubermatic` Helm chart using Helm 2.x.

{{% notice warning %}}
Before performing the migration, make sure to back up all Kubermatic resources!
{{% /notice %}}

Make sure that you also have recent etcd snapshots for all user clusters.

#### Overview

A parallel installation of both the old and the new `kubermatic-operator` chart is not possible,
regardless of the namespaces. Only one KKP setup can be live in a given cluster at any time.

Because of this, it is recommended to perform an uninstall of the old chart followed by installing
the new Operator chart. This will lead to some downtime, both in the dashboard as well as all
cluster reconciliation logic.

{{% notice warning %}}
**It is of critical importance that the Kubermatic CRDs are not deleted during the migration.**
This would in turn delete all affected resources, like clusters, users etc.
KKP never used Helm to manage its CRDs (Helm was only previously used for the cert-manager and
Velero CRDs), so it is safe to uninstall the chart.
{{% /notice %}}

Compared to the old installation method, were the `kubermatic` chart was installed both on the
master and all seed clusters (toggling the `isMaster` flag in the chart accordingly), with the
operator seeds are now managed automatically. The operator is only ever installed once in the
master cluster and manages the seed clusters from there. Still, care must be taken to not
accidentally have both Helm chart and operator-managed resources in seed clusters. For this
reasons, the reconciliation of seed clusters (installing the seed-controller-manager Deployment,
the Seed validation webhook etc.) can be disabled by setting the
`operator.kubermatic.io/skip-reconciling` annotation on Seed resources.

All in the all, a migration plan can look like this:

1. Perform a backup or all Kubermatic resources.
1. Ensure etcd snapshots for all user clusters are in place.
1. Convert Helm values / `datacenters.yaml` to KubermaticConfiguration, Seeds, Presets etc.
   using the Kubermatic Installer. Make sure to create Seeds that are *paused* (have the
   `operator.kubermatic.io/skip-reconciling` annotation set; this can be achieved by running
   the installer with the `--pause-seeds` flag).
1. Uninstall `kubermatic` Helm chart from the master cluster. **Leave the CRDs in place.**
1. Install the `kubermatic-operator` Helm chart using Helm 3.
1. Wait for the master cluster to become ready.
1. Start migrating Seeds:
   1. Pick a Seed
   1. Uninstall `kubermatic` Helm chart from the seed cluster.
   1. Update the `Seed` resource in the master and remove the `operator.kubermatic.io/skip-reconciling` annotation.
   1. Wait for the operator to reconcile the seed cluster.
   1. Continue with the next seed cluster.

{{% notice info %}}
This poses a good opportunity to also make the switch to Helm 3. All other charts for the KKP
stack (cert-manager, nginx-ingress-controller, etc.) can be migrated in-place from Helm 2 to 3.
{{% /notice %}}

The following sections describe the steps above in greater detail.

#### Backups

Backing up Kubermatic resources can be done in many ways, for example simply by using `kubectl`
and writing the YAML into local files:

```bash
kubectl get addons -A -o yaml > addons.yaml
kubectl get addonconfigs -A -o yaml > addonconfigs.yaml
kubectl get admissionplugins -A -o yaml > admissionplugins.yaml
kubectl get clusters -A -o yaml > clusters.yaml
kubectl get kubermaticsettings -A -o yaml > kubermaticsettings.yaml
kubectl get presets -A -o yaml > presets.yaml
kubectl get projects -A -o yaml > projects.yaml
kubectl get seeds -A -o yaml > seeds.yaml
kubectl get users -A -o yaml > users.yaml
kubectl get userprojectbindings -A -o yaml > userprojectbindings.yaml
kubectl get usersshkeies -A -o yaml > usersshkeies.yaml
kubectl get verticalpodautoscalers -A -o yaml > verticalpodautoscalers.yaml
```

Administrators should also check each seed cluster's Minio to verify that etcd snapshots for
all user clusters are in place.

#### Convert `datacenters.yaml` / Helm Values

The Kubermatic Installer provides commands to automatically convert the old configuration files
into their new formats.

The `convert-helm-values` command takes a Helm values.yaml file and outputs (to stdout) a number
of YAML files, representing the generated KubermaticConfiguration, Seeds, Secrets, Presets etc.
Using this command is recommended, as it does all conversions in a single step, because the old
`values.yaml` contained all information in some form or another.

Note that the conversion commands by default outputs Seed resources with the
`operator.kubermatic.io/skip-reconciling` annotation already in place, so the Seeds are safe to
apply during a migration.

```bash
./kubermatic-installer convert-helm-values myvalues.yaml
# apiVersion: operator.kubermatic.io/v1alpha1
# kind: KubermaticConfiguration
# metadata:
#   name: kubermatic
#   namespace: kubermatic
# ...
#
# ---
# apiVersion: kubermatic.k8s.io/v1
# kind: Seed
# ...
#
# ---
# ...
```

In case migrating the Helm values.yaml is not feasible, the installer also offers a dedicated
command to convert just the `datacenters.yaml` into Seeds/Secrets. By default, the `datacenters.yaml`
did not contain kubeconfigs, so you have to manually provide a kubeconfig with contexts for every
seed cluster in the `datacenters.yaml` (the installer will error out if contexts are missing).
If you do not provide a kubeconfig, Seeds will still be converted (and paused by default), but
you are responsible yourself to provide the appropriate kubeconfig Secrets and to reference them
in the Seeds.

```bash
./kubermatic-installer convert-datacenters --kubeconfig kubeconfig-with-all-seeds datacenters.yaml
# apiVersion: kubermatic.k8s.io/v1
# kind: Seed
# ...
#
# ---
# ...
```

Refer to the [KubermaticConfiguration]({{< ref "../../../../architecture/concepts/configuration" >}}) and
[Seed]({{< ref "../../../../architecture/concepts/seeds" >}}) documentation for more information.

Once the conversion is completed, carefully check the new configuration files for mistakes. Do
note that the installer generally does not output default values, so if you configured for example
2 Apiserver replicas and this is the default for Kubermatic anyway, the generated `KubermaticConfiguration`
will skip the value altogether, relying on the defaulting.

It is now recommended to setup Kubermatic in a test environment, using the newly converted files.

#### Upgrading the Master Cluster

It's now time to change the Helm charts. Begin by uninstalling the `kubermatic` chart. This will make
the dashboard unavailable and begin the downtime.

**Helm 3**

```bash
helm --namespace kubermatic delete kubermatic
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic delete --purge kubermatic
```

Once the Helm chart is gone, it's time to perform a clean installation of the new Kubermatic Operator.
The first step for that is to add new Kubermatic CRDs:

```bash
kubectl apply -f charts/kubematic/crd/
```

To prevent Kubermatic from rejecting existing clusters, make sure to apply the Seeds before installing
the Operator. We assume that `seeds-with-secrets.yaml` is part of the output of one of the conversion
commands described above. Likewise, `kubermaticconfiguration.yaml` is part of the conversion output
from above.

```bash
kubectl apply -f seeds-with-secrets.yaml -f kubermaticconfiguration.yaml
```

The output from the installer command can also be applied directly, though we recommend reviewing the
output to prevent surprises.


```bash
./kubermatic-installer convert-helm-values --pause-seeds myvalues.yaml | kubectl apply -f -

# or
./kubermatic-installer convert-datacenters --pause-seeds --kubeconfig kubeconfig-with-all-seeds datacenters.yaml | kubectl apply -f -
```

Check that all seeds are in place now:

```bash
kubectl -n kubermatic get seeds
```

Now it's time to install the operator. Make sure to configure the `kubermatic-operator` chart properly,
taking care of setting the correct ImagePullSecret in the `values.yaml`. Once you're satisfied, install
the operator:

**Helm 3**

```bash
helm -n kubermatic install --values myvalues.yaml kubermatic-operator charts/kubermatic-operator/
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic install --values myvalues.yaml --namespace kubermatic kubermatic-operator charts/kubermatic-operator/
```

Once the operator is up, it will begin reconciling the KKP Master and Seed clusters. It will now take a
few minutes for the new installation to become ready.

#### Migrate Seeds

Due to the `operator.kubermatic.io/skip-reconciling` annotation on Seeds, the operator will not provision them.
However Kubermatic is still seeing the Seed and wants to use it, so the migration of seeds should now be
completed relatively quickly.

To migrate a seed, first uninstall the `kubermatic` Helm chart from it. As with the master, **do not delete the CRDs.**

**Helm 3**

```bash
helm --namespace kubermatic delete kubermatic
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic delete --purge kubermatic
```

Like we did on the master, apply the updated CRDs now:

```bash
kubectl apply -f charts/kubematic/crd/
```

The seed cluster is now ready to be managed by the operator. On the master cluster, update the Seed resource
for the seed cluster and remove the `operator.kubermatic.io/skip-reconciling` annotation:

```
kubectl annotate seed your-seed operator.kubermatic.io/skip-reconciling-
```

Once the Seed is updated, the operator will start to reconcile the seed cluster. It is recommended to follow the
operator's logs to spot any problems.

After all seed clusters have been migrated, the operator migration is finished and for health and safety reasons,
everyone involved should take a break.

## Migrating from the `nodeport-proxy` Helm Chart

In conjunction with the operator being promoted to the recommended installation method, we also deprecate the
old `nodeport-proxy` Helm chart. The proxy is still a required component of any Kubermatic setup, but is now
managed by the Kubermatic Operator (similar to how it manages seed clusters).

The migration to the operator-managed nodeport-proxy is relatively simple. The operator by default creates the
new nodeport-proxy inside the Kubermatic namespace (`kubermatic` by default), whereas the old proxy was
living in the `nodeport-proxy` namespace. Due to this, no naming conflicts can occur and in fact, both proxies
can co-exist in the same cluster.

The only important aspect is where the DNS record for the seed cluster is pointing. To migrate from the old
to new nodeport-proxy, all that needs to be done is switch the DNS record to the new LoadBalancer service. The
new services uses the same ports, so it does not matter what service a user is reaching.

To migrate, find the new LoadBalancer service's public endpoint:

```bash
kubectl -n kubermatic get svc
#NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                                          AGE
#kubermatic-api         NodePort       10.47.248.61    <none>          80:32486/TCP,8085:32223/TCP                      216d
#kubermatic-dashboard   NodePort       10.47.247.32    <none>          80:32382/TCP                                     128d
#kubermatic-ui          NodePort       10.47.240.175   <none>          80:31585/TCP                                     216d
#nodeport-proxy         LoadBalancer   10.47.254.72    34.89.181.151   32180:32428/TCP,30168:30535/TCP,8002:30791/TCP   182d
#seed-webhook           ClusterIP      10.47.249.0     <none>          443/TCP                                          216d
```

Take the `nodeport-proxy`'s EXTERNAL IP, in this case `34.89.181.151`, and update your DNS record for
`*.<seedname>.kubermatic.example.com` to point to this new IP.

It will take some time for the DNS changes to propagate to every user, so it is recommended to leave the old
nodeport-proxy in place for a period of time (e.g. a week), before finally removing it:

**Helm 3**

```bash
helm --namespace nodeport-proxy delete nodeport-proxy
kubectl delete ns nodeport-proxy
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic delete --purge nodeport-proxy
kubectl delete ns nodeport-proxy
```

These steps need to be performed on all seed clusters.
