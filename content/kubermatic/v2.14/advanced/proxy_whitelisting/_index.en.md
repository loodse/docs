+++
title = "Proxy Whitelisting"
date = 2019-09-13T12:07:15+02:00
weight = 110

+++

To enable KKP behind a proxy environment, the following targets need to be reachable.

{{% notice note %}}
If you use the [KKP offline mode]({{< ref "../offline_mode" >}}), images will get pulled from the defined private registry (e.g. `172.20.0.2:5000`) instead of the public registries. For more details see the [KKP offline mode]({{< ref "../offline_mode" >}}) section.
{{% /notice %}}

## KKP Machine Controller

Resources pulled on machine controller nodes

### kubelet - Binary

The machine controller is downloading a few components to install the kubelet, see [download_binaries_script.go](https://github.com/kubermatic/machine-controller/blob/master/pkg/userdata/helper/download_binaries_script.go):

```bash
# Binaries for the Kubernetes kubelet Get Downloaded From:
https://storage.googleapis.com/kubernetes-release/release/

# CNI Plugins
https://github.com/containernetworking/plugins/releases/

# KKP Health-Monitor Script
# (Placed at pkg/userdata/scripts/health-monitor.sh)
https://raw.githubusercontent.com/kubermatic/machine-controller/
```

### kubelet - Docker Images

After kubelet starts, it needs a few more images to work in a proper way:

**`gcr.io`:**

```bash
# ContainerLinux Requires the Hyperkube Image
gcr.io/google_containers/hyperkube-amd64

# DNS Node Cache
gcr.io/google_containers/k8s-dns-node-cache
```

**`k8s.gcr.io`:**

```bash
# Every kubelet Requires the Pause Container:
k8s.gcr.io/pause
```

**`docker.io`:**

```bash
# Calico Overlay
calico/node

# DNS Addon
coredns/coredns

# Log Shipper Fluent-Bit
fluent/fluent-bit
```

**`quay.io`:**

```bash
# Util Container for Debugging or Custom Controller
quay.io/kubermatic/util

# Prometheus Metrics Scraping
quay.io/prometheus/node-exporter

# Core Os Container
quay.io/coreos/flannel
quay.io/coreos/kube-rbac-proxy
quay.io/coreos/container-linux-update-operator
```

### OS Resources
Additional to the kubelet dependencies, the [machine controller OS provider](https://github.com/kubermatic/machine-controller/tree/master/pkg/userdata) installs some os specific packages over cloud-init:

#### CentOS 7/8
Init script: [pkg/userdata/centos](https://github.com/kubermatic/machine-controller/tree/master/pkg/userdata/centos)

- default yum repositories
- docker yum repository: `download.docker.com/linux/centos`

### CoreOS / Flatcar Linux / SLES
Init script: [pkg/userdata/coreos](https://github.com/kubermatic/machine-controller/tree/master/pkg/userdata/coreos), [pkg/userdata/flatcar](https://github.com/kubermatic/machine-controller/blob/master/pkg/userdata/flatcar), [pkg/userdata/sles](https://github.com/kubermatic/machine-controller/tree/master/pkg/userdata/sles)

- no additional targets

### Ubuntu 18.04/20.04
Init script: [pkg/userdata/ubuntu](https://github.com/kubermatic/machine-controller/tree/master/pkg/userdata/ubuntu)

- default apt repositories
- docker apt repository: `download.docker.com/linux/ubuntu`

# KKP Seed Cluster Setup

## Cloud Provider API Endpoints
KKP interacts with the different cloud provider directly to provision the required infrastructure to manage Kubernetes clusters:

### AWS
API endpoint documentation: [AWS service endpoints](https://docs.aws.amazon.com/general/latest/gr/rande.html)

KKP interact in several ways with different cloud provider, e.g.:
- creating EC2 instances
- creating security groups
- access instance profiles

```bash
# e.g. For Region Eu-Central-1
iam.amazonaws.com
s3.eu-central-1.amazonaws.com
ec2.eu-central-1.amazonaws.com
```

### Azure
API endpoint documentation: [Azure API Docs - Request URI](https://docs.microsoft.com/en-us/rest/api/azure/#request-uri)
```bash
# Resource Manager API
management.azure.com
# Azure classic deployment API
management.core.windows.net
# Azure Authentication API
login.microsoftonline.com
```

### vSphere
API Endpoint URL of all targeted vCenters specified in [seed cluster `spec.datacenters.EXAMPLEDC.vsphere.endpoint`]({{< ref "../../concepts/seeds" >}}), e.g. `vcenter.example.com`.


## KubeOne Seed Cluster Setup

If [KubeOne](https://github.com/kubermatic/kubeone) is used to setup the seed cluster, kubeone will use in addition to OS specific default repositories the following URIs (see [os.go](https://github.com/kubermatic/kubeone/blob/master/pkg/scripts/os.go)):

```bash
# debian / ubuntu
packages.cloud.google.com/apt
download.docker.com/linux/ubuntu
apt.kubernetes.io
## on azure VM's
azure.archive.ubuntu.com
# security packages ubuntu
security.ubuntu.com

# centos
packages.cloud.google.com/yum
download.docker.com/linux/centos

# CoreOS / Flatcar Linux
storage.googleapis.com/kubernetes-release/release
github.com/containernetworking/plugins/releases/download

# gobetween (if used, e.g. at vsphere terraform setup)
github.com/yyyar/gobetween/releases
```
**At installer host / bastion server**:
```bash
## terraform modules
registry.terraform.io
releases.hashicorp.com

## kubeone binary
https://github.com/kubermatic/kubeone/releases
```

## cert-manager (if used)
For creating certificates with let's encrypt we need access:

```bash
https://acme-v02.api.letsencrypt.org/directory
```

## EFK Logging Stack (if used)
To download the elasticsearch artifacts (deprecated in flavor of Loki):

```
docker.elastic.co/elasticsearch/elasticsearch-oss
docker.elastic.co/kibana/kibana-oss
```
