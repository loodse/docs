+++
title = "Upgrading from 2.15 to 2.16"
date = 2021-02-03T11:09:15+01:00
weight = 100

+++

This document describes the major changes between KKP 2.15 and 2.16. It's recommended to read them carefully
before performing an upgrade.

## Removing the Legacy `kubermatic` Helm Chart

As announced with 2.15, this release is now the last to provide the `kubermatic` Helm chart. Users must
migrate to using either the KKP Installer, or manually install the `kubermatic-operator` chart.

## Removed ELK Logging Stack

The Helm charts for `elasticsearch`, `kibana` and `fluentbit` have been removed, as announced with KKP 2.15.
Users are encouraged to migrate to Grafana Loki, for which Helm charts are provided by Kubermatic.

## Certificate Handling

In previous releases, KKP used explicit Certificate resources to manage TLS certificates. This resulted in a
rather strong dependency on cert-manager to be installed into the cluster, plus issues handling
non-Let's Encrypt certificates.

To allow for more flexibility in providing certificates, KKP 2.16 switches to using the `cert-manager.io/issuer`
or `cert-manager.io/cluster-issuer` annotations on Ingress objects. cert-manager is still the recommended
solution for acquiring certificates (and is installed by default), but other avenues may be chosen instead.

## Hetzner / HCloud improvements in 2.16.3

Version 2.16.3 ships significant improvements to how userclusters on Hetzner are supported. Starting with
Kubernetes 1.18, KKP now supports the external cloud-controller-manager (CCM) and CSI, which allows to use storage
and LoadBalancers without any user intervention. These changes are so important that they were backported into
the 2.16.x release branch. The following sections explain the migration strategy.

### CCM

To use the HCloud CCM, the following conditions must be met:

* The usercluster must be using Kubernetes 1.18+.
* A `network` must be configured for the Hetzner datacenter (in the Seed resources) or the Preset (if a Preset
  is used). Do note that in 2.16.3 this is optional, but in future KKP versions this field will be mandatory.
* Only newly created userclusters will be able to use the CCM; enabling this on existing cluster is not
  supported and might cause issues.

If all three conditions are met, newly created userclusters will get the `externalCloudProvider` feature flag,
which will ensure that the HCloud CCM is deployed inside the seed cluster (similar to how the machine-controller
works).

### CSI

KKP previously shipped an outdated CSI Driver for Hetzner, which has now been updated to the most recent
version. This affects the `default-storage-class` addon, which is installed by default into every usercluster.
After updating to 2.16.3, the new addon with the updated CSI components will be installed into all Hetzner
userclusters.
