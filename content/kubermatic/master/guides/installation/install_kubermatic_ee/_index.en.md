+++
title = "Install Kubermatic Kubernetes Platform (KKP) EE"
date = 2018-04-28T12:07:15+02:00
weight = 20
enterprise = true
+++

This chapter explains the installation procedure of KKP Enterprise Edition (EE) into a pre-existing
Kubernetes cluster.

{{% notice note %}}
At the moment you need to be invited to get access to Kubermatic's EE Docker repository before you can try it out.
Please [contact sales](mailto:sales@kubermatic.com) to receive your credentials.
{{% /notice %}}

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
This guide assumes a clean installation into an empty cluster. Please refer to the [upgrade notes]({{< ref "../../upgrading" >}})
for more information on migrating existing installations to the Kubermatic Installer.
{{% /notice %}}

## Installation

The installation procedure is identical to the [Community Edition]({{< ref "../install_kubermatic" >}}), with the exception that
a different installer needs to be downloaded and that the Docker credentials need to be configured.

When downloading the installer, make sure to choose the `-ee-` variant on GitHub. Extract it like documented in the CE install
guide.

During configuration, it's required to set the Docker Pull Secret, which allows the local Docker daemons to pull the KKP
images from the private Docker repository. The Docker Pull Secret is a tiny JSON snippet and needs to be configured in the
KubermaticConfiguration (e.g. in the `kubermatic.yaml`):

```yaml
apiVersion: operator.kubermatic.io/v1alpha1
kind: KubermaticConfiguration
metadata:
  name: kubermatic
  namespace: kubermatic
spec:
  # skipping everything else in this file
  # for demonstration purposes

  # This is where the JSON snippet needs to be configured. It does not need to be
  # a multiline JSON string.
  imagePullSecret: |
    {
      "auths": {
        "quay.io": {....}
      }
    }
```

Follow the CE install guide as normal, the remaining steps apply equally to the Enterprise Edition.

### Next Steps

* [Add a Seed cluster]({{< ref "../add_seed_cluster_ee" >}}) to start creating user clusters.
* Install the [monitoring stack]({{< ref "../monitoring_stack" >}}) to gain metrics and alerting.
* Install the [logging stack]({{< ref "../logging_stack" >}}) to collect cluster-wide metrics in a central place.
