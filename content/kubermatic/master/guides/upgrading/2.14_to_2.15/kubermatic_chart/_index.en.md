+++
title = "Upgrading Helm Chart (EE)"
date = 2020-06-09T11:09:15+02:00
weight = 20

+++

In case a [migration to the Operator]({{< ref "../chart_migration" >}}) is not possible, it's still
supported to upgrade the `kubermatic` Helm chart for 2.15.

## Upgrade Procedure

Download the [latest 2.15 release](https://github.com/kubermatic/kubermatic/releases) from GitHub
(make sure to choose the EE version) and extract the archive locally.

```bash
wget https://github.com/kubermatic/kubermatic/releases/download/v2.15.0/kubermatic-ee-v2.15.0-linux-amd64.tar.gz
tar -xzvf kubermatic-ee-v2.15.0-linux-amd64.tar.gz
```

Update the KKP and cert-manager CRDs **on the master and all seed clusters**:

```bash
kubectl apply -f charts/cert-manager/crd/
kubectl apply -f charts/kubermatic/crd/
```

Then use your Helm `values.yaml` and upgrade the releases in your master cluster:

**Helm 3**

```bash
helm --namespace nginx-ingress-controller upgrade --install --values myvalues.yaml nginx-ingress-controller charts/nginx-ingress-controller/
helm --namespace cert-manager upgrade --install --values myvalues.yaml cert-manager charts/cert-manager/
helm --namespace oauth upgrade --install --values myvalues.yaml oauth charts/oauth/
helm --namespace kubermatic upgrade --install --values myvalues.yaml kubermatic charts/kubermatic/
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic upgrade --install --values myvalues.yaml --namespace nginx-ingress-controller nginx-ingress-controller charts/nginx-ingress-controller/
helm --tiller-namespace kubermatic upgrade --install --values myvalues.yaml --namespace cert-manager cert-manager charts/cert-manager/
helm --tiller-namespace kubermatic upgrade --install --values myvalues.yaml --namespace oauth oauth charts/oauth/
helm --tiller-namespace kubermatic upgrade --install --values myvalues.yaml --namespace kubermatic kubermatic charts/kubermatic/
```

Once the master cluster is updated, update the `kubermatic` and `nodeport-proxy` chart on all seed clusters
as well. Remember to set `isMaster` to `false` in the `values.yaml` for your seed clusters.

**Helm 3**

```bash
helm --namespace nodeport-proxy upgrade --install --values myvalues.yaml nodeport-proxy charts/nodeport-proxy/
helm --namespace kubermatic upgrade --install --values myvalues.yaml kubermatic charts/kubermatic/
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic upgrade --install --values myvalues.yaml --namespace nodeport-proxy nodeport-proxy charts/nodeport-proxy/
helm --tiller-namespace kubermatic upgrade --install --values myvalues.yaml --namespace kubermatic kubermatic charts/kubermatic/
```

Afterwards, manually upgrade all other charts you might have installed as part of the monitoring or logging
stacks.
