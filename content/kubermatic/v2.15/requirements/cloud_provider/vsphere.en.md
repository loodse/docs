+++
title = "VSphere"
date = 2018-07-04T12:07:15+02:00
weight = 7

+++

## VSphere

{{% notice warning %}}
The Kubernetes vSphere driver contains bugs related to detaching volumes from offline nodes. See the [**Volume detach bug**](#volume-detach-bug) section for more details.
{{% /notice %}}

### VM Images

When creating worker nodes for a user cluster, the user can specify an existing image. Defaults may be set in the [seed cluster `spec.datacenters.EXAMPLEDC.vsphere.endpoint`]({{< ref "../../concepts/seeds" >}}).

Supported operating systems

* CentOS 7 [qcow2](https://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2)
* CoreOS  [ova](https://stable.release.core-os.net/amd64-usr/current/coreos_production_vmware_ova.ova) (End of Life, [2020-05-26](https://coreos.com/os/eol/))
* Flatcar Container Linux [ova](https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_vmware_ova.ova)
* Ubuntu 18.04 [ova](https://cloud-images.ubuntu.com/releases/18.04/release/ubuntu-18.04-server-cloudimg-amd64.ova)

#### Importing the OVA

1. Go into the VSphere WebUI, select your datacenter, right click onto it and choose "Deploy OVF Template"
1. Fill in the "URL" field with the appropriate url
1. Click through the dialog until "Select storage"
1. Select the same storage you want to use for your machines
1. Select the same network you want to use for your machines
1. Leave everyhting in the "Customize Template" and "Ready to complete" dialog as it is
1. Wait until the VM got fully imported and the "Snapshots" => "Create Snapshot" button is not grayed out anymore
1. The template VM must have the disk.enableUUID flag set to 1, this can be done using the [govc tool](https://github.com/vmware/govmomi/tree/master/govc) with the following command:

```bash
govc vm.change -e="disk.enableUUID=1" -vm='/PATH/TO/VM'
```

#### Importing the QCOW2

1. Convert it to vmdk: `qemu-img convert -f qcow2 -O vmdk CentOS-7-x86_64-GenericCloud.qcow2 CentOS-7-x86_64-GenericCloud.vmdk`
1. Upload it to a Datastore of your vSphere installation
1. Create a new virtual machine that uses the uploaded vmdk as rootdisk

#### Modifications

Modifications like Network, disk size, etc. must be done in the ova template before creating a worker node from it.
If user clusters have dedicated networks, all user clusters therefore need a custom template.

### VM Folder

During creation of a user cluster Kubermatic Kubernetes Platform (KKP) creates a dedicated VM folder in the root path on the Datastore (Defined in the [seed cluster `spec.datacenters.EXAMPLEDC.vsphere.datastore`]({{< ref "../../concepts/seeds" >}})).
That folder will contain all worker nodes of a user cluster.

### Credentials / Cloud-Config

Kubernetes needs to talk to the vSphere to enable Storage inside the cluster.
For this, kubernetes needs a config called `cloud-config`.
This config contains all details to connect to a vCenter installation, including credentials.

As this Config must also be deployed onto each worker node of a user cluster, its recommended to have individual credentials for each user cluster.

### Permissions

The vsphere user has to have to following permissions on the correct resources:

#### Seed Cluster

For provisioning actions of the KKP seed cluster, a technical user (e.g. `cust-seed-ccm`) is needed:

* Role `k8c-storage-vmfolder-propagate`
  * Granted at **VM Folder** and **Template Folder**, propagated
  * Permissions
    * Virtual machine
      * Change Configuration
        * Add existing disk
        * Add new disk
        * Add or remove device
        * Remove disk
    * Folder
      * Create folder
      * Delete dolder

* Role `k8c-storage-datastore-propagate`
  * Granted at **Datastore**, propagated
  * Permissions
    * Datastore
      * Allocate space
      * Low level file operations

* Role `Read-only` (predefined)
  * Granted at ..., **not** propagated
    * Datacenter

#### User Cluster

For provisioning actions of the KKP in scope of an user cluster, a technical user (e.g. `cust-user-cluster`) is needed:

* Role `k8c-user-vcenter`
  * Granted at **vcenter** level, **not** propagated
  * Needed to customize VM during provisioning
  * Permissions
    * Profile-driven storage
      * Profile-driven storage view
    * VirtualMachine
      * Provisioning
        * Modify customization specification
        * Read customization specifications

* Role `k8c-user-datacenter`
  * Granted at **datacenter** level, **not** propagated
  * Needed for cloning the template VM (obviously this is not done in a folder at this time)
  * Permissions
    * Datastore
      * Allocate space
      * Browse datastore
      * Low level file operations
      * Remove file
    * vApp
      * vApp application configuration
      * vApp instance configuration
    * Virtual Machine
      * Change CPU count
      * Memory
      * Settings
    * Inventory
      * Create from existing

* Role `k8c-user-cluster-propagate`
  * Granted at **cluster** level, propagated
  * Needed for upload of `cloud-init.iso` (Ubuntu and CentOS) or defining the Ignition config into Guestinfo (CoreOS)
  * Permissions
    * Host
      * Configuration
        * System Management
      * Local operations
        * Reconfigure virtual machine
    * Resource
      * Assign virtual machine to resource pool
      * Migrate powered off virtual machine
      * Migrate powered on virtual machine
    * vApp
      * vApp application configuration
      * vApp instance configuration

* Role k8s-network-attach
  * Granted for each network that should be used (distributed switch + network)
  * Permissions
    * Network
      * Assign network

* Role `k8c-user-datastore-propagate`
  * Granted at **datastore / datastore cluster** level, propagated
  * Permissions
    * Datastore
      * Allocate space
      * Browse datastore
      * Low level file operations

* Role `k8c-user-folder-propagate`
  * Granted at **VM Folder** and **Template Folder** level, propagated
  * Needed for managing the node VMs
  * Permissions
    * Folder
      * Create folder
      * Delete folder
    * Global
      * Set custom attribute
    * Virtual machine
      * Change Configuration
      * Edit Inventory
      * Guest operations
      * Interaction
      * Provisioning
      * Snapshot management

The described permissions have been tested with vSphere 6.7 and might be different for other vSphere versions.

#### Terraform Setup

It's also possible to create the roles by a terraform script. The following repo can be used as reference:
* https://github.com/kubermatic-labs/kubermatic-vsphere-permissions-terraform

#### Volume Detach Bug

After a node is powered-off, the Kubernetes vSphere driver doesn't detach disks associated with PVCs mounted on that node. This makes it impossible to reschedule pods using these PVCs until the disks are manually detached in vCenter.

Upstream Kubernetes has been working on the issue for a long time now and tracking it under the following tickets:

* <https://github.com/kubernetes/kubernetes/issues/63577>
* <https://github.com/kubernetes/kubernetes/issues/61707>
* <https://github.com/kubernetes/kubernetes/issues/67900>
* <https://github.com/kubernetes/kubernetes/issues/71829>
* <https://github.com/kubernetes/kubernetes/issues/75342>

### Datastores and Datastore Clusters

*Datastore* in VMWare vSphere is an abstraction for storage.
*Datastore Cluster* is a collection of datastores with shared resources and a
shared management interface.

In KKP *Datastores* are used for two purposes:
- Storing the VMs files for the worker nodes of vSphere user clusters.
- Generating the vSphere cloud provider storage configuration for user clusters.
    In particular to provide the `dafault-datastore` value, that is the default
    datastore for dynamic volume provisioning.

*Datastore Clusters* can only be used for the first purpose as it cannot be
specified directly in [vSphere cloud configuration][vsphere-cloud-config].

There are two places where Datastores and Datastore Clusters can be configured
in KKP

- At datacenter level (either with [Seed CRD]({{< ref "../../concepts/seeds" >}})
    or [datacenters.yaml]({{< ref "../../concepts/seeds" >}})) is possible to
    specify the default *Datastore* that will be used for user clusters dynamic
    volume provisioning and workers VMs placement in case no *Datastore* or
    *Datastore Cluster* is specified at cluster level.
- At *Cluster* level it is possible to provide either a *Datastore* or a
    *Datastore Cluster* respectively with `spec.cloud.vsphere.datastore` and
    `spec.cloud.vsphere.datastoreCluster` fields.

{{% notice warning %}}
At the moment of writing this document *Datastore and *Datastore Cluster*
are not supported yet at `Cluster` level by Kubermatic UI.
{{% /notice %}}

It is possible to specify *Datastore* or *Datastore Clusters* in a
[preset]({{< ref "../../advanced/presets" >}}).

[vsphere-cloud-config]: https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/existing.html#template-config-file
