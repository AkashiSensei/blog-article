> 历史材料快照，收集于 2026-10-01。来源：[README.md](</Users/liuyizhou/develop/crater/operations/act-gpu-cluster/crater/crater-helm-charts/archive/2026-07-20-storage-server-cephfs-csi-incident/README.md>)。
> 正文保留原记录；术语与证据边界的校正见 [素材索引](../materials.zh-CN.md)。复盘中的配置文件链接已改为本地原文件。

# 小集群 storage-server 因 CephFS CSI 缺失无法启动的问题记录

> 记录日期：2026-07-20
>
> 状态：已修复；CephFS CSI DaemonSet 已完成 31/31 更新
>
> 影响范围：小集群 `crater-workspace` 命名空间中的 storage-server
>
> 变更记录：初始调查均为只读操作；获得明确授权后于 2026-07-20 19:35 应用 ConfigMap，20:11 完成 CSI rollout；未手动重启或删除业务 Pod

## 1. 问题摘要

小集群更新 Crater 后，storage-server Deployment 新创建的 Pod `webdav-deployment-54bf67b97-655mx` 一直处于 `ContainerCreating`。直接报错是：

```text
MountVolume.MountDevice failed ... driver name
rook-ceph.cephfs.csi.ceph.com not found in the list of registered CSI drivers
```

这不是 storage-server 镜像启动后的应用错误。容器尚未启动，失败发生在 kubelet 为容器准备 `/crater` 存储卷的阶段。

storage-server 被固定调度到控制平面节点，而 Rook 管理的 `csi-cephfsplugin` DaemonSet 当前没有容忍控制平面节点的 `NoSchedule` 污点。因此，三个控制平面节点均没有 CephFS CSI 节点插件，kubelet 也没有注册到 `rook-ceph.cephfs.csi.ceph.com` 驱动，导致新 Pod 无法完成 CephFS 挂载。

两只 2026-03-07 创建的旧 storage-server Pod 仍然正常运行，是因为它们使用的 CephFS 内核挂载已在 CSI 插件仍能运行于控制平面节点时建立。CSI 插件不在应用读写数据的持续数据路径中；插件消失后，已经存在的 Linux 内核 CephFS 挂载仍可继续工作。但是，一旦旧 Pod 被重建、迁移，节点重启，或者 kubelet 需要重新挂载，同样会失败。

## 2. 故障发生时的现象

### 2.1 storage-server Pod

| Pod | 创建时间（Asia/Shanghai） | 节点 | 状态 |
| --- | --- | --- | --- |
| `webdav-deployment-575b9c7dc9-4wkkr` | 2026-03-07 15:38:20 | `dell-78` | `Running`、`Ready` |
| `webdav-deployment-575b9c7dc9-br67s` | 2026-03-07 15:45:58 | `dell-80` | `Running`、`Ready` |
| `webdav-deployment-54bf67b97-655mx` | 2026-07-19 20:00:07 | `dell-79` | `Pending`、`ContainerCreating` |

新 Pod 已经被调度到 `dell-79`，但 storage-server 容器从未真正启动：没有有效的 container ID 和 image ID。即使 `latest` 镜像内容在此期间发生了变化，当前故障也发生在拉起容器之前，不能归因于新镜像中的应用代码。

检查 Crater Helm 修订 67 和 68 后，storage-server Deployment 的声明没有变化。本次 rollout 创建新 Pod，只是暴露了此前已经存在的节点存储条件问题。

### 2.2 PVC、PV 和 VolumeAttachment

- storage-server 将 PVC `crater-storage` 挂载到容器内的 `/crater`。
- PVC 位于 `crater-workspace` 命名空间，状态为 `Bound`。
- PVC 使用 StorageClass `rook-cephfs`，容量为 8 TiB，访问模式为 RWX。
- 对应 PV 为 `pvc-cb098421-2fe8-4268-ad26-76faf994d4cb`。
- 面向 `dell-79` 的 VolumeAttachment 已存在，且显示 `attached=true`。

`Bound` 和 `attached=true` 只说明控制面上的声明绑定与附件状态已经建立，并不等于目标节点已经成功挂载文件系统。实际挂载仍需要目标节点上的 CSI 节点插件与 kubelet 协作完成。

### 2.3 控制平面节点与 CSI

storage-server 被调度到以下控制平面节点：

- `dell-78`
- `dell-79`
- `dell-80`

三个节点都带有：

```yaml
key: node-role.kubernetes.io/control-plane
effect: NoSchedule
```

storage-server Deployment 自身具有对应的 node selector 和 toleration，因此能被调度到这些节点。故障发生时，Rook `csi-cephfsplugin` DaemonSet 没有该 toleration，所以不会在这些节点创建 CSI 插件 Pod。

