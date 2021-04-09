+++
title = "Expose Strategy"
date = 2018-05-04T12:07:15+02:00
weight = 5

+++

# Overview

The expose strategy defines how clusters manged by Kubermatic Kubernetes Platform (KKP) are made available to the outside world.

**Note**: The expose strategy of a cluster can not be changed after its creation without rotating all
of its nodes, as the `kubeconfig` of the `kubelet` will point to the wrong address.

Currently, there are three possible ways to expose clusters:

## Nodeport

A DNS record is created for each Seed cluster with a value of `*.<<seed-cluster-name>>.base.domain`
pointing to one or more of the seed clusters nodes. A `NodePort` will be opened for every user
cluster, clients will use the combination of the DNS entry and the port to connect.

This is very simple to set up and does not have any requirements onto the seed cluster.

## Global LoadBalancer

It is also possible to use one LoadBalancer per seed cluster instead of `NodePort`s. When doing so,
the `NodeportProxy` has to be deployed into the seed. It will create a Kubernetes Service of type
`LoadBalancer`. Afterwards, a DNS entry for `*.<<seed-cluster-name>>.base.domain` has to be created
that points to the `LoadBalancer`\`s address.

Whenever a new cluster is added or deleted, a controller that is part of the `NodePortProxy` will
add/remove a port on the `LoadBalancer` points to a set of Envoy proxies. These envoy proxies will
then redirect the traffic to the correct pods.

The envoy proxies are needed, because Kubernetes Services are not supported as an endpoint of another
Kubernetes Service.

This requires a functioning cloud provider that realizes services of type `LoadBalancer`. It is very
cost-efficient, as only one such service is needed.

## One LoadBalancer Per User Cluster (KKP 2.11+)

A third option is to create one `LoadBalancer` per user cluster. This is done by setting the
`kubermatic.exposeStrategy` key in the Helm chart to `LoadBalancer`.

This will result in one service of type `LoadBalancer` per user cluster being created. The
`NodeportProxy` will be automatically deployed by KKP to use this one service for the
traffic of both the OpenVPN and the apiserver.

This is simple to setup, but will result in one service of type `LoadBalancer` per cluster
KKP manages. This my result in additional charges by your cloud provider.
