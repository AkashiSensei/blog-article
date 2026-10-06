From Missing RDMA Resources to Failed CSI Mounts: Taints and Tolerations for Platform Components

I'm using the National Day holiday to write up a few issues that have been waiting in my backlog.

First, some background. Crater is an open-source AI cluster management platform we develop on top of Kubernetes. It helps users run training and inference workloads on the cluster's GPUs and other compute resources, or launch development environments such as Jupyter.

Multiple accounts share the cluster, but sometimes an account needs exclusive use of an entire compute node. We enforce this scheduling restriction by adding a `crater.raids.io/account=<account-name>:NoSchedule` taint to the node and a matching toleration to that account's jobs. These nodes also need Pods for drivers, networking, and storage plugins. If the taint blocks those Pods, it can cause failures.

I'll use two incidents to explain what happened and what we changed, without going through every similar issue.

# Symptoms

## RDMA resource discovery

RDMA provides high-speed, low-latency data transfer between machines and is commonly used in multi-node training, where nodes frequently exchange gradients.

After a node's kernel was upgraded, jobs that did not require RDMA could still run, but RDMA jobs could no longer be scheduled onto nodes that were supposed to support it.

## CSI storage mounts

We install and update Crater's frontend, backend, and storage service through Helm. After one image update, newly created storage service (`storage-server`) Pods failed to start, while older Pods continued running and serving file access.

# Investigation and analysis

First, the relationship between taints and tolerations: a node with a `NoSchedule` taint cannot accept a Pod that lacks a matching toleration. Adding that toleration to the Pod's `spec.tolerations` removes this particular scheduling barrier; node selection, affinity, and resource requirements still have to be satisfied. Also, `NoSchedule` does not directly evict Pods that are already running. See the [Kubernetes documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).

## RDMA resource discovery

During the investigation, we found that the host's network adapter and driver were working, but Kubernetes reported zero RDMA resources. The following output from `dell-gpu-06` includes only the relevant fields.

```text
feature.node.kubernetes.io/kernel-version.full=5.15.0-170-generic
network.nvidia.com/operator.mofed.wait=true
Taints:             crater.raids.io/account=q-48:NoSchedule
Unschedulable:      false
Capacity:
  rdma/rdma_a100:     0
  rdma/rdma_v100:     0
Allocatable:
  rdma/rdma_a100:     0
  rdma/rdma_v100:     0
```

RDMA was already usable on the host, but Kubernetes still needed a device plugin to report the resources so that the scheduler could know how much RDMA capacity the node had available.

