+++
title = "Upgrading KKP Operator"
date = 2021-04-22T17:33:39+02:00
weight = 30

+++

Upgrading a KKP setup that is already managed by the Operator is as simple as
updating the Helm charts and following the [general upgrade notes]({{< ref "../" >}}).

## Upgrade Procedure

Download the [latest 2.17 release](https://github.com/kubermatic/kubermatic/releases) from GitHub
(make sure to choose the right version, CE for the Community or EE for the Enterprise Edition) and
extract the archive locally.

```bash
wget https://github.com/kubermatic/kubermatic/releases/download/v2.17.0/kubermatic-ee-v2.17.0-linux-amd64.tar.gz
tar -xzvf kubermatic-ce-v2.17.0-linux-amd64.tar.gz
```

Run the installer to perform the upgrade using the `kubermatic.yaml` and `values.yaml` you used to create the platform.

```bash
./kubermatic-installer deploy --config kubermatic.yaml --helm-values values.yaml --migrate-cert-manager
```

Note that the `--migrate-cert-manager` flag is necessary for the migration to cert-manager `1.2.0` to succeed.
It will trigger an automatic migration of all `Certificate` resources from
version `v1alpha1` to `v1` (find more details [here](https://github.com/kubermatic/kubermatic/pull/6739)).


The Operator will automatically update the KKP Seed Controller Manager on every seed cluster.
Manually upgrade all other charts you might have installed as part of the monitoring or logging
stacks.

## Migrating from the `nodeport-proxy` Helm Chart

{{% notice info %}}
The operator installs the `nodeport-proxy` by default. This step is required only
for installations where `nodeport-proxy` was originally installed by Helm chart
and it was not migrated yet as part of the migration to the operator.
{{% /notice %}}

The `nodeport-proxy` Helm chart has been deprecated in 2.15, the proxy however is still a required component
of any Kubermatic setup, but is now managed by the Kubermatic Operator (similar to how it manages seed clusters).

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
nodeport-proxy in place for a period of time (e.g. a few days to be
conservative), before finally removing it:

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
