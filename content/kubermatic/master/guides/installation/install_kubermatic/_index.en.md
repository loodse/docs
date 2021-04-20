+++
title = "Install Kubermatic Kubernetes Platform (KKP) CE"
date = 2018-04-28T12:07:15+02:00
weight = 20

+++

This chapter explains the installation procedure of KKP into a pre-existing Kubernetes cluster.

## Terminology

* **User/Customer cluster** -- A Kubernetes cluster created and managed by KKP
* **Seed cluster** -- A Kubernetes cluster which is responsible for hosting the master components of a customer cluster
* **Master cluster** -- A Kubernetes cluster which is responsible for storing the information about users, projects and SSH keys. It hosts the KKP components and might also act as a seed cluster.
* **Seed datacenter** -- A definition/reference to a seed cluster
* **Node datacenter** -- A definition/reference of a datacenter/region/zone at a cloud provider (aws=zone, digitalocean=region, openstack=zone)

## Requirements

Before installing, make sure your Kubernetes cluster meets the [minimal requirements]({{< ref "." >}})
and make yourself familiar with the requirements for your chosen cloud provider.

For this guide you will have to have `kubectl` and [Helm](https://www.helm.sh/) (version 3) installed locally.

{{% notice warning %}}
This guide assumes a clean installation into an empty cluster. Please refer to the [upgrade notes]({{< ref "../../upgrading" >}}) for more information on
migrating existing installations to the Kubermatic Installer.
{{% /notice %}}

## Installation

To begin the installation, make sure you have a kubeconfig file at hand, with a user context that grants `cluster-admin`
permissions.

### Download the Installer

Download the [tarball](https://github.com/kubermatic/kubermatic/releases/) (e.g. kubermatic-ce-X.Y-linux-amd64.tar.gz)
containing the Kubermatic Installer and the required Helm charts for your operating system and extract it locally. Note that
for Windows, ZIP files are provided instead of tar.gz files.

```bash
# For latest version:
VERSION=$(curl -w '%{url_effective}' -I -L -s -S https://github.com/kubermatic/kubermatic/releases/latest -o /dev/null | sed -e 's|.*/v||')
# For specific version set it explicitly:
# VERSION=2.15.x
wget https://github.com/kubermatic/kubermatic/releases/download/v${VERSION}/kubermatic-ce-v${VERSION}-linux-amd64.tar.gz
tar -xzvf kubermatic-ce-v${VERSION}-linux-amd64.tar.gz
```

### Prepare Configuration

The installation and configuration for a KKP system consists of two important files:

* A `values.yaml` used to configure the various Helm charts KKP ships with. This is where nginx, Prometheus,
  Dex etc. can be adjusted to the target environment. A single `values.yaml` is used to configure all Helm charts
  combined.
* A `kubermatic.yaml` that configures KKP itself and is an instance of the
  [KubermaticConfiguration]({{< ref "." >}}) CRD. This configuration will
  be stored in the cluster and serves as the basis for the Kubermatic Operator to manage the actual KKP installation.

The release archive hosted on GitHub contains examples for both of the configuration files (`values.example.yaml` and
`kubermatic.example.yaml`). It's a good idea to take them as a starting point and add more options as necessary.

The key items to configure are:

* The base domain under which KKP shall be accessible (e.g. `kubermatic.example.com`).
* The certificate issuer: KKP requires that its dashboard and Dex are only accessible via HTTPS, so a
  certificate is required. By default cert-manager is used, but you have to choose between the production or
  staging Let's Encrypt services (if in doubt, choose the production server).
  It is possible to use a custom CA (i.e. self-signed certificates), but this is outside of the scope of this
  document.
* For proper authentication, shared secrets must be configured between Dex and KKP. Likewise, Dex uses
  yet another random secret to encrypt cookiesstored in the users' browsers.
* The expose strategy, that is the strategy used to expose the control plane
  components to the worker nodes (see the [expose strategy]({{< ref "../expose_strategy">}}))

There are many more options, but these are essential to get a minimal system up and running. The secret keys
mentioned above can be generated using any password generator or on the shell using
`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c32`. On MacOS, use `brew install gnu-tar` and
`cat /dev/urandom | gtr -dc A-Za-z0-9 | head -c32`. Alternatively, the Kubermatic Installer will suggest some
properly generated secrets for you when it notices that some are missing, for example:

```bash
./kubermatic-installer deploy --config kubermatic.yaml --helm-values values.yaml
INFO[15:15:20] 🛫 Initializing installer…                     edition="Community Edition" version=v2.15.11
INFO[15:15:20] 🚦 Validating the provided configuration…
ERROR[15:15:20]    The provided configuration files are invalid:
ERROR[15:15:20]    KubermaticConfiguration: spec.auth.serviceAccountKey must be a non-empty secret, for example: ZPCs7_KzgJxUSA5lCk_oNzL7RQFTQ6cOnHuTLAh4pGw
ERROR[15:15:20]    Operation failed: please review your configuration and try again.
```

{{% notice note %}}
A couple of settings are duplicated across the `values.yaml` and the KubermaticConfiguration CRD. The installer
will take care of filling in the gaps, so it is sufficient to configure the base domain in the
KubermaticConfiguration and let the installer set it in `values.yaml` as well.
{{% /notice %}}

### Create a StorageClass

KKP uses a custom storage class for the volumes created for user clusters. This class, `kubermatic-fast`, needs
to be created before the installation can succeed and is strongly recommended to use SSDs. The etcd clusters for
every user cluster will store their data in this StorageClass and etcd is highly sensitive to slow disk I/O.

The installer can automatically create an SSD-based StorageClass for a subset of cloud providers. It can also
simply copy the default StorageClass, but this is not recommended for production setups unless the default class
is using SSDs.

Use the `--storageclass` parameter for automatically creating the class during installation. Currently th efollowing
providers are supported:

- AWS
- Azure
- DigitalOcean
- GCE
- Hetzner

Run the installer with `--help` to also see the current list of supported providers.

If no automatic provisioning is possible, please manually create a StorageClass called `kubermatic-fast`. Consult
the [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/#parameters) for more
information about the possible parameters for your storage backend.

### Run Installer

Once the configuration files have been prepared, it's time to run the installer, which will validate them
and then install all the required components into the cluster. Open a terminal window and run the installer
like so:

```bash
./kubermatic-installer deploy \
  --config kubermatic.yaml \
  --helm-values values.yaml \
  --storageclass aws
```

{{% notice warning %}}
If you get an error about Helm being too old, download the most recent version from https://helm.sh/ and either
replace your system's Helm installation or specify the path to the Helm 3 binary via `--helm-binary ...` (for
example `./kubermatic-installer deploy .... --helm-binary /home/me/Downloads/helm-3.3.1`)
{{% /notice %}}

Once the installer has finished, the KKP Master cluster has been installed and will be ready to use once
the necessary DNS records have been configured (see the next steps).

{{% notice note %}}
Note that because we don't yet have a TLS certificate and no DNS records configured, some of the pods will crashloop.
This is normal for fresh setups and once the DNS records have been set, things will sort themselves out.
{{% /notice %}}

If you change your mind later on and adjust configuration options, it's safe to just run the installer again
to apply your changes.

### Create DNS Records

In order to acquire a valid certificate, a DNS name needs to point to your cluster. Depending on your environment,
this can mean a LoadBalancer service or a NodePort service. The nginx-ingress-controller Helm chart will by default
create a LoadBalancer, unless you reconfigure it because your environment does not support LoadBalancers.

The installer will do its best to inform you about the required DNS records to set up. You will receive an output
similar to this:

```bash
INFO[13:03:33]    📝 Applying Kubermatic Configuration…
INFO[13:03:33]    ✅ Success.
INFO[13:03:33]    📡 Determining DNS settings…
INFO[13:03:33]       The main LoadBalancer is ready.
INFO[13:03:33]
INFO[13:03:33]         Service             : nginx-ingress-controller / nginx-ingress-controller
INFO[13:03:33]         Ingress via hostname: EXAMPLEEXAMPLEEXAMPLEEXAMPLE-EXAMPLE.eu-central-1.elb.amazonaws.com
INFO[13:03:33]
INFO[13:03:33]       Please ensure your DNS settings for "kubermatic.example.com" include the following records:
INFO[13:03:33]
INFO[13:03:33]          kubermatic.example.com.    IN  CNAME  EXAMPLEEXAMPLEEXAMPLEEXAMPLE-EXAMPLE.eu-central-1.elb.amazonaws.com.
INFO[13:03:33]          *.kubermatic.example.com.  IN  CNAME  EXAMPLEEXAMPLEEXAMPLEEXAMPLE-EXAMPLE.eu-central-1.elb.amazonaws.com.
INFO[13:03:33]
INFO[13:03:33] 🛬 Installation completed successfully. ✌
```

Follow the instructions on screen to setup your DNS. If the installer for whatever reason is unable to determine
the appropriate DNS settings, it will tell you so and you can manually collect the required information from the
cluster. See the following sections for more information regarding the required DNS records.

#### With LoadBalancers

When your cloud provider supports LoadBalancers, you can find the target IP / hostname by looking at the
`nginx-ingress-controller` Service:

```bash
kubectl -n nginx-ingress-controller get services
#NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
#nginx-ingress-controller   LoadBalancer   10.47.248.232   1.2.3.4        80:32014/TCP,443:30772/TCP   449d
```

The `EXTERNAL-IP` is what we need to put into the DNS record. Note that this can be a hostname (for example on AWS,
this can be `my-loadbalancer-1234567890.us-west-2.elb.amazonaws.com`) and in this case, the DNS record needs to
be a `CNAME` rather than an `A` record.

#### Without LoadBalancers

Without a LoadBalancer, you will need to use the NodePort service (refer to the `charts/nginx-ingress-controller/values.yaml`
for more information) and setup the DNS records to point to one or many of your cluster's nodes. You can get a list of
external IPs like so:

```bash
kubectl get nodes -o wide
#NAME                        STATUS   ROLES    AGE     VERSION         INTERNAL-IP   EXTERNAL-IP
#worker-node-cbd686cd-50nx   Ready    <none>   3h36m   v1.15.8-gke.3   10.156.0.36   1.2.3.4
#worker-node-cbd686cd-59s2   Ready    <none>   21m     v1.15.8-gke.3   10.156.0.14   1.2.3.5
#worker-node-cbd686cd-90j3   Ready    <none>   45m     v1.15.8-gke.3   10.156.0.22   1.2.3.6
```

{{% notice note %}}
Some cloud providers list the external IP as the `INTERNAL-IP` and show no value for the `EXTENAL-IP`. In this case,
use the internal IP.
{{% /notice %}}

For this example we choose the second node, and so `1.2.3.5` is our DNS record target.

#### DNS Records

The main DNS record must connect the `kubermatic.example.com` domain with the target IP / hostname. Depending on whether
or not your LoadBalancer/node uses hostnames instead of IPs (like AWS ELB), create either an **A** or a **CNAME** record,
respectively.

```plain
kubermatic.example.com.   IN   A   1.2.3.4
```

or, for a CNAME:

```plain
kubermatic.example.com.   IN   CNAME   myloadbalancer.example.com.
```

#### Identity Aware Proxy

It's a common step to later setup an identity-aware proxy (IAP) to
[securely access other KKP components]({{< ref "." >}}) from the logging or monitoring
stacks. This involves setting up either individual DNS records per IAP deployment (one for Prometheus, one for Grafana, etc.)
or simply creating a single **wildcard** record: `*.kubermatic.example.com`.

Whatever you choose, the DNS record needs to point to the same endpoint (IP or hostname, meaning A or CNAME
records respectively) as the previous record, i.e. `1.2.3.4`. This is because the one nginx-ingress-controller is routing
traffic both for KKP and all other services.

```plain
*.kubermatic.example.com.   IN   A       1.2.3.4
; or for a CNAME:
*.kubermatic.example.com.   IN   CNAME   myloadbalancer.example.com.
```

If CNAME records are not possible, you would configure individual records instead:

```plain
prometheus.kubermatic.example.com.     IN   A       1.2.3.4
alertmanager.kubermatic.example.com.   IN   A       1.2.3.4
```

#### Validation

With the 2 DNS records configured, it's now time to wait for the certificate to be acquired. You can watch the progress
by doing `watch kubectl -n kubermatic get certificates` until it shows `READY=True`:

```bash
watch kubectl -n kubermatic get certificates
#NAME         READY   SECRET           AGE
#kubermatic   True    kubermatic-tls   1h
```

If the certificate does not become ready, `kubectl describe` it and follow the chain from Certificate to Order to Challenges.
Typical faults include bad DNS records or a misconfigured KubermaticConfiguration pointing to a different domain.

### Have a Break

With all this in place, you should be able to access https://kubermatic.example.com/ and login either with your static
password from the `values.yaml` or using any of your chosen connectors. All pods running inside the `kubermatic` namespace
should now be running. If they are not, check their logs to find out what's broken.

### Next Steps

* [Add a Seed cluster]({{< ref "../add_seed_cluster" >}}) to start creating user clusters.
* Install the [monitoring stack]({{< ref "../monitoring_stack" >}}) to gain metrics and alerting.
* Install the [logging stack]({{< ref "../logging_stack" >}}) to collect cluster-wide metrics in a central place.
