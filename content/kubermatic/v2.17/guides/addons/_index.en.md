+++
title = "Addons"
date = 2018-06-21T14:07:15+02:00
weight = 20

+++

Addons are specific services and tools extending the functionality of Kubernetes/OpenShift.

### Default Addons

Default addons are installed in each user-cluster in KKP. The default addons are:

* [Canal](https://github.com/projectcalico/canal): policy based networking for cloud native applications
* [Dashboard](https://github.com/kubernetes/dashboard): General-purpose web UI for Kubernetes clusters
* [DNS](https://github.com/coredns/coredns): Kubernetes DNS service
* [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/): Kubernetes network proxy
* [rbac](https://kubernetes.io/docs/reference/access-authn-authz/rbac/): Kubernetes Role-Based Access Control, needed for
  [TLS node bootstrapping](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)
* [OpenVPN client](https://openvpn.net/index.php/open-source/overview.html): virtual private network (VPN). Lets the control
  plan access the Pod & Service network. Required for functionality like `kubectl proxy` & `kubectl port-forward`.
* [nodelocal-dns-cache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/): Improves DNS performance inside
  the user-cluster
* [logrotate](https://github.com/blacklabelops/logrotate): rotates logs on worker nodes
* pod-security-policy: Policies to configure KKP access when PSPs are enabled
* default-storage-class: A cloud provider specific StorageClass
* kubelet-configmap: A set of ConfigMaps used by kubeadm

Installation and configuration of these addons is done by 2 controllers which are part of the KKP
seed-controller-manager:

* `addon-installer-controller`: Ensures a given set of addons will be installed in all clusters
* `addon-controller`: Templates the addons & applies the manifests in the user clusters

The KKP Operator comes with a `kubermatic-operator-util` tool, which can output a full default
KubermaticConfiguration. This will also include the default configuration for addons and can serve as
a starting point for adjustments.

```bash
docker run --rm quay.io/kubermatic/api:KUBERMATIC_VERSION kubermatic-operator-util defaults
#apiVersion: operator.kubermatic.io/v1alpha1
#kind: KubermaticConfiguration
#metadata:
#  name: kubermatic
#  namespace: kubermatic
#spec:
#  ...
#  userClusters:
#    addons:
#      kubernetes: ...
#      openshift: ...
```

### Configuration

To configure which addons shall be installed in all user clusters, update the relevant
[KubermaticConfiguration]({{< ref "../../tutorials_howtos/kkp_configuration" >}}) in the `spec.userCluster.addons`
section. For Kubernetes and OpenShift, configure a Docker image that contains the required addon manifests
(as YAML files) and an `AddonList` manifest that lists the addons and their requirements. Take a look
at the default configuration above as a starting point.

To deploy any changes, edit the KubermaticConfiguration in-place using `kubectl edit` or apply it from
a file using `kubectl apply`. The KKP Operator will reconcile the installation and after a few moments
your changes will take effect.

### Accessible Addons

Accessible addons can be installed in each user-cluster in KKP on user demand (in contrast to
regular addons, which are always installed and cannot be removed by the user). If an addon is both default
and accessible, then it will be installed in the user-cluster, but also be visible to the user, who can manage
it from the KKP dashboard like the other accessible addons. The accessible addons are:

* [node-exporter](https://github.com/prometheus/node_exporter): Exports metrics from the node
* [Open Policy Agent Gatekeeper](https://github.com/open-policy-agent/gatekeeper): OPA Policy Controller for Kubernetes - (**beta**)

Accessible addons can be managed in the UI from the cluster details view:

![Addons](/img/kubermatic/master/ui/addons_overview.png?classes=shadow,border "Addons View")

#### Configuration

To configure which addons shall be accessible, set the `spec.api.accessibleAddons` field inside the
KubermaticConfiguration:

```yaml
spec:
  api:
    accessibleAddons:
      - node-exporter
```

The dashboard's view for managing addons can be configured by creating an `AddonConfig` custom resources.
This can be used to control the addon's icon, description and form fields to ask for rudimentary user
input. It's recommended to create an `AddonConfig` for each accessible addon.

{{% notice note %}}
`AddonConfig` resources are not namespaced. Make sure to not mix up addon configs if you run multiple
KKP installations in the same cluster (which is not a recommended setup).
{{% /notice %}}

The following demonstrates an AddonConfig for the node-expoter addon. It configures a description and
logo as well as some form fields (which in this case are not used by the addon itself).

```yaml
apiVersion: kubermatic.k8s.io/v1
kind: AddonConfig
metadata:
  name: node-exporter
spec:
  description: "The Prometheus Node Exporter exposes a wide variety of hardware- and kernel-related metrics."
  logoFormat: "svg+xml"
  # The logo must be base64 encoded.
  logo: |+
    PHN2ZyB3aWR0aD0iMTY2IiBoZWlnaHQ9IjQwIiB2aWV3Qm94PSIwIDAgMTY2IDQwIiBmaWxsPSJub25lIiB4bWxucz0iaHR0
    cDovL3d3dy53My5vcmcvMjAwMC9zdmciPgo8cGF0aCBkPSJNMTkuOTEyNCA1LjE3NTU0QzExLjY3NjggNS4xNzU1NCA1IDEx
    Ljg1MTYgNSAyMC4wODc0QzUgMjguMzIyOSAxMS42NzY4IDM0Ljk5OTUgMTkuOTEyNCAzNC45OTk1QzI4LjE0NzkgMzQuOTk5
    NSAzNC44MjQ1IDI4LjMyMjkgMzQuODI0NSAyMC4wODc0QzM0LjgyNDUgMTEuODUxNiAyOC4xNDc2IDUuMTc1NTQgMTkuOTEy
    NCA1LjE3NTU0VjUuMTc1NTRaTTE5LjkxMjQgMzMuMDg0N0MxNy41NjkyIDMzLjA4NDcgMTUuNjY5NSAzMS41MTk1IDE1LjY2
    OTUgMjkuNTg5MkgyNC4xNTUzQzI0LjE1NTMgMzEuNTE5MiAyMi4yNTU1IDMzLjA4NDcgMTkuOTEyNCAzMy4wODQ3Wk0yNi45
    MjAzIDI4LjQzMTZIMTIuOTAzN1YyNS44OUgyNi45MjA1VjI4LjQzMTZIMjYuOTIwM1pNMjYuODcgMjQuNTgxOUgxMi45NDM3
    QzEyLjg5NzQgMjQuNTI4NCAxMi44NSAyNC40NzU4IDEyLjgwNTMgMjQuNDIxNkMxMS4zNzA1IDIyLjY3OTUgMTEuMDMyNiAy
    MS43NyAxMC43MDQ1IDIwLjg0MzJDMTAuNjk4OSAyMC44MTI2IDEyLjQ0NDIgMjEuMTk5NyAxMy42ODE4IDIxLjQ3ODJDMTMu
    NjgxOCAyMS40NzgyIDE0LjMxODcgMjEuNjI1NSAxNS4yNDk3IDIxLjc5NTNDMTQuMzU1OCAyMC43NDc0IDEzLjgyNSAxOS40
    MTUzIDEzLjgyNSAxOC4wNTM3QzEzLjgyNSAxNS4wNjQ1IDE2LjExNzYgMTIuNDUyNCAxNS4yOTA1IDEwLjM0MTFDMTYuMDk1
    NSAxMC40MDY2IDE2Ljk1NjYgMTIuMDQgMTcuMDE0NyAxNC41OTRDMTcuODcwNSAxMy40MTEzIDE4LjIyODcgMTEuMjUxNiAx
    OC4yMjg3IDkuOTI3MzhDMTguMjI4NyA4LjU1NjMzIDE5LjEzMjEgNi45NjM3IDIwLjAzNTggNi45MDkyMkMxOS4yMzAzIDgu
    MjM2ODUgMjAuMjQ0NSA5LjM3NTAxIDIxLjE0NjEgMTIuMTk4NEMyMS40ODQyIDEzLjI1OSAyMS40NDExIDE1LjA0MzcgMjEu
    NzAyMSAxNi4xNzU1QzIxLjc4ODcgMTMuODI0NyAyMi4xOTI5IDEwLjM5NDcgMjMuNjg0MiA5LjIxMDU0QzIzLjAyNjMgMTAu
    NzAxOSAyMy43ODE2IDEyLjU2NzkgMjQuMjk4MiAxMy40NjVDMjUuMTMxNiAxNC45MTI0IDI1LjYzNjggMTYuMDA5IDI1LjYz
    NjggMTguMDgyOUMyNS42MzY4IDE5LjQ3MzQgMjUuMTIzNCAyMC43ODI2IDI0LjI1NzQgMjEuODA2MUMyNS4yNDIxIDIxLjYy
    MTMgMjUuOTIyMSAyMS40NTQ3IDI1LjkyMjEgMjEuNDU0N0wyOS4xMiAyMC44MzA4QzI5LjEyMDMgMjAuODMwNSAyOC42NTU1
    IDIyLjc0MTYgMjYuODcgMjQuNTgxOVoiIGZpbGw9IiNFNjUyMkMiLz4KPHBhdGggZD0iTTU3LjMyMDMgMjdINTUuNjI0TDQ5
    LjE4MTYgMTcuMTM4N1YyN0g0Ny40ODU0VjE0LjIwMzFINDkuMTgxNkw1NS42NDE2IDI0LjEwODRWMTQuMjAzMUg1Ny4zMjAz
    VjI3WiIgZmlsbD0iYmxhY2siLz4KPHBhdGggZD0iTTU5LjYzMTggMjIuMTU3MkM1OS42MzE4IDIxLjIyNTYgNTkuODEzNSAy
    MC4zODc3IDYwLjE3NjggMTkuNjQzNkM2MC41NDU5IDE4Ljg5OTQgNjEuMDU1NyAxOC4zMjUyIDYxLjcwNjEgMTcuOTIwOUM2
    Mi4zNjIzIDE3LjUxNjYgNjMuMTA5NCAxNy4zMTQ1IDYzLjk0NzMgMTcuMzE0NUM2NS4yNDIyIDE3LjMxNDUgNjYuMjg4MSAx
    Ny43NjI3IDY3LjA4NSAxOC42NTkyQzY3Ljg4NzcgMTkuNTU1NyA2OC4yODkxIDIwLjc0OCA2OC4yODkxIDIyLjIzNjNWMjIu
    MzUwNkM2OC4yODkxIDIzLjI3NjQgNjguMTEwNCAyNC4xMDg0IDY3Ljc1MjkgMjQuODQ2N0M2Ny40MDE0IDI1LjU3OTEgNjYu
    ODk0NSAyNi4xNTA0IDY2LjIzMjQgMjYuNTYwNUM2NS41NzYyIDI2Ljk3MDcgNjQuODIwMyAyNy4xNzU4IDYzLjk2NDggMjcu
    MTc1OEM2Mi42NzU4IDI3LjE3NTggNjEuNjI5OSAyNi43Mjc1IDYwLjgyNzEgMjUuODMxMUM2MC4wMzAzIDI0LjkzNDYgNTku
    NjMxOCAyMy43NDggNTkuNjMxOCAyMi4yNzE1VjIyLjE1NzJaTTYxLjI2NjYgMjIuMzUwNkM2MS4yNjY2IDIzLjQwNTMgNjEu
    NTA5OCAyNC4yNTIgNjEuOTk2MSAyNC44OTA2QzYyLjQ4ODMgMjUuNTI5MyA2My4xNDQ1IDI1Ljg0ODYgNjMuOTY0OCAyNS44
    NDg2QzY0Ljc5MSAyNS44NDg2IDY1LjQ0NzMgMjUuNTI2NCA2NS45MzM2IDI0Ljg4MThDNjYuNDE5OSAyNC4yMzE0IDY2LjY2
    MzEgMjMuMzIzMiA2Ni42NjMxIDIyLjE1NzJDNjYuNjYzMSAyMS4xMTQzIDY2LjQxNDEgMjAuMjcwNSA2NS45MTYgMTkuNjI2
    QzY1LjQyMzggMTguOTc1NiA2NC43Njc2IDE4LjY1MDQgNjMuOTQ3MyAxOC42NTA0QzYzLjE0NDUgMTguNjUwNCA2Mi40OTcx
    IDE4Ljk2OTcgNjIuMDA0OSAxOS42MDg0QzYxLjUxMjcgMjAuMjQ3MSA2MS4yNjY2IDIxLjE2MTEgNjEuMjY2NiAyMi4zNTA2
    WiIgZmlsbD0iYmxhY2siLz4KPHBhdGggZD0iTTY5LjkzMjYgMjIuMTY2QzY5LjkzMjYgMjAuNzA3IDcwLjI3ODMgMTkuNTM1
    MiA3MC45Njk3IDE4LjY1MDRDNzEuNjYxMSAxNy43NTk4IDcyLjU2NjQgMTcuMzE0NSA3My42ODU1IDE3LjMxNDVDNzQuNzk4
    OCAxNy4zMTQ1IDc1LjY4MDcgMTcuNjk1MyA3Ni4zMzExIDE4LjQ1N1YxMy41SDc3Ljk1N1YyN0g3Ni40NjI5TDc2LjM4Mzgg
    MjUuOTgwNUM3NS43MzM0IDI2Ljc3NzMgNzQuODI4MSAyNy4xNzU4IDczLjY2OCAyNy4xNzU4QzcyLjU2NjQgMjcuMTc1OCA3
    MS42NjcgMjYuNzI0NiA3MC45Njk3IDI1LjgyMjNDNzAuMjc4MyAyNC45MTk5IDY5LjkzMjYgMjMuNzQyMiA2OS45MzI2IDIy
    LjI4OTFWMjIuMTY2Wk03MS41NTg2IDIyLjM1MDZDNzEuNTU4NiAyMy40Mjg3IDcxLjc4MTIgMjQuMjcyNSA3Mi4yMjY2IDI0
    Ljg4MThDNzIuNjcxOSAyNS40OTEyIDczLjI4NzEgMjUuNzk1OSA3NC4wNzIzIDI1Ljc5NTlDNzUuMTAzNSAyNS43OTU5IDc1
    Ljg1NjQgMjUuMzMzIDc2LjMzMTEgMjQuNDA3MlYyMC4wMzkxQzc1Ljg0NDcgMTkuMTQyNiA3NS4wOTc3IDE4LjY5NDMgNzQu
    MDg5OCAxOC42OTQzQzczLjI5MyAxOC42OTQzIDcyLjY3MTkgMTkuMDAyIDcyLjIyNjYgMTkuNjE3MkM3MS43ODEyIDIwLjIz
    MjQgNzEuNTU4NiAyMS4xNDM2IDcxLjU1ODYgMjIuMzUwNloiIGZpbGw9ImJsYWNrIi8+CjxwYXRoIGQ9Ik04NC40MzQ2IDI3
    LjE3NThDODMuMTQ1NSAyNy4xNzU4IDgyLjA5NjcgMjYuNzUzOSA4MS4yODgxIDI1LjkxMDJDODAuNDc5NSAyNS4wNjA1IDgw
    LjA3NTIgMjMuOTI2OCA4MC4wNzUyIDIyLjUwODhWMjIuMjFDODAuMDc1MiAyMS4yNjY2IDgwLjI1MzkgMjAuNDI1OCA4MC42
    MTEzIDE5LjY4NzVDODAuOTc0NiAxOC45NDM0IDgxLjQ3ODUgMTguMzYzMyA4Mi4xMjMgMTcuOTQ3M0M4Mi43NzM0IDE3LjUy
    NTQgODMuNDc2NiAxNy4zMTQ1IDg0LjIzMjQgMTcuMzE0NUM4NS40Njg4IDE3LjMxNDUgODYuNDI5NyAxNy43MjE3IDg3LjEx
    NTIgMTguNTM2MUM4Ny44MDA4IDE5LjM1MDYgODguMTQzNiAyMC41MTY2IDg4LjE0MzYgMjIuMDM0MlYyMi43MTA5SDgxLjcw
    MTJDODEuNzI0NiAyMy42NDg0IDgxLjk5NzEgMjQuNDA3MiA4Mi41MTg2IDI0Ljk4NzNDODMuMDQ1OSAyNS41NjE1IDgzLjcx
    MzkgMjUuODQ4NiA4NC41MjI1IDI1Ljg0ODZDODUuMDk2NyAyNS44NDg2IDg1LjU4MyAyNS43MzE0IDg1Ljk4MTQgMjUuNDk3
    MUM4Ni4zNzk5IDI1LjI2MjcgODYuNzI4NSAyNC45NTIxIDg3LjAyNzMgMjQuNTY1NEw4OC4wMjA1IDI1LjMzODlDODcuMjIz
    NiAyNi41NjM1IDg2LjAyODMgMjcuMTc1OCA4NC40MzQ2IDI3LjE3NThaTTg0LjIzMjQgMTguNjUwNEM4My41NzYyIDE4LjY1
    MDQgODMuMDI1NCAxOC44OTA2IDgyLjU4MDEgMTkuMzcxMUM4Mi4xMzQ4IDE5Ljg0NTcgODEuODU5NCAyMC41MTM3IDgxLjc1
    MzkgMjEuMzc1SDg2LjUxNzZWMjEuMjUyQzg2LjQ3MDcgMjAuNDI1OCA4Ni4yNDggMTkuNzg3MSA4NS44NDk2IDE5LjMzNTlD
    ODUuNDUxMiAxOC44Nzg5IDg0LjkxMjEgMTguNjUwNCA4NC4yMzI0IDE4LjY1MDRaIiBmaWxsPSJibGFjayIvPgo8cGF0aCBk
    PSJNMTAxLjk4NiAyMS4wODVIOTYuNDQwNFYyNS42MjAxSDEwMi44ODNWMjdIOTQuNzUyOVYxNC4yMDMxSDEwMi43OTVWMTUu
    NTkxOEg5Ni40NDA0VjE5LjcwNTFIMTAxLjk4NlYyMS4wODVaIiBmaWxsPSJibGFjayIvPgo8cGF0aCBkPSJNMTA3LjkxOSAy
    MC45NjE5TDExMC4wMjggMTcuNDkwMkgxMTEuOTI3TDEwOC44MTUgMjIuMTkyNEwxMTIuMDIzIDI3SDExMC4xNDNMMTA3Ljk0
    NSAyMy40NDA0TDEwNS43NDggMjdIMTAzLjg1OEwxMDcuMDY2IDIyLjE5MjRMMTAzLjk1NSAxNy40OTAySDEwNS44MzZMMTA3
    LjkxOSAyMC45NjE5WiIgZmlsbD0iYmxhY2siLz4KPHBhdGggZD0iTTEyMS42OTEgMjIuMzUwNkMxMjEuNjkxIDIzLjc5Nzkg
    MTIxLjM2IDI0Ljk2MzkgMTIwLjY5OCAyNS44NDg2QzEyMC4wMzYgMjYuNzMzNCAxMTkuMTQgMjcuMTc1OCAxMTguMDA5IDI3
    LjE3NThDMTE2Ljg1NCAyNy4xNzU4IDExNS45NDYgMjYuODA5NiAxMTUuMjg0IDI2LjA3NzFWMzAuNjU2MkgxMTMuNjU4VjE3
    LjQ5MDJIMTE1LjE0NEwxMTUuMjIzIDE4LjU0NDlDMTE1Ljg4NSAxNy43MjQ2IDExNi44MDUgMTcuMzE0NSAxMTcuOTgyIDE3
    LjMxNDVDMTE5LjEyNSAxNy4zMTQ1IDEyMC4wMjcgMTcuNzQ1MSAxMjAuNjg5IDE4LjYwNjRDMTIxLjM1NyAxOS40Njc4IDEy
    MS42OTEgMjAuNjY2IDEyMS42OTEgMjIuMjAxMlYyMi4zNTA2Wk0xMjAuMDY1IDIyLjE2NkMxMjAuMDY1IDIxLjA5MzggMTE5
    LjgzNyAyMC4yNDcxIDExOS4zOCAxOS42MjZDMTE4LjkyMyAxOS4wMDQ5IDExOC4yOTYgMTguNjk0MyAxMTcuNDk5IDE4LjY5
    NDNDMTE2LjUxNSAxOC42OTQzIDExNS43NzYgMTkuMTMwOSAxMTUuMjg0IDIwLjAwMzlWMjQuNTQ3OUMxMTUuNzcxIDI1LjQx
    NSAxMTYuNTE1IDI1Ljg0ODYgMTE3LjUxNyAyNS44NDg2QzExOC4yOTYgMjUuODQ4NiAxMTguOTE0IDI1LjU0MSAxMTkuMzcx
    IDI0LjkyNThDMTE5LjgzNCAyNC4zMDQ3IDEyMC4wNjUgMjMuMzg0OCAxMjAuMDY1IDIyLjE2NloiIGZpbGw9ImJsYWNrIi8+
    CjxwYXRoIGQ9Ik0xMjMuMzM1IDIyLjE1NzJDMTIzLjMzNSAyMS4yMjU2IDEyMy41MTcgMjAuMzg3NyAxMjMuODggMTkuNjQz
    NkMxMjQuMjQ5IDE4Ljg5OTQgMTI0Ljc1OSAxOC4zMjUyIDEyNS40MDkgMTcuOTIwOUMxMjYuMDY1IDE3LjUxNjYgMTI2Ljgx
    MiAxNy4zMTQ1IDEyNy42NSAxNy4zMTQ1QzEyOC45NDUgMTcuMzE0NSAxMjkuOTkxIDE3Ljc2MjcgMTMwLjc4OCAxOC42NTky
    QzEzMS41OTEgMTkuNTU1NyAxMzEuOTkyIDIwLjc0OCAxMzEuOTkyIDIyLjIzNjNWMjIuMzUwNkMxMzEuOTkyIDIzLjI3NjQg
    MTMxLjgxMyAyNC4xMDg0IDEzMS40NTYgMjQuODQ2N0MxMzEuMTA0IDI1LjU3OTEgMTMwLjU5OCAyNi4xNTA0IDEyOS45MzYg
    MjYuNTYwNUMxMjkuMjc5IDI2Ljk3MDcgMTI4LjUyMyAyNy4xNzU4IDEyNy42NjggMjcuMTc1OEMxMjYuMzc5IDI3LjE3NTgg
    MTI1LjMzMyAyNi43Mjc1IDEyNC41MyAyNS44MzExQzEyMy43MzMgMjQuOTM0NiAxMjMuMzM1IDIzLjc0OCAxMjMuMzM1IDIy
    LjI3MTVWMjIuMTU3MlpNMTI0Ljk3IDIyLjM1MDZDMTI0Ljk3IDIzLjQwNTMgMTI1LjIxMyAyNC4yNTIgMTI1LjY5OSAyNC44
    OTA2QzEyNi4xOTEgMjUuNTI5MyAxMjYuODQ4IDI1Ljg0ODYgMTI3LjY2OCAyNS44NDg2QzEyOC40OTQgMjUuODQ4NiAxMjku
    MTUgMjUuNTI2NCAxMjkuNjM3IDI0Ljg4MThDMTMwLjEyMyAyNC4yMzE0IDEzMC4zNjYgMjMuMzIzMiAxMzAuMzY2IDIyLjE1
    NzJDMTMwLjM2NiAyMS4xMTQzIDEzMC4xMTcgMjAuMjcwNSAxMjkuNjE5IDE5LjYyNkMxMjkuMTI3IDE4Ljk3NTYgMTI4LjQ3
    MSAxOC42NTA0IDEyNy42NSAxOC42NTA0QzEyNi44NDggMTguNjUwNCAxMjYuMiAxOC45Njk3IDEyNS43MDggMTkuNjA4NEMx
    MjUuMjE2IDIwLjI0NzEgMTI0Ljk3IDIxLjE2MTEgMTI0Ljk3IDIyLjM1MDZaIiBmaWxsPSJibGFjayIvPgo8cGF0aCBkPSJN
    MTM4LjYyOCAxOC45NDkyQzEzOC4zODIgMTguOTA4MiAxMzguMTE1IDE4Ljg4NzcgMTM3LjgyOCAxOC44ODc3QzEzNi43NjIg
    MTguODg3NyAxMzYuMDM4IDE5LjM0MTggMTM1LjY1NyAyMC4yNVYyN0gxMzQuMDMxVjE3LjQ5MDJIMTM1LjYxM0wxMzUuNjQg
    MTguNTg4OUMxMzYuMTczIDE3LjczOTMgMTM2LjkyOSAxNy4zMTQ1IDEzNy45MDcgMTcuMzE0NUMxMzguMjI0IDE3LjMxNDUg
    MTM4LjQ2NCAxNy4zNTU1IDEzOC42MjggMTcuNDM3NVYxOC45NDkyWiIgZmlsbD0iYmxhY2siLz4KPHBhdGggZD0iTTE0Mi43
    NzYgMTUuMTg3NVYxNy40OTAySDE0NC41NTJWMTguNzQ3MUgxNDIuNzc2VjI0LjY0NDVDMTQyLjc3NiAyNS4wMjU0IDE0Mi44
    NTUgMjUuMzEyNSAxNDMuMDE0IDI1LjUwNTlDMTQzLjE3MiAyNS42OTM0IDE0My40NDEgMjUuNzg3MSAxNDMuODIyIDI1Ljc4
    NzFDMTQ0LjAxIDI1Ljc4NzEgMTQ0LjI2OCAyNS43NTIgMTQ0LjU5NiAyNS42ODE2VjI3QzE0NC4xNjggMjcuMTE3MiAxNDMu
    NzUyIDI3LjE3NTggMTQzLjM0OCAyNy4xNzU4QzE0Mi42MjEgMjcuMTc1OCAxNDIuMDczIDI2Ljk1NjEgMTQxLjcwNCAyNi41
    MTY2QzE0MS4zMzUgMjYuMDc3MSAxNDEuMTUgMjUuNDUzMSAxNDEuMTUgMjQuNjQ0NVYxOC43NDcxSDEzOS40MTlWMTcuNDkw
    MkgxNDEuMTVWMTUuMTg3NUgxNDIuNzc2WiIgZmlsbD0iYmxhY2siLz4KPHBhdGggZD0iTTE1MC40MDUgMjcuMTc1OEMxNDku
    MTE2IDI3LjE3NTggMTQ4LjA2NyAyNi43NTM5IDE0Ny4yNTkgMjUuOTEwMkMxNDYuNDUgMjUuMDYwNSAxNDYuMDQ2IDIzLjky
    NjggMTQ2LjA0NiAyMi41MDg4VjIyLjIxQzE0Ni4wNDYgMjEuMjY2NiAxNDYuMjI1IDIwLjQyNTggMTQ2LjU4MiAxOS42ODc1
    QzE0Ni45NDUgMTguOTQzNCAxNDcuNDQ5IDE4LjM2MzMgMTQ4LjA5NCAxNy45NDczQzE0OC43NDQgMTcuNTI1NCAxNDkuNDQ3
    IDE3LjMxNDUgMTUwLjIwMyAxNy4zMTQ1QzE1MS40MzkgMTcuMzE0NSAxNTIuNCAxNy43MjE3IDE1My4wODYgMTguNTM2MUMx
    NTMuNzcxIDE5LjM1MDYgMTU0LjExNCAyMC41MTY2IDE1NC4xMTQgMjIuMDM0MlYyMi43MTA5SDE0Ny42NzJDMTQ3LjY5NSAy
    My42NDg0IDE0Ny45NjggMjQuNDA3MiAxNDguNDg5IDI0Ljk4NzNDMTQ5LjAxNyAyNS41NjE1IDE0OS42ODUgMjUuODQ4NiAx
    NTAuNDkzIDI1Ljg0ODZDMTUxLjA2NyAyNS44NDg2IDE1MS41NTQgMjUuNzMxNCAxNTEuOTUyIDI1LjQ5NzFDMTUyLjM1MSAy
    NS4yNjI3IDE1Mi42OTkgMjQuOTUyMSAxNTIuOTk4IDI0LjU2NTRMMTUzLjk5MSAyNS4zMzg5QzE1My4xOTQgMjYuNTYzNSAx
    NTEuOTk5IDI3LjE3NTggMTUwLjQwNSAyNy4xNzU4Wk0xNTAuMjAzIDE4LjY1MDRDMTQ5LjU0NyAxOC42NTA0IDE0OC45OTYg
    MTguODkwNiAxNDguNTUxIDE5LjM3MTFDMTQ4LjEwNSAxOS44NDU3IDE0Ny44MyAyMC41MTM3IDE0Ny43MjUgMjEuMzc1SDE1
    Mi40ODhWMjEuMjUyQzE1Mi40NDEgMjAuNDI1OCAxNTIuMjE5IDE5Ljc4NzEgMTUxLjgyIDE5LjMzNTlDMTUxLjQyMiAxOC44
    Nzg5IDE1MC44ODMgMTguNjUwNCAxNTAuMjAzIDE4LjY1MDRaIiBmaWxsPSJibGFjayIvPgo8cGF0aCBkPSJNMTYwLjYwMSAx
    OC45NDkyQzE2MC4zNTQgMTguOTA4MiAxNjAuMDg4IDE4Ljg4NzcgMTU5LjgwMSAxOC44ODc3QzE1OC43MzQgMTguODg3NyAx
    NTguMDExIDE5LjM0MTggMTU3LjYzIDIwLjI1VjI3SDE1Ni4wMDRWMTcuNDkwMkgxNTcuNTg2TDE1Ny42MTIgMTguNTg4OUMx
    NTguMTQ2IDE3LjczOTMgMTU4LjkwMSAxNy4zMTQ1IDE1OS44OCAxNy4zMTQ1QzE2MC4xOTYgMTcuMzE0NSAxNjAuNDM3IDE3
    LjM1NTUgMTYwLjYwMSAxNy40Mzc1VjE4Ljk0OTJaIiBmaWxsPSJibGFjayIvPgo8L3N2Zz4K
  formSpec:
    - displayName: Replicas
      internalName: replicas
      required: true
      type: number
      helpText: "Number of replicas."
    - displayName: Description
      internalName: desc
      required: false
      type: text
    - displayName: Debug
      internalName: debug
      required: false
      type: boolean
    - displayName: Spec
      internalName: spec
      required: false
      type: text-area
      helpText: "Use YAML format."
```

There is a short explanation of the single `formSpec` fields:
- `displayName` is the name that is displayed in the UI as the control label.
- `internalName` is the name used internally. It can be referenced with template variables (see the description below).
- `required` indicates if the control should be required in the UI.
- `type` indicates type of the control that will be displayed in the UI. Can be one of `number`, `text`, `boolean` and
  `text-area`.
- `helpText`  is the text that will be displayed in the UI next to the control. It can contain some hints for the
   end-users.

Each of the fields from the `formSpec` is then available using the `.Variables` template variable.
See [the section below](#manifest-templating) for more information.

After applying above config the UI should look like below:

![Addon Form](/img/kubermatic/master/ui/addon_form.png?height=450px&classes=shadow,border "Addon Form")

### Custom Addons

All manifests and config files for the default addons are stored in a Docker image, whose name is configured
in the KubermaticConfiguration at `spec.userClusters.addons.kubernetes.dockerRepository` and
`spec.userClusters.addons.openshift.dockerRepository`. These default to `quay.io/kubermatic/addons` and
`quay.io/kubermatic/openshift-addons`, respectively.

#### Creating a Docker Image

Depending on the cluster orchestrator, choose the Kubernetes or OpenShift Docker image as the basis for your
own image. All addon manifest are stored in `/addons/<addonname>`, e.g. `/addons/kube-proxy`. When creating
your own image, copy the manifests into a directory below `/addons`.

Suppose you have this directory structure:

```plaintext
.
├─ Dockerfile
└─ my-custom-addon
   └─ deployment.yaml
```

A simple Dockerfile would be

```dockerfile
FROM quay.io/kubermatic/addons:YOUR_KUBERMATIC_VERSION

ADD ./ /addons/
```

You can then build and push your customized addon image to a Docker registry of your choice:

```bash
export TAG=YOUR_KUBERMATIC_VERSION
docker build -t customer/addons:${TAG} .
docker push customer/addons:${TAG}
```

#### Manifest Templating

KKP treats each file in an addon directory (`/addon/<addonname>`) as a
[Go template](https://golang.org/pkg/text/template/), so it is possible to inject dynamic and
user-cluster related information into addons at runtime. Please refer to the Go documentation for
the exact templating syntax.

KKP injects an instance of the `TemplateData` struct into each template. The following
Go snippet shows the available information:

```
{{< readfile "kubermatic/v2.17/data/addondata.go" >}}
```

KKP also injects [Sprig](http://masterminds.github.io/sprig/) functions and the following
custom functions into templates:

When referencing a Docker registry, it's recommended to use the `Registry(string) string` helper
function, which takes the registry override functionality for the KKP API into account.

```plaintext
{{ Registry "quay.io" }}/some-org/some-app:v1.0
```

The above will return `quay.io` when no override is set, otherwise the overridden registry.

#### Configure KKP

It's now time to configure KKP to use your Docker image instead of its default, and to tell it
about your new addon. By default, the KKP Operator will use the KKP version for the Docker
image tag (so if you configure `docker.io/example/my-addons` as the repository, the final image
name will be `docker.io/example/my-addons:KKP_VERSION`). To help with developing custom addons,
it is however possible to define an additional suffix for the image tag, which will be appended
to the KKP version with a dash (i.e. `docker.io/example/my-addons:KKP_VERSION-YOUR_SUFFIX`).

This versioning scheme ensures that admins can easily roll out new Docker images for their addons,
but also that updating the Docker image is not accidentally forgotten when updating KKP itself.

Edit the KubermaticConfiguration and update the Docker repository for example for Kubernetes:

```yaml
spec:
  userClusters:
    addons:
      kubernetes:
        # Do not specify a tag here, as the KKP Operator will always use the KKP
        # version instead.
        dockerRepository: docker.io/customer/addons

        # Leave this empty if you have not created your own Docker images, otherwise this
        # can be used to force pulling a new image whenever KKP is updated.
        # With this exact configuration (assuming KKP v2.15.1 is used), the final image name
        # will be "docker.io/customer/addons:v2.15.1-1".
        dockerTagSuffix: '1'
```

You also need to add your new addon to the `defaultManifests`:

```yaml
spec:
  userClusters:
    addons:
      kubernetes:
        defaultManifests: |-
          apiVersion: v1
          kind: List
          items:

          # add your new addon here
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: my-custom-addon

          # remember to keep the original default addons, or else user clusters will
          # be defunct
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: canal
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: csi
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: dns
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: kube-proxy
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: openvpn
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: rbac
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: kubelet-configmap
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: default-storage-class
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: nodelocal-dns-cache
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: pod-security-policy
          - apiVersion: kubermatic.k8s.io/v1
            kind: Addon
            metadata:
              name: logrotate
```

If you want to define the addon as accessible (i.e. configurable), add it to the `spec.api.accessibleAddons`
list.

After updating the KubermaticConfiguration, the changes will take effect after a few moments.
