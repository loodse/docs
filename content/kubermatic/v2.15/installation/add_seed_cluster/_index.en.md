+++
title = "Add Seed Cluster for CE"
date = 2018-08-09T12:07:15+02:00
weight = 40
+++

This document describes how a new seed cluster can be added to an existing KKP master cluster.

{{% notice note %}}
For smaller scale setups it's possible to use the existing master cluster as a seed cluster (a "shared"
cluster installation). In this case both master and seed components will run on the same cluster and in
the same namespace. It is however not possible to use the same cluster for multiple seeds.
{{% /notice %}}

Please refer to the [architecture]({{< ref "../../concepts/architecture" >}}) diagrams for more information
about the cluster relationships.

## Install KKP Dependencies

Compared to master clusters, seed clusters are still mostly manually installed. Future versions of KKP
will improve the setup experience further.

When using Helm 2, install Tiller into the seed cluster first:

```bash
kubectl create namespace kubermatic
kubectl create serviceaccount -n kubermatic tiller
kubectl create clusterrolebinding tiller-cluster-role --clusterrole=cluster-admin --serviceaccount=kubermatic:tiller
helm --service-account tiller --tiller-namespace kubermatic init
```

### Cluster Backups

KKP performs regular backups of user cluster by snapshotting the etcd of each cluster. By default these backups
are stored locally inside the seed cluster, but they can be reconfigured to work with any S3-compatible storage.
The in-cluster storage is provided by [Minio](https://min.io/) and the accompanying `minio` Helm chart.

If your cluster has no default storage class, it's required to configure a class explicitly for Minio. You can check
the cluster's storage classes via:

```bash
kubectl get storageclasses
#NAME                 PROVISIONER              AGE
#kubermatic-fast      kubernetes.io/aws-ebs   195d
#kubermatic-backup    kubernetes.io/aws-ebs   195d
#standard (default)   kubernetes.io/aws-ebs   2y43d
```

As Minio does not require any of the SSD's advantages, we can use HDDs. It's recommended to create a separate storage class `kubermatic-backup` with a different location/security level. For a cluster running on AWS, an example class could look like this:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: kubermatic-backup
provisioner: kubernetes.io/aws-ebs
parameters:
  type: sc1
```

To configure the storage class and size, extend your `values.yaml`. For more information about the Minio options, take a look at [minio chart `values.yaml`](https://github.com/kubermatic/kubermatic/blob/master/charts/minio/values.yaml) and the [min.io documentation - S3 gateway](https://docs.min.io/docs/minio-gateway-for-s3.html):

```yaml
minio:
  storeSize: '200Gi'
  # SC will store the etcd backup of the seed hosted user clusters
  storageClass: kubermatic-backup
  # access key/secret for the exposed minio S3 gateway
  credentials:
    # generated access key length should be at least 3 characters
    accessKey: "YOUR-ACCESS-KEY"
    # generated secret key length should be at least 8 characters
    secretKey: "YOUR-SECRET-KEY"
```

It's also advisable to install the `s3-exporter` Helm chart, as it provides basic metrics about user cluster backups.

### Install Charts

With this you can install the charts:

**Helm 3**

```bash
helm --namespace minio upgrade --install --wait --values /path/to/your/helm-values.yaml minio charts/minio/
helm --namespace kube-system upgrade --install --wait --values /path/to/your/helm-values.yaml s3-exporter charts/s3-exporter/
```

**Helm 2**

```bash
helm --tiller-namespace kubermatic upgrade --install --values /path/to/your/helm-values.yaml --namespace minio minio charts/minio/
helm --tiller-namespace kubermatic upgrade --install --values /path/to/your/helm-values.yaml --namespace kube-system s3-exporter charts/s3-exporter/
```
## Add the Seed Resource

{{% notice warning %}}
For not combined master/seed cluster setup, please use the [Add Seed Cluster for EE]({{< ref "../add_seed_cluster_ee" >}}) docs!
{{% /notice %}}

To connect the new seed cluster with the master, you need to create a kubeconfig Secret and a Seed resource. This allows
the KKP components in the master cluster to communicate with the seed cluster and reconcile user-cluster control planes.

{{% notice warning %}}
To make sure that the kubeconfig stays valid forever, it must not contain temporary login tokens. Depending on the
cloud provider, the default kubeconfig that is provided may not contain username+password / a client certificate, but instead
try to talk to local token helper programs like `aws-iam-authenticator` for AWS or `gcloud` for the Google Cloud (GKE).
These kubeconfig files **will not work** for setting up Seeds.
{{% /notice %}}

The Kubermatic repository provides a [script](https://github.com/kubermatic-labs/community-components/blob/master/helper/kubeconfig-kubermatic-serviceaccount.sh) that can be used to prepare a kubeconfig for usage in Kubermatic. The script will create
a ServiceAccount in the seed cluster, bind it to the `cluster-admin` role and then put the ServiceAccount's token into
the kubeconfig file. Afterwards the file can be used in KKP.

The Seed resource itself needs to be called `kubermatic` (for the Community Edition) and needs to reference the new
kubeconfig Secret like so:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kubeconfig-kubermatic
  namespace: kubermatic
type: Opaque
data:
  # You can use `base64 -w0 my-kubeconfig-file` to encode the
  # kubeconfig properly for inserting into this Secret.
  kubeconfig: <base64 encoded kubeconfig>

---
apiVersion: kubermatic.k8s.io/v1
kind: Seed
metadata:
  # The Seed *must* be named "kubermatic".
  name: kubermatic
  namespace: kubermatic
spec:
  # these two fields are only informational
  country: DE
  location: Hamburg

  # list of datacenters where this seed cluster is allowed to create clusters in
  datacenters: []

  # reference to the kubeconfig to use when connecting to this seed cluster
  kubeconfig:
    name: kubeconfig-kubermatic
    namespace: kubermatic
```

Refer to the [Seed CRD documentation]({{< ref "../../concepts/seeds" >}}) for a complete example of the
Seed CustomResource and all possible datacenters.

You can override the global [Expose Strategy]({{< ref "../../concepts/expose-strategy">}}) at
Seed level if you wish to.

Apply the manifest above in the master cluster and KKP will pick up the new Seed and begin to
reconcile it by installing the required KKP components. You can watch the progress by using
`kubectl` and `watch`:

```bash
kubectl apply -f seed-with-secret.yaml
Secret/kubeconfig-kubermatic created.
Seed/kubermatic created.

watch kubectl -n kubermatic get pods
#NAME                                                   READY   STATUS    RESTARTS   AGE
#kubermatic-api-55765568f7-br9jl                        1/1     Running   0          5m4s
#kubermatic-api-55765568f7-xbvz2                        1/1     Running   0          5m13s
#kubermatic-dashboard-5d784d586b-f46f8                  1/1     Running   0          35m
#kubermatic-dashboard-5d784d586b-rgl29                  1/1     Running   0          35m
#kubermatic-master-controller-manager-f58d4df59-w7rkz   1/1     Running   0          5m13s
#kubermatic-operator-7f6957869d-89g55                   1/1     Running   0          5m37s
#nodeport-proxy-envoy-6d8bb6fbff-9z57l                  2/2     Running   0          5m6s
#nodeport-proxy-envoy-6d8bb6fbff-dl58l                  2/2     Running   0          4m54s
#nodeport-proxy-envoy-6d8bb6fbff-k4gp8                  2/2     Running   0          4m44s
#nodeport-proxy-updater-7fd55f948-cll8n                 1/1     Running   0          4m44s
#seed-proxy-kubermatic-6dd5cc95cf-r6wvb                 1/1     Running   0          80m
```

## Update DNS

Depending on the chosen [Expose Strategy]({{< ref "../../concepts/expose-strategy">}}), the control planes of all user clusters
running in the Seed cluster will be exposed by the `nodeport-proxy` or using
services of type `NodePort` directly.

By default each user cluster gets a virtual domain name like
`[cluster-id].[seed-name].[kubermatic-domain]`, e.g. `hdu328tr.kubermatic.kubermatic.example.com`
for the Seed from the previous step when `kubermatic.example.com` is the main domain where the
KKP dashboard/API are available.

To facilitate this, a wildcard DNS record `*.[seed-name].[kubermatic-domain]` must be created. The target of the
DNS wildcard record should be the `EXTERNAL-IP` of the `nodeport-proxy` service in the `kubermatic` namespace.

### With LoadBalancers

When your cloud provider supports LoadBalancers, you can find the target IP / hostname by looking at the
`nodeport-proxy` Service:

```bash
kubectl -n kubermatic get services
#NAME             TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
#nodeport-proxy   LoadBalancer   10.47.248.232   8.7.6.5        80:32014/TCP,443:30772/TCP   449d
```

The `EXTERNAL-IP` is what we need to put into the DNS record.

### Without LoadBalancers

Without a LoadBalancer, you will need to point to one or many of the seed cluster's nodes. You can get a
list of external IPs like so:

```bash
kubectl get nodes -o wide
#NAME                        STATUS   ROLES    AGE     VERSION         INTERNAL-IP   EXTERNAL-IP
#worker-node-cbd686cd-50nx   Ready    <none>   3h36m   v1.15.8-gke.3   10.156.0.36   8.7.6.4
#worker-node-cbd686cd-59s2   Ready    <none>   21m     v1.15.8-gke.3   10.156.0.14   8.7.6.3
#worker-node-cbd686cd-90j3   Ready    <none>   45m     v1.15.8-gke.3   10.156.0.22   8.7.6.2
```

### DNS Record

Create an A or CNAME record as needed pointing to the target:

```plain
*.kubermatic.kubermatic.example.com.   IN   A   8.7.6.5
```

or, for a CNAME:

```plain
*.kubermatic.kubermatic.example.com.   IN   CNAME   myloadbalancer.example.com.
```
