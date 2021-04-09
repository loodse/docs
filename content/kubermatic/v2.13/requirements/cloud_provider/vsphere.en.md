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

When creating worker nodes for a user cluster, the user can specify an existing image. Defaults may be set in the [datacenters.yaml](https://docs.kubermatic.io/installation/install_kubermatic/#defining-the-datacenters).

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

During creation of a user cluster Kubermatic Kubernetes Platform (KKP) creates a dedicated VM folder in the root path on the Datastore (Defined in the [datacenters.yaml](https://docs.kubermatic.io/installation/install_kubermatic/#defining-the-datacenters)).
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

#### Volume Detach Bug

After a node is powered-off, the Kubernetes vSphere driver doesn't detach disks associated with PVCs mounted on that node. This makes it impossible to reschedule pods using these PVCs until the disks are manually detached in vCenter.

Upstream Kubernetes has been working on the issue for a long time now and tracking it under the following tickets:

* <https://github.com/kubernetes/kubernetes/issues/63577>
* <https://github.com/kubernetes/kubernetes/issues/61707>
* <https://github.com/kubernetes/kubernetes/issues/67900>
* <https://github.com/kubernetes/kubernetes/issues/71829>
* <https://github.com/kubernetes/kubernetes/issues/75342>
