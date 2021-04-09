+++
title = "Restoring from backup"
date = 2018-07-24T12:07:15+02:00
weight = 0

+++

## Intro

The etcd's of the user-clusters are being backed up on a configured interval.
This document will lead through the process of restoring a complete etcd StatefulSet from a single snapshot.

### Pausing the Cluster

Restoring a etcd requires manual intervention.
As the StatefulSet needs to be modified, the affected cluster needs to be removed from the controllers management:

```bash
# set cluster.spec.pause=true
kubectl edit cluster xxxxxxxxxx
```

### Pausing the StatefulSet

To restore an etcd, the etcd must not be running.
Therefore the `etcd` statefulset must be configured:
 1) to just execute as `command:` `exec /bin/sleep 86400` and remove all `args:`.
 2) remove the `readinessProbe:` and `livenessProbe:` specifications

```bash
# change command to run 'exec /bin/sleep 86400'
kubectl -n cluster-xxxxxxxxxx edit statefulset etcd
```

### Deleting All PVC's

To ensure that we start on each pod with a empty disk, we delete all PVC's.
The StatefulSet will create new ones with empty PV's automatically.

```bash
kubectl -n cluster-xxxxxxxxxx delete pvc -l app=etcd
```

### Deleting All Pods

To ensure all Pods start with the sleep command and with new PV's, all etcd pods must be deleted.

```bash
kubectl -n cluster-xxxxxxxxxx delete pod -l app=etcd
```

### Restoring the etcd (Must Be Executed on All etcd Pods)

The restore command is different for each member. Make sure to update it gets executed.

```bash
# Copy snapshot into pod
kubectl cp local-snapshot.db cluster-xxxxxxxxxx/etcd-0:/snapshot.db
# Exec into the pod
kubectl -n cluster-xxxxxxxxxx exec -ti etcd-0 sh

cd /
# Inside the pod, restore from the snapshot
# This command is specific to each member.
export MEMBER=etcd-0
export CLUSTER_ID=xxxxxxxxxx

etcdctl snapshot restore snapshot.db \
  --name ${MEMBER} \
  --initial-cluster etcd-0=http://etcd-0.etcd.cluster-${CLUSTER_ID}.svc.cluster.local:2380,etcd-1=http://etcd-1.etcd.cluster-${CLUSTER_ID}.svc.cluster.local:2380,etcd-2=http://etcd-2.etcd.cluster-${CLUSTER_ID}.svc.cluster.local:2380 \
  --initial-cluster-token ${CLUSTER_ID} \
  --initial-advertise-peer-urls http://${MEMBER}.etcd.cluster-${CLUSTER_ID}.svc.cluster.local:2380 \
  --data-dir /var/run/etcd/pod_${MEMBER}/
```

### Un-Pausing the Cluster

To let the kubermatic Kubernetes Platform (KKP)-controller-manager update the etcd to normal state, un-pause it.

```bash
# set cluster.spec.pause=false
kubectl edit cluster xxxxxxxxxx
```

### Delete etcd-Pods

As the rolling-update of the etcd won't finish, all etcd pods must be manually.

```bash
kubectl -n cluster-xxxxxxxxxx delete pod -l app=etcd
```