故障发生时，三个控制平面节点的 `CSINode` 对象都只列出 `fuse.csi.fluid.io`，没有列出：

```text
rook-ceph.cephfs.csi.ceph.com
```

这与新 Pod 事件中的“driver not found in the list of registered CSI drivers”完全一致。

## 3. 相关组件是什么，它们如何协作

| 组件 | 简单理解 | 本问题中的对象 |
| --- | --- | --- |
| CephFS | 真正保存文件的分布式文件系统 | storage-server 的数据最终保存在这里 |
| Rook | Kubernetes 中管理 Ceph 和 Ceph CSI 组件的 Operator | `rook-ceph-operator` |
| Ceph CSI | 帮助 Kubernetes 创建、挂载和卸载 CephFS 的存储驱动 | `rook-ceph.cephfs.csi.ceph.com` |
| PV | Kubernetes 对一份实际持久存储的记录 | `pvc-cb098421-2fe8-4268-ad26-76faf994d4cb` |
| PVC | Pod 对持久存储的申请，Pod 通常直接引用它 | `crater-workspace/crater-storage` |

Rook、Ceph CSI 和 CephFS 不是同一个组件。CephFS 负责保存数据；Ceph CSI 负责为 Kubernetes 建立到 CephFS 的挂载；Rook 负责部署和持续管理 Ceph 及 Ceph CSI 的 Pod。由于 Rook Operator 会持续让实际配置回到期望状态，直接修改 CSI DaemonSet 可能被它再次覆盖。

PV 和 PVC 也不保存业务数据。PVC 可以理解为 Pod 提交的“存储申请”，PV 是 Kubernetes 为这份申请找到的“存储记录”。PVC 与 PV 绑定后会显示 `Bound`，但这只表示存储已经分配，不表示它已经挂载到 Pod 所在节点。

storage-server 使用存储的关系可以简化为：

```text
storage-server Pod
  └─ PVC crater-storage：Pod 的存储申请
      └─ PV pvc-cb098421-...：对应的存储记录
          └─ 节点上的 Ceph CSI 插件：负责挂载
              └─ CephFS：真正保存数据

Rook Operator
  └─ 部署和管理 Ceph CSI 插件
```

Ceph CSI 包含控制器侧组件和节点侧插件。控制器侧当时已经完成了存储分配，所以 PVC 是 `Bound`，VolumeAttachment 也是 `attached=true`；但是 `dell-79` 上没有节点侧 `csi-cephfsplugin`，因此 kubelet 无法把 CephFS 挂载到 storage-server 的 `/crater`。这就是 Pod 停留在 `ContainerCreating` 的原因。

相关基础资料：