In this deployment, the [OFED](https://www.openfabrics.org/ofed-for-linux/) (OpenFabrics Enterprise Distribution) driver component installs and loads the host's RDMA network adapter drivers, enabling the adapter's RDMA functionality. The RDMA shared device plugin is a Kubernetes device plugin that discovers RDMA devices and reports resources Pods can request, allowing multiple Pods to share those devices. See the [device plugin documentation](https://github.com/Mellanox/k8s-rdma-shared-dev-plugin).

These two components are deployed through separate DaemonSets. The Kubernetes DaemonSet controller creates an OFED driver Pod and an RDMA device plugin Pod on each eligible node. Both DaemonSets we checked showed **34/34 available**, but this node had neither an OFED driver Pod nor an RDMA device plugin Pod.

Below are excerpts from the two DaemonSets' `describe` output, with unrelated fields omitted. Both Pod templates had only a toleration for the GPU taint.

```text
Name:           rdma-shared-dp-ds
Desired Number of Nodes Scheduled: 34
Number of Nodes Scheduled with Available Pods: 34
  Tolerations:          nvidia.com/gpu:NoSchedule op=Exists

Name:           mofed-ubuntu22.04-76944977f-ds
Desired Number of Nodes Scheduled: 34
Number of Nodes Scheduled with Available Pods: 34
  Tolerations:          nvidia.com/gpu:NoSchedule op=Exists
```

That count can be misleading: 34/34 only means that all 34 Pods the DaemonSet considers necessary are available. It does not mean every node that needs RDMA has the plugin. Nodes excluded by taints or other conditions may never enter that count.

Our cluster's RDMA configuration follows an [article by a senior colleague in our lab](https://zhuanlan.zhihu.com/p/1897271506645530541) and uses NVIDIA Network Operator to manage OFED drivers and the RDMA shared device plugin. The Operator deploys these components according to a cluster-scoped custom resource, `NicClusterPolicy`, which specifies the drivers and device plugins to enable, Pod tolerations, and other settings. To understand how a network adapter becomes a resource the scheduler can use, we first need to distinguish node-local components, cluster-wide controllers, and the Kubernetes control plane.

| Component | Where it runs | Role in this process |
| --- | --- | --- |
| Node Feature Discovery (NFD) worker | A Pod on each node, usually deployed through a DaemonSet | Detects local network adapters, OS, and kernel information, then writes a `NodeFeature` through the API Server |
| NFD master | A cluster-wide controller Pod, deployed through a Deployment | Watches `NodeFeature` objects and updates Node labels through the API Server |
| Network Operator | A cluster-wide controller Pod, deployed through a Deployment | Reads Node labels and NicClusterPolicy, and manages the OFED driver DaemonSets and RDMA device plugin DaemonSet separately |
| DaemonSet controller | Kubernetes control plane | Creates the corresponding Pods on eligible nodes according to each DaemonSet's Pod template |
| Scheduler | A cluster-wide component; the default kube-scheduler belongs to the control plane, while a job scheduler can also be deployed separately | Checks node conditions and resources, then binds a Pod to a target node |
| OFED driver | A Pod on a target RDMA node, deployed through an OFED driver DaemonSet | Installs and loads the host's RDMA network adapter drivers |
| RDMA shared device plugin | A Pod on a target RDMA node, deployed through the RDMA device plugin DaemonSet | Discovers local RDMA devices, registers with the local kubelet, and reports device resources |
| kubelet | A host process on each node | Manages local Pods, checks readiness, calls device plugins, and reports node resources |
| Container runtime, such as containerd | A host process on each node | Creates and runs containers using the configuration provided by kubelet |
| API Server | Kubernetes control plane | Provides interfaces for reading, writing, and watching objects such as Nodes, Pods, and DaemonSets |

NFD master and Network Operator manage the whole cluster, but they are still Pods running on individual nodes. They do not have to run physically on control-plane nodes; their placement depends on the node selection and toleration settings in their deployments. The information and control flow is as follows. Kubernetes object reads and writes between components all go through the API Server.

1. The NFD worker Pod on a node detects the local network adapter, OS, and kernel information, then writes a `NodeFeature` object through the API Server. NFD master watches these objects and updates the corresponding Node object's labels through the API Server. Those labels belong to the cluster's Node record. At this point, the cluster knows the node's features; allocatable RDMA resources still have to be reported separately by the device plugin.
2. Network Operator reads Node labels and `NicClusterPolicy`, then creates and maintains the OFED driver DaemonSets and RDMA device plugin DaemonSet separately. In version 25.1.0, which we checked for this article, Network Operator groups nodes by OS name, OS version, and full kernel version. Each group has an OFED driver DaemonSet, which deploys one OFED driver Pod on each eligible node in that group. After a kernel upgrade and the resulting NFD label update, the node may match an existing OFED driver DaemonSet, or Network Operator may need to create one for the new kernel.
3. The DaemonSet controller checks node selection, taint tolerations, and other conditions against the OFED driver DaemonSet's Pod template, then creates OFED driver Pods for eligible nodes. Once the scheduler binds a Pod to its node, that node's kubelet calls the container runtime to start the OFED driver container. The container prepares the host driver and loads the kernel modules. Kubelet checks the driver container's readiness and reports the OFED driver Pod's status through the API Server.
4. Network Operator reads the driver container's Ready status in the OFED driver Pod. Once the container is ready, the Operator uses the API Server to set `network.nvidia.com/operator.mofed.wait` to `false` on that Pod's node, indicating that it no longer needs to wait for OFED readiness.
5. The RDMA device plugin DaemonSet's Pod template requires the appropriate network adapter label and a wait label of `false`. The DaemonSet controller watches Node changes through the API Server. When the wait label changes, it rechecks the RDMA device plugin's node selection and toleration conditions. If the node meets those conditions and lacks the corresponding plugin Pod, the controller creates an RDMA device plugin Pod. After the scheduler binds it to the node, the local kubelet calls the container runtime to start the plugin Pod's init container, which checks the local `mlx5_core` module. Once that check passes, the RDMA device plugin container starts.
6. The RDMA device plugin discovers local devices matching its configuration and registers its resource name and plugin interface with kubelet through a local socket. It then continuously reports the resource list and health through `ListAndWatch`. The RDMA device plugin communicates with kubelet on the same node; kubelet is responsible for reporting the resources to the cluster.
7. Kubelet updates the corresponding Node's `status.capacity` and `status.allocatable` through the API Server. Only then can `kubectl describe node` show the RDMA resources. Capacity is the total resource quantity, while Allocatable is the quantity available for allocation to Pods. The scheduler also subtracts existing Pods' resource requests to determine what remains. The shared plugin's reported resource units depend on its configuration and do not directly equal the number of physical network adapters.
8. The job scheduler reads the Node's RDMA resource status and, together with the RDMA job Pod's resource requests, node selection, and tolerations, binds the job Pod to a target node. That node's kubelet then calls the local RDMA device plugin's `Allocate` interface to obtain the device access configuration and passes it to the container runtime to start the job container.

For the transition from driver readiness to plugin deployment, see Network Operator's [wait label implementation](https://github.com/Mellanox/network-operator/blob/v25.1.0/controllers/nicclusterpolicy_controller.go#L185) and [RDMA device plugin template](https://github.com/Mellanox/network-operator/blob/v25.1.0/manifests/state-rdma-shared-device-plugin/0060_rdma-shared-dev-plugin-ds.yaml). Resource registration and allocation are described in Kubernetes' [Device Plugin documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/).

The key node selection conditions in the RDMA device plugin DaemonSet are below, under `spec.template.spec.nodeSelector`.

```yaml
nodeSelector:
  feature.node.kubernetes.io/pci-15b3.present: "true"
  network.nvidia.com/operator.mofed.wait: "false"
```

This is where the failure occurred. The target node had the account-exclusive taint `crater.raids.io/account=q-48:NoSchedule`, but both the OFED driver and RDMA device plugin DaemonSet templates tolerated only `nvidia.com/gpu:NoSchedule`, with no toleration for the account taint. The OFED driver Pod could not be deployed to the node, and the wait label was still `true` when we inspected it. The RDMA device plugin DaemonSet required that label to be `false`, so the node had no RDMA device plugin Pod either, and its RDMA resources could not be reported. Even if the wait label became `false`, the RDMA device plugin Pod would still need a matching toleration for the account taint.

A DaemonSet cannot simply run on every node. Kubernetes automatically adds some system tolerations to its Pods, such as `node.kubernetes.io/unschedulable`, but it does not automatically tolerate our account taint. The node's `Unschedulable` field was already `false`, so cordoning was not the cause here. See the [DaemonSet documentation](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#taints-and-tolerations).

The incident notes also recorded that DKMS had rebuilt the host driver for the new kernel. This explains why RDMA had recovered on the host, but it does not replace Network Operator's readiness check on the OFED driver Pod or report resources on behalf of the RDMA device plugin. The retained output does not capture the complete history of label changes. We can confirm that the wait label was `true` at the time, but cannot conclude that every kernel upgrade immediately sets it to `true`.

## CSI storage mounts

The second incident occurred in a Crater cluster using Ceph as its storage backend. Ceph is a distributed storage system that stores data across multiple machines, provides redundancy and fault tolerance through mechanisms such as replication, and supports file, block, and object storage. [CephFS](https://docs.ceph.com/en/latest/cephfs/) is Ceph's shared filesystem, allowing multiple nodes to mount, read, and write the same files. It was the filesystem used by storage-server in this incident.

Integrating Ceph with Kubernetes also involves Rook and Ceph CSI. Rook deploys and manages the storage components. CSI is the interface Kubernetes defines for storage plugins, and Ceph CSI implements it so that Kubernetes can create, mount, and unmount Ceph volumes. The components in this deployment had the following roles.

| Component | Role and deployment method |
| --- | --- |
| [Rook Operator](https://rook.io/docs/rook/latest/Getting-Started/intro/) | Installed independently by applying YAML; its Pod is managed by a Deployment. Reads storage configuration and manages Ceph services, Ceph CSI controllers, and node plugins |
| Ceph CSI controller | Its Deployment is created and maintained by Rook. Handles cluster-wide operations such as volume creation; this incident involved the CephFS driver |
| Ceph CSI node plugin | Its DaemonSet is created and maintained by Rook. Runs on nodes that need storage access, registers the driver with the local kubelet, and mounts or unmounts volumes at kubelet's request; here, these were the `csi-cephfsplugin` Pods |
| storage-server | Crater's file access service, deployed through a Deployment. Provides users with file access through an already-mounted CephFS filesystem |

This deployment placed storage-server on three control-plane nodes, sharing those nodes with Kubernetes control-plane components. Those nodes therefore also needed CephFS CSI node plugins.

Checking the Pod states showed that two old storage-server Pods created on March 7, 2026 were still `Running / Ready`. New Pods created on July 19 had been scheduled onto control-plane nodes but were stuck in `ContainerCreating`, with the following mount error in their events.

```text
MountVolume.MountDevice failed ... driver name
rook-ceph.cephfs.csi.ceph.com not found in the list of registered CSI drivers
```

The PVC was already `Bound`, and VolumeAttachment showed `attached=true`, but kubelet failed while preparing the volume mount, before the container started. Successful volume binding and attachment do not guarantee that the target node can mount the volume.

The control-plane nodes had the taint `node-role.kubernetes.io/control-plane:NoSchedule`. Storage-server had a matching toleration and could be scheduled there. However, the CephFS CSI DaemonSet managed by Rook lacked that toleration. None of the three nodes had a `csi-cephfsplugin` Pod, and their `CSINode` objects, which record node storage drivers, did not list the CephFS driver either.

Even after the Ceph CSI controller has prepared the volume and the PVC is bound, the actual mount still requires a `csi-cephfsplugin` Pod on the target node. The registrar container in that Pod registers the driver with the local kubelet, which then calls the CephFS CSI driver in the same Pod to perform the mount. A `csi-cephfsplugin` Pod on another node cannot register the driver for this node. See the [CSI registrar documentation](https://kubernetes-csi.github.io/docs/node-driver-registrar.html).

The old Pods used CephFS kernel mounts that had already been established. Subsequent file reads and writes were handled by the kernel client, without going through a CSI Pod, so existing mounts could continue working even with the plugin absent. The failure became visible only when a new Pod, a migrated Pod, or a node reboot required a new mount.

The CephFS CSI DaemonSet's saved revisions (ControllerRevision) also showed the following configuration changes.

| Time (Asia/Shanghai) | Change |
| --- | --- |
| 2026-03-07 15:45:45 | Revision 16 added the control-plane NoSchedule toleration |
| 2026-03-09 22:11:41 | Revision 17 reverted to a template without that toleration |
| 2026-07-19 | New storage-server Pods exposed the mount failure |

The revisions confirm that the toleration had been added and later removed. Without audit records, however, we cannot determine whether Rook regenerated the DaemonSet from an older configuration, or someone rolled it back or reapplied an older configuration manually.

# Configuring the fix

Both incidents required adding tolerations to the component Pods that the workloads depended on. Where to make that change depends on who manages those Pods and how their configuration is generated. The source may be Helm values, a custom resource (CR), or a ConfigMap; a field name alone does not tell us which Pod it affects.

Kubernetes checks the actual Pod's `spec.tolerations`. Objects such as Deployments and DaemonSets store tolerations in their Pod templates at `spec.template.spec.tolerations`, and newly created Pods receive those settings from the template. There is no automatic inheritance between different Pods: a job Pod's tolerations do not propagate to driver or storage plugin Pods, and an Operator Pod's tolerations do not propagate to the component Pods it manages.

A ConfigMap stores key-value configuration in Kubernetes; it has no taint toleration or scheduling behavior of its own. When an Operator reads a CR or ConfigMap and writes tolerations into a component's DaemonSet template, that configuration flow is implemented by the Operator. Whether it rereads an updated ConfigMap, and which objects it updates, also depends on the component using it. We therefore need to find the configuration source the component actually reads and confirm that the change reaches the target Pod's tolerations.

Several approaches can get these component Pods running, but we also need to consider how long the change lasts, whether it survives component recreation or upgrades, and how much maintenance it requires from people deploying Crater outside our own cluster.

| Approach | Advantage | Limitation |
| --- | --- | --- |
| Remove the node taint | Quickly removes this scheduling restriction | Also removes the scheduling restriction used for account exclusivity or control-plane nodes |
| Edit the component DaemonSet's tolerations directly | A narrowly scoped change, useful for testing an emergency fix | The Operator may overwrite it, and a new DaemonSet for a new kernel will not automatically receive the change |
| Patch the CR or ConfigMap that the Operator reads in the live cluster | Changes the Operator's configuration source, so later reconciliation can retain and apply the tolerations | If the source files are not updated as well, redeployment or configuration synchronization may still overwrite it |
| Update the CR, ConfigMap, or relevant Helm values in the deployment source files | Reusable configuration that can continue generating the correct Pod templates after redeployment or component recreation | Requires identifying the actual configuration source and checking fields and update behavior when upgrading components |
| Have Crater automatically check or modify third-party component configuration | Reduces the investigation and manual changes required from external deployers | Requires support for different installation methods and versions; automatic changes may conflict with administrators' configuration management and trigger driver or CSI updates |

We kept the node taints and added tolerations with a specific key and effect only to components that needed to run on those nodes. Persistent configuration can survive Pod recreation and Operator reconciliation, but must also be retained in future deployments. Whether Crater should take over third-party component configuration needs to be considered separately for each case.

For an open-source project, we would naturally prefer to fix issues like these in code or Helm so that external users do not encounter them in the first place. These two cases are harder to handle that way with our current architecture. RDMA already requires cluster administrators to configure it for their own environment; Crater integrates with that setup rather than configuring it. Ceph is also optional and is not installed alongside Crater through Helm. We support different storage backends, and NFS may well be more common among external users. As a result, both fixes involve fairly manual configuration.

## RDMA resource discovery

Restoring RDMA resources on the target node required both the OFED driver Pod and the RDMA device plugin Pod to tolerate the account-exclusive taint. We deploy Network Operator through Helm, but manage `NicClusterPolicy` separately in YAML. These two configuration sources affect different Pods.

| Configuration source | Affected object | How the configuration is applied |
| --- | --- | --- |
| Helm `operator.tolerations` | Network Operator's own Deployment | Helm renders the Deployment's Pod template, which is used to create Network Operator Pods |
| NicClusterPolicy `spec.tolerations` | DaemonSets for the OFED driver, RDMA device plugin, and other managed components | Network Operator reads the CR and writes the settings into each component's Pod template |

We therefore added the account taint toleration to the administrator-maintained `NicClusterPolicy` source YAML, preserving its existing entries. RDMA resource names, device selection, and other parameters were already configured there. Keeping the toleration in the same source also allows Network Operator to apply it during reconciliation to future OFED driver DaemonSets for new kernels, without patching each generated DaemonSet individually. Changing only Helm's `operator.tolerations` would affect only Network Operator Pods and would not fix deployment of the OFED driver or RDMA device plugin.

The object being changed here is a `NicClusterPolicy` resource instance, or CR. The CRD defines the fields this resource type supports; it is not the configuration that needed changing in this incident.

After the change is applied, Network Operator reads `NicClusterPolicy` and updates `spec.template.spec.tolerations` in the OFED driver DaemonSets and RDMA device plugin DaemonSet. The DaemonSet controller rechecks node eligibility and creates the corresponding component Pods on nodes that were previously excluded by the account taint and now meet the conditions. Network Operator continues maintaining these DaemonSets according to the CR, so direct edits to the generated DaemonSet templates may later be overwritten.

Updating a template also needs to be distinguished from updating existing Pods. The OFED driver DaemonSet in version 25.1.0 uses the `OnDelete` strategy, so a template change does not automatically replace old OFED driver Pods; replacing them depends on deletion or the component's upgrade logic. In this incident, the target node had no OFED driver Pod, so the controller could create one once the node met the new conditions. See the [OFED update strategy](https://github.com/Mellanox/network-operator/blob/v25.1.0/manifests/state-ofed-driver/0050_ofed-driver-ds.yaml#L23).

For the account taint toleration, we used the key `crater.raids.io/account`, the `Exists` operator to match any account value, and the effect `NoSchedule`. OFED drivers and RDMA device plugins need to serve exclusive nodes belonging to different accounts, so hardcoding an account name would not work well. To allow only one account, `Equal` could instead be used with a specific value. Node selection conditions for network adapters, kernel versions, and other features still apply.

Patching the `NicClusterPolicy` CR is also an option in an emergency, but JSON merge patch replaces the entire tolerations array, so existing entries must not be lost.

For external deployers, we documented how to locate, modify, and verify the configuration in the public [RDMA support documentation](https://raids-lab.github.io/crater/zh/docs/admin/more/rdma/), including the section on tolerating Crater taints in RDMA-related utility Pods. This makes it possible to prepare the configuration when enabling account-exclusive nodes, rather than discovering the omission after a node upgrade or driver recreation.

## CSI storage mounts

For new storage-server Pods to mount their volumes, all three control-plane nodes first needed CephFS CSI node plugins. Storage-server already tolerated the control-plane taint. The missing toleration belonged to the `csi-cephfsplugin` Pods, so changing storage-server's configuration would not resolve the failure.

Moving storage-server to nodes that already had CephFS CSI plugins could also unblock the mounts, but would change the service's placement, and other nodes that might run storage workloads would still need checking. Since this cluster already placed storage-server on control-plane nodes, we chose to deploy the missing CephFS CSI plugins there.

Rook v1.17.4 in this cluster is infrastructure we install and maintain independently, directly through YAML, rather than through the Crater Helm Chart. Crater does not bundle Ceph; it uses a StorageClass or existing PVC to connect to storage, and can work with CephFS, NFS, or other backends that support shared read-write access. The CephFS CSI DaemonSet's tolerations therefore needed to be configured through `CSI_CEPHFS_PLUGIN_TOLERATIONS` in the `rook-ceph/rook-ceph-operator-config` ConfigMap that Rook actually reads, with the change also saved in Rook's deployment source files.

In this deployment, Rook handles configuration changes in that ConfigMap, reads the toleration list in `CSI_CEPHFS_PLUGIN_TOLERATIONS`, and writes it into the CephFS CSI DaemonSet's `spec.template.spec.tolerations`. The DaemonSet controller then creates `csi-cephfsplugin` Pods according to the new template. This configuration flow is implemented by Rook; CephFS CSI node plugins do not inherit tolerations from storage-server Pods or Rook's own Pod.

We used the CephFS-specific field to avoid also changing the RBD plugin's node coverage. This version prefers the plugin-specific list and falls back to the generic `CSI_PLUGIN_TOLERATIONS` only when the specific list is not configured. It does not automatically merge the two lists. If generic tolerations were already configured, any entries still needed by CephFS must also be included in the specific list. See the [Rook configuration implementation](https://github.com/rook/rook/blob/v1.17.4/pkg/operator/ceph/csi/spec.go#L504).

The CephFS CSI DaemonSet used `RollingUpdate`, so changing its template caused the DaemonSet controller to gradually replace old plugin Pods and create plugin Pods on newly eligible control-plane nodes. The scheduler and kubelet on each target node then handled binding and startup. Verification therefore needed to cover the tolerations in the ConfigMap, the CephFS CSI DaemonSet template, and the actual plugin Pods, as well as completion of the rollout. Editing the CephFS CSI DaemonSet directly might work temporarily, but Rook could overwrite that change when maintaining it from the ConfigMap.

Changing tolerations can trigger a rollout of CSI plugins on existing nodes too, so storage health and the permitted number of unavailable plugin Pods need to be checked before applying the change. In this case, `maxUnavailable=1` allowed at most one unavailable Pod. When one node became stuck because of missing image content, the remaining rollout also stopped, so successful application of the configuration was not enough: we still had to confirm that the full rollout completed.

For external deployers, Crater connects to existing storage, while Rook or other CSI components remain managed through their own deployment configuration. Our public [storage architecture documentation](https://raids-lab.github.io/crater/zh/docs/admin/deployment/storage/) includes troubleshooting and persistent configuration requirements for CephFS CSI node coverage, helping deployers identify the correct configuration source.

# Final solution

## RDMA resource discovery

We added the account taint toleration to `spec.tolerations` in `nic-cluster-policy`, preserving existing fields and tolerations. The relevant configuration is below.

```yaml
spec:
  tolerations:
    - key: crater.raids.io/account
      operator: Exists
      effect: NoSchedule
```

After we applied the configuration, Network Operator maintained the OFED driver and RDMA device plugin DaemonSets according to the updated `NicClusterPolicy`.

## CSI storage mounts

We kept storage-server on the control-plane nodes and added the following field to the `rook-ceph/rook-ceph-operator-config` ConfigMap, saving the change in Rook's deployment source files as well. When merging the configuration, we preserved the other tolerations still needed by the CephFS node plugin.

```yaml
data:
  CSI_CEPHFS_PLUGIN_TOLERATIONS: |
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
```

After checking the change with a server-side dry run and diff, we applied the ConfigMap to the cluster on July 20. Rook then automatically updated the CephFS CSI DaemonSet.

# Verification

After the fixes, our main verification was whether new jobs could be scheduled and new service Pods could start. We also checked the component states on the nodes to confirm that the configuration had taken effect.

## RDMA resource discovery

We compared the tolerations in `NicClusterPolicy`, the OFED driver and RDMA device plugin DaemonSet templates, and the actual Pods. We also used the following commands to inspect the components and resource state on `dell-gpu-06`.

```bash
kubectl describe node dell-gpu-06
kubectl -n nvidia-network-operator get pods -o wide --field-selector spec.nodeName=dell-gpu-06
```

These queries checked the target node's OFED driver Pod and RDMA device plugin Pod, driver readiness, the `network.nvidia.com/operator.mofed.wait` label, and RDMA resources in Capacity / Allocatable. We compared those results with actual job scheduling.

After applying the tolerations in `NicClusterPolicy`, RDMA-enabled jobs could be scheduled onto the corresponding nodes again. The earlier behavior—jobs could be scheduled without RDMA but could not be scheduled with it—no longer occurred. This confirmed that the target nodes once again met the scheduling requirements for RDMA jobs and that the missing tolerations were no longer blocking component deployment and resource discovery.

## CSI storage mounts

We compared the tolerations in the Rook ConfigMap, the CephFS CSI DaemonSet template, and the actual plugin Pods, then queried the plugin Pods and driver registrations on the control-plane nodes.

```bash
kubectl -n rook-ceph get pods -o wide --field-selector spec.nodeName=<node>
kubectl get csinode <node> -o yaml
```

After we applied the Rook ConfigMap, all three control-plane nodes had CephFS CSI node plugins, and their `CSINode` objects listed `rook-ceph.cephfs.csi.ceph.com`. The CephFS CSI DaemonSet eventually completed the update of all 31/31 Pods.

The new storage-server Pods that had been stuck in `ContainerCreating` then completed their mounts automatically and started successfully on the control-plane nodes. We did not manually delete or restart these application Pods. This verified that new Pods could establish new mounts, rather than merely showing that old Pods could keep using existing mounts, and confirmed that the configuration had resolved the missing CSI driver on the nodes.

These incidents reminded me that configuring tolerations for jobs or platform services also means checking whether their drivers and plugins can run on the same nodes. An old Pod may work only because an earlier driver or mount is still usable. New Pods need to start and use those resources successfully after upgrades or recreation before we can be confident that the configuration works.
