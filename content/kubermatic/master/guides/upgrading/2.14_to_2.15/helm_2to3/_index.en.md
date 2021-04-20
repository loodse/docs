+++
title = "Migrating to Helm 3"
date = 2020-06-09T11:09:15+02:00
weight = 40

+++

Helm supports in-place migrations using the [2to3 plugin](https://github.com/helm/helm-2to3).
The migration involves converting the Helm 2 release information (stored in ConfigMaps/Secrets in
the Tiller namespace) to Helm 3 releases, cleaning up Helm 2 releases afterwards and then removing
Tiller and its associated resources in the final step.

During the migration, existing resources are not modified, so LoadBalancers will for example not
potentially trigger an IP change.

### Example

{{% notice warning %}}
Make sure that during a migration no other operations are performed using Helm 2, for example
due to "accidental" automated deployment jobs. Ideally, all charts are migrated sequentially
and then Tiller is removed.
{{% /notice %}}

In this example, the `cert-manager` chart is migrated to Helm 3.

Before we begin, we have to install the 2to3 plugin. Throughout the example, we're using Helm 3
exclusively.

```bash
helm plugin install https://github.com/helm/helm-2to3.git
#Downloading and installing helm-2to3 v0.6.0 ...
#https://github.com/helm/helm-2to3/releases/download/v0.6.0/helm-2to3_0.6.0_linux_amd64.tar.gz
#Installed plugin: 2to3
```

Next you need a kubeconfig for the cluster in which the releases should be migrated. The
kubeconfig can either be specified explicitly for each `2to3` call
(e.g. `helm 2to3 convert --kubeconfig=<path>`) or set as a `KUBECONFIG` environment variable.

```bash
export KUBECONFIG=/path/to/kubeconfig
```

We can now convert the release information from version 2 to 3. This will by default only
convert (and delete) the 10 most recent releases, so a cleanup is later required to remove the
remaining releases (if there were more than 10).

Remove `--dry-run` to actually perform the conversion.

```bash
helm 2to3 convert \
  --dry-run \
  --tiller-ns kubermatic \
  --delete-v2-releases \
  cert-manager
```

Now the remaining Helm 2 releases can be removed:

```bash
helm 2to3 cleanup \
  --dry-run \
  --tiller-ns kubermatic \
  --name cert-manager
```

This completes the migration for the `cert-manager` chart.

After all migrations have completed, Tiller can then also be removed by running the `cleanup`
command again without a `--name` flag (see `--help`):

```bash
helm 2to3 cleanup \
  --dry-run \
  --tiller-ns kubermatic
```