- [Rook 项目简介](https://rook.io/docs/rook/latest-release/Getting-Started/intro/)
- [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Kubernetes CSI Developer Documentation](https://kubernetes-csi.github.io/docs/)
- [Rook Ceph CSI 常见问题排查](https://rook.io/docs/rook/v1.17/Troubleshooting/ceph-csi-common-issues/)

## 4. 为什么旧 Pod 没有 CSI 插件仍能继续运行

这里需要区分“建立或管理挂载”与“应用持续读写数据”。

本集群配置了：

```text
CSI_FORCE_CEPHFS_KERNEL_CLIENT=true
```

在旧 Pod 启动时，节点上的 CSI 插件曾调用内核 CephFS 客户端建立挂载，并由 kubelet 将该节点挂载 bind mount 到容器中。挂载成功后，应用的正常读写路径是：

```text
storage-server 进程
  └─ 容器内 /crater
      └─ 节点上已经存在的 CephFS 内核挂载
          └─ Ceph MON / MDS / OSD
```

CSI Pod 本身不转发每一次文件读写，也不是持续位于这条数据路径中的代理。因此，即使 CSI Pod 随后消失，只要：

- 节点没有重启；
- 已建立的内核挂载仍然有效；
- 旧 Pod 没有被重建或迁移；
- Ceph 集群和网络仍可访问；

旧容器就可能继续通过已经存在的挂载访问 CephFS。

这不表示 CSI 已经不再需要。缺少节点插件后，下列生命周期操作无法可靠完成：

- 在新节点或当前节点上为新 Pod 建立挂载；
- Pod 重建或迁移后的重新挂载；
- 节点重启后的挂载恢复；
- 挂载失效后的重新发布；
- 正常的卸载、清理等节点侧 CSI 操作。

`VolumeAttachment` 对象也不会承载数据或维持内核挂载；它只是 Kubernetes 控制面记录的期望状态和附件状态。

所以，旧 Pod 正常只能证明其挂载在过去已经成功建立，不能证明当前存储链路健康。即使只是重启旧 Pod 内的容器，能否复用现有挂载也取决于 kubelet 和节点挂载状态；不应将其视为安全操作。删除或重建旧 Pod 的风险更明确：在 CSI 恢复之前，新 Pod 很可能无法挂载。

## 5. CSI 容忍配置的变化时间线

Rook CephFS CSI DaemonSet 的 ControllerRevision 记录显示：

| 修订 | 发生时间（Asia/Shanghai） | 变化 |
| --- | --- | --- |
| 原模板 `csi-cephfsplugin-85664db4d4` | 2026-02-28 18:41:58 创建 | 没有控制平面 toleration |
| 修订 15 `csi-cephfsplugin-5c49c67b` | 2026-03-07 15:37:07 | 仅增加 rollout restart 时间注解，仍无 toleration |
| 修订 16 `csi-cephfsplugin-f7d47ddb5` | 2026-03-07 15:45:45 | 增加控制平面 `NoSchedule` toleration |
| 故障发生时的修订 17，复用 `85664db4d4` 模板 | 2026-03-09 22:11:41 更新；22:13:08 出现该 hash 的首个 CSI Pod | 删除 restart 注解和控制平面 toleration |

修订 16 相对于修订 15 唯一有实质意义的 Pod 模板变化是：

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
```

故障发生时的修订 17 又回到了原先没有 toleration 的模板。它复用了已有 ControllerRevision，所以其 ControllerRevision 对象的创建时间早于本次回退时间；不能把对象创建时间误认为修订 17 实际生效的时间。这是 DaemonSet 回滚可以复用旧 ControllerRevision 的正常行为。

修订 16 生效于 15:45:45，而第二只旧 storage-server Pod 创建于 15:45:58。这个时间关系与“CSI 插件短暂进入控制平面节点并完成挂载”一致。第一只旧 Pod 创建于 15:38:20，早于修订 16；现存对象不足以还原它当时具体通过哪一次 CSI Pod 或临时操作完成挂载，不能仅凭现有证据下更精确的结论。

参考：[回滚 DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/rollback-daemon-set/)

## 6. Rook 配置来源和根因判断

### 6.1 当前安装方式

当前 Rook Operator 不是由 Helm release 管理：

- 集群中没有 Rook Helm release；
- 没有对应的 Helm owner Secret 或 ConfigMap；
- 资源没有 Helm 管理标签和注解；
- managed fields 显示曾使用 `kubectl create`、client-side apply、`edit`、`set` 和 `rollout`。

Operator 最初的 last-applied 配置使用 Rook v1.12.4，后续 ReplicaSet 历史显示依次升级过 v1.13.8、v1.14.3、v1.15.6、v1.16.8，当前为 v1.17.4。

仓库内 `rook-ceph/` 记录的是 Helm、外部 Ceph 和 Rook v1.15.6，与线上实际的原始 YAML 管理、集群内 Ceph 和 v1.17.4 不一致，不能直接用仓库中的 Helm 配置覆盖当前 Rook。

### 6.2 持久配置与直接原因

Rook v1.17 读取的 CSI 配置来自 `ConfigMap/rook-ceph-operator-config`。修复前其中既没有 `CSI_PLUGIN_TOLERATIONS`，也没有 `CSI_CEPHFS_PLUGIN_TOLERATIONS`。Operator 日志表明该 ConfigMap controller 仍在正常执行 reconcile。

因此，目前可以分层描述根因：

1. **直接原因**：控制平面节点没有注册 CephFS CSI 驱动，kubelet 无法完成新 storage-server Pod 的卷挂载。
2. **配置原因**：CephFS CSI DaemonSet 没有容忍控制平面节点污点，插件不能在 storage-server 所在节点运行。
3. **持久化原因**：Rook Operator 的持久 ConfigMap 中未声明该 toleration。修订 16 的 toleration 很可能是一次直接修改 DaemonSet 的临时变更，随后被 Operator reconcile 回持久配置；显式 rollback 或重新 apply 旧模板也能产生相同结果。

第 3 点中的具体触发动作属于推断。API Server 没有启用可用的审计参数，历史 Event 已经过期，资源也没有 change-cause，因此当前证据无法确定 2026-03-09 22:11 究竟由哪个人或哪个命令触发了修订 17。

## 7. 当前 Ceph 与版本风险

调查时 Ceph 状态为 `HEALTH_WARN`：

- 16/16 OSD `up` 且 `in`；
- `osd.1` 有 BlueStore slow operation 告警；
- 25 个 PG 正在 remapped/backfill，约 5% 对象 misplaced；
- 集群正在恢复；
- CephFS MDS 有 active 和 standby-replay。

因此，即便 CSI 修复路径已经较明确，也不宜在 Ceph 恢复未完成时贸然触发大范围 CSI 滚动更新。

另外，当前版本组合为：

- Kubernetes Server v1.35.2；
- 日常检查所用 kubectl v1.32.2；
- Rook v1.17.4；
- Ceph v19.2.2 Squid。

kubectl 与 Server 相差三个 minor 版本，超过 Kubernetes 官方支持的一个 minor 版本偏差。Rook v1.17 的官方支持范围为 Kubernetes 1.28–1.33；Rook v1.19 支持 Kubernetes 1.30–1.35 和 Ceph Squid。版本不匹配不是此次“驱动未注册”的直接原因，但会增加后续运维与升级风险。

参考：

- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Rook Maintenance and Support](https://rook.io/docs/rook/latest-release/Getting-Started/maintenance-and-support/)

## 8. 修复实施结果与过程中问题

应用前先将当前 ConfigMap 下载并整理为声明式文件 [`rook-ceph-operator-config.yaml`](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/crater/crater-helm-charts/archive/2026-07-20-storage-server-cephfs-csi-incident/rook-ceph-operator-config.yaml)，然后仅增加 CephFS 专用 toleration：

```yaml
data:
  CSI_CEPHFS_PLUGIN_TOLERATIONS: |
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
```

选择 CephFS 专用配置，是为了避免全局 `CSI_PLUGIN_TOLERATIONS` 同时影响 RBD 插件。服务端 dry-run 和 `kubectl diff` 均确认只新增这一项后，于 2026-07-20 19:35 应用该 ConfigMap。

Rook Operator 随后自动读取配置并更新 `csi-cephfsplugin` DaemonSet，不需要手动重启 Operator、CSI 或 storage-server。修复结果如下：

- `dell-78`、`dell-79`、`dell-80` 均启动了 CephFS CSI Pod。
- 三个控制平面节点的 `CSINode` 均注册了 `rook-ceph.cephfs.csi.ceph.com`。
- Pending 的 storage-server Pod 自动完成挂载并启动。
- storage-server Deployment 自动完成此前的 rollout，两个副本均为 `Running`、`Ready`。
- `/crater` 已确认挂载为 CephFS。

### 8.1 `dell-63` containerd 镜像异常

DaemonSet 使用 `RollingUpdate` 且 `maxUnavailable=1`，因此在新增三个控制平面 Pod 后继续依次替换工作节点上的旧 CSI Pod。更新进行到 11/31 时，在 `dell-63` 阻塞：

```text
CreateContainerError
error unpacking image: failed to get reader from content store:
content digest ... not found
```

`csi-node-driver-registrar:v2.10.0` 和 `cephcsi:v3.10.2` 都出现了本地 layer 缺失。`dell-63` 本身仍为 `Ready`，没有磁盘、内存或 PID 压力，因此问题指向该节点的 containerd 本地镜像内容存储，而不是本次 toleration 配置。

由于 `maxUnavailable=1`，其余节点当时没有继续被替换，状态停在 `desired=31`、`updated=11`、`ready=30`、`unavailable=1`。`dell-63` 当时没有注册 CephFS CSI 驱动，但其余节点上的 CSI Pod 继续保持可用。

维护者随后在 `dell-63` 上对两个镜像分别执行了定向 `crictl pull`。命令返回 `Image is up to date`，没有明确显示 layer 下载过程；但在执行后，kubelet 的下一次容器创建重试成功，CSI Pod 恢复为 `3/3 Running`，`dell-63` 重新注册 CephFS CSI 驱动，DaemonSet 也自动继续剩余更新。因此可以确认故障已消除，但仅凭现有信息不能严格区分是 `crictl pull` 补齐或重新校验了内容，还是其执行时机恰好与 kubelet 自动重试重合。

本次没有清理镜像缓存、删除故障 Pod、修改镜像拉取策略或重启 containerd。

## 9. 最终验证结果

- storage-server 两个副本均为 `Running`、`Ready`，无重启。
- `/crater` 为 8 TiB CephFS 挂载，检查时使用率约 69%。
- CephFS CSI DaemonSet 已达到 `desired=31`、`updated=31`、`ready=31`、`available=31`、`unavailable=0`。
- `dell-63`、`dell-78`、`dell-79`、`dell-80` 均已注册 `rook-ceph.cephfs.csi.ceph.com`。
- CephFS volume healthy，16/16 OSD `up` 且 `in`。
- Ceph 仍为 `HEALTH_WARN`，保留一个 BlueStore slow operation 告警并继续恢复 PG；misplaced 已降至约 5.38%，本次变更后未观察到恶化。
