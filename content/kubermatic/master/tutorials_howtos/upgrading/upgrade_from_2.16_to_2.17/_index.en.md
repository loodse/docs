+++
title = "Upgrading from 2.16 to 2.17"
date = 2021-04-22T17:33:39+02:00
weight = 110

+++

## [EE] End of support for `kubermatic` Helm Chart

After the Kubermatic Operator has been introduced as a beta in version 2.14, it is now the recommended way of
installing and managing KKP. This means that the `kubermatic` Helm chart was deprecated as of
version 2.15 and starting from version 2.17 it is not supported any longer.

The Kubermatic Operator does not support previously deprecated features like the `datacenters.yaml`
or the full feature set of the `kubermatic` chart's customization options. Instead, datacenters
have to be converted to `Seed` resources, while the chart configuration must be converted to a
`KubermaticConfiguration`. The Kubermatic Installer offers commands to perform these conversions
automatically.

Note that the following customization options are not yet supported in the Kubermatic Operator:

* `maxParallelReconcile` (always defaults to `10`)
* Node and Pod affinities, node selectors for the KKP components
* Worker goroutine count for the KKP components

## End of support for CoreOS Container Linux

KKP 2.17 completely dropped support for CoreOS Container Linux, since it is
end-of-life since May 2020 and no longer receives updates.
Before upgrading your KKP installation be sure that none of the user clusters
is still using CoreOS operating system.

You can find some information about how to handle the migration
[here](../../guides/kkp_os_support/coreos_eos).

## Upgrade paths

Depending on the installation method you have to chose one of the following
upgrade paths:

* [Upgrade from Chart-based installations]({{< ref "./chart_migration" >}})
* [Upgrade from Operator-based installations]({{< ref "./kubermatic_operator" >}})

