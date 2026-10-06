# 平台组件污点与容忍：素材与证据索引

整理日期：2026-10-01。此次检索读取本地博客材料、Crater 运维仓库、crater-insights 和 Crater 文档，并核对 Kubernetes 官方文档、Network Operator v25.1.0 与 Rook v1.17.4 的配置和源码。未查询当前集群；下文的运行状态均为历史记录。

## 1. 已找到的材料

| 材料 | 能提供什么 | 来源 |
| --- | --- | --- |
| RDMA 原始笔记，2026-04-06 | 独占污点、NicClusterPolicy 补丁、持久配置思路 | [本目录快照](sources/rdma-taints-note.2026-04-06.md)；[原文件](</Users/liuyizhou/develop/blob-article/云计算和AI Infra/K8s中RDMA节点的内核升级问题/note.md>) |
| RDMA 节点现场输出 | 内核版本、污点、等待标签、RDMA 资源数量 | [node.txt](</Users/liuyizhou/develop/blob-article/云计算和AI Infra/K8s中RDMA节点的内核升级问题/node.txt>) |
| RDMA DaemonSet 现场输出 | 节点选择条件、容忍、34/34 状态、目标节点没有插件 Pod | [kcl.txt](</Users/liuyizhou/develop/blob-article/云计算和AI Infra/K8s中RDMA节点的内核升级问题/kcl.txt>) |
| RDMA 原始输出的重点摘录 | 汇集节点字段与两个 DaemonSet 的覆盖、容忍信息 | [本目录摘录](sources/rdma-scheduling-evidence.md) |
| RDMA 洞见笔记 | 硬件到 Kubernetes 资源上报的链路，以及 DKMS 的解释 | [rdma.md](/Users/liuyizhou/develop/crater/crater-insights/infrastructure/rdma.md:234) |
| Crater RDMA 管理文档 | 内核升级后的问题分析、补丁与验证步骤 | [rdma.mdx](/Users/liuyizhou/develop/crater/main/crater/website/content/docs/admin/more/rdma.mdx:1006) |
| Network Operator 部署源文件 | 25.1.0 Helm values 与独立的 NicClusterPolicy | [values.yaml](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/network-operator/25.1.0/values.yaml:213)、[NicClusterPolicy.yaml](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/network-operator/25.1.0/ShareDevice/NicClusterPolicy.yaml) |
| storage-server 完整故障复盘，2026-07-20 | 对应“3 月 7 日的旧 Pod 正常，新 Pod 无法启动”的原始记录；包含时间线、根因边界与修复结果 | [本目录快照](sources/storage-server-cephfs-csi-incident.2026-07-20.md)；[原文件](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/crater/crater-helm-charts/archive/2026-07-20-storage-server-cephfs-csi-incident/README.md) |
| Rook 修复后的声明式配置 | CephFS 专用容忍，以及内核客户端配置 | [rook-ceph-operator-config.yaml](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/crater/crater-helm-charts/archive/2026-07-20-storage-server-cephfs-csi-incident/rook-ceph-operator-config.yaml) |
| Crater 存储管理文档 | 已将此事故提炼为 CSI 节点覆盖的排障经验 | [storage.mdx](/Users/liuyizhou/develop/crater/main/crater/website/content/docs/admin/deployment/storage.mdx:138) |

原博客目录中的 `article.md` 只有标题，实际内容在 `note.md`、`node.txt`、`kcl.txt` 中。Crater 文档读取时本地 HEAD 为 `74e7fd6e`；这里引用的是本地文件，不代表已经核验对应内容在远端的发布状态。

## 2. 案例一：RDMA 节点内核升级后，硬件可用但资源没有上报

### 2.1 可直接由现场输出确认的事实

目标节点为 `dell-gpu-06`。`node.txt` 记录了：

```text
feature.node.kubernetes.io/kernel-version.full=5.15.0-170-generic
network.nvidia.com/operator.mofed.wait=true
Taints: crater.raids.io/account=q-48:NoSchedule
Unschedulable: false
rdma/rdma_a100: 0
rdma/rdma_v100: 0
```

`kcl.txt` 中，OFED DaemonSet 的节点选择条件包括同一个内核版本 `5.15.0-170-generic`；RDMA shared device plugin 的节点选择条件包括 `network.nvidia.com/operator.mofed.wait=false`。两个 DaemonSet 模板都只显示了 `nvidia.com/gpu:NoSchedule` 的容忍，没有 `crater.raids.io/account` 的容忍。按节点筛选的输出中没有 Network Operator 管理的 Pod。

因此，已有证据支持两处阻塞：OFED 组件缺少独占污点容忍，无法覆盖目标节点；下游设备插件还被 `mofed.wait=true` 排除。不能仅凭“设备插件 Pod 不存在”认定硬件故障。

特别适合写进博客的细节：两个 DaemonSet 都显示 **34/34 Running**，仍然遗漏了目标节点。DaemonSet 的 desired 是按节点选择和污点等条件计算的覆盖目标；总数一致不等于覆盖了所有业务需要的节点。

### 2.2 故障链路与驱动自愈

历史分析把触发背景记为节点内核升级，链路可以写成：

```text
内核升级，驱动环境需要重新校验
  → OFED 驱动组件需要进入目标节点
  → 节点独占污点没有被容忍，组件无法进入
  → mofed.wait 持续为 true
  → RDMA shared device plugin 的节点条件不满足
  → kubelet 没有获得可用的 RDMA 扩展资源
  → 请求 RDMA 的新作业无法使用该节点
```

crater-insights 与 Crater 管理文档记载，宿主机此时的网卡和驱动正常，并用 DKMS 在新内核下安装驱动解释这种现象；笔记中还给出了 `mlnx-ofed-kernel` 对 `5.15.0-170-generic` 处于 `installed` 的输出。该部分属于已有排障笔记的记录，原博客目录没有完整的宿主机诊断日志，不能包装成此次重新实测的结论。

写作重点是分开三个判断：硬件与内核驱动能否工作、Operator 是否完成自己的就绪流程、kubelet 是否上报资源。宿主机具备 RDMA 能力，并不能代替设备插件向 Kubernetes 注册资源。

### 2.3 配置入口与持久化边界

本地记录使用 Network Operator 25.1.0，Operator 由 Helm 部署，`NicClusterPolicy` 单独管理。NVIDIA 发布说明记载，自 24.10.0 起 Chart 不再创建 NicClusterPolicy CR；25.1.0 API 中的 `spec.tolerations` 用于向它管理的 DaemonSet 注入容忍。[发布说明](https://docs.nvidia.com/networking/display/kubernetes2501/release-notes.html)、[API 参考](https://docs.nvidia.com/networking/display/kubernetes2501/customizations/crds.html)。

经过固定版本源码核对：`operator.tolerations` 渲染到 Operator **Deployment 自身**的 PodSpec；OFED 与 RDMA shared device plugin 的渲染数据读取 `cr.Spec.Tolerations`。两处配置不能互相替代。[Operator 模板](https://github.com/Mellanox/network-operator/blob/v25.1.0/deployment/network-operator/templates/operator.yaml#L46)、[OFED 渲染](https://github.com/Mellanox/network-operator/blob/v25.1.0/pkg/state/state_ofed.go#L571)、[RDMA 插件渲染](https://github.com/Mellanox/network-operator/blob/v25.1.0/pkg/state/state_rdma_shared_device_plugin.go#L152)。

补入独占节点容忍的配置片段为：

```yaml
spec:
  tolerations:
    - key: crater.raids.io/account
      operator: Exists
      effect: NoSchedule
```

这是现有 NicClusterPolicy 的字段片段，不是完整资源文件。应保留已有容忍及其他字段，修改实际管理这个 CR 的源文件，再检查生成的 DaemonSet 和目标节点。

**当前检索发现的差异**：本地 `ShareDevice/NicClusterPolicy.yaml` 仍没有 `spec.tolerations`。旧笔记提出了持久化方案，但这份文件不足以证明方案已经落到 Git。原博客目录也没有补丁后的节点资源输出，暂不能给出 RDMA 恢复的精确时间、恢复数值或端到端测试结果。

## 3. 案例二：CephFS CSI 缺席，storage-server 新 Pod 卡在 ContainerCreating

### 3.1 与回忆对应的现场

2026-07-20 的复盘完整对应此次提供的描述，记录日期和修复日期均已明确。

| Pod | 创建时间（Asia/Shanghai） | 节点 | 调查时状态 |
| --- | --- | --- | --- |
| `webdav-deployment-575b9c7dc9-4wkkr` | 2026-03-07 15:38:20 | `dell-78` | Running、Ready |
| `webdav-deployment-575b9c7dc9-br67s` | 2026-03-07 15:45:58 | `dell-80` | Running、Ready |
| `webdav-deployment-54bf67b97-655mx` | 2026-07-19 20:00:07 | `dell-79` | Pending，容器等待原因为 ContainerCreating |

新 Pod 已调度，容器尚未启动，事件报错是：

```text
MountVolume.MountDevice failed ... driver name
rook-ceph.cephfs.csi.ceph.com not found in the list of registered CSI drivers
```

PVC `crater-storage` 已 Bound，对应 VolumeAttachment 显示 `attached=true`，仍不能完成节点侧挂载。复盘还确认 Crater Helm 修订 67、68 的 storage-server Deployment 声明没有变化；该次 rollout 暴露的是之前已经存在的节点条件问题。

### 3.2 故障链路

三个控制平面节点都有 `node-role.kubernetes.io/control-plane:NoSchedule`。storage-server 有节点选择和相应容忍，可以运行在那里；Rook 管理的 `csi-cephfsplugin` DaemonSet 缺少该容忍，三个节点都没有 CephFS CSI Pod，`CSINode` 也没有列出 CephFS 驱动。

```text
业务 Pod 可以进入控制平面节点
  → CSI 节点插件没有对应容忍，无法覆盖这些节点
  → kubelet 找不到注册的 CephFS CSI 驱动
  → 无法为新 Pod 建立 /crater 挂载
  → 容器尚未启动，等待原因为 ContainerCreating
```

CSI 注册是节点级的：node-driver-registrar 在所在节点向 kubelet 注册驱动，kubelet 再向驱动发起节点侧调用。其他节点有插件不能替代目标节点的注册。[CSI node-driver-registrar 文档](https://kubernetes-csi.github.io/docs/node-driver-registrar.html)。

### 3.3 为什么 3 月 7 日的旧 Pod 还能工作

该集群使用 `CSI_FORCE_CEPHFS_KERNEL_CLIENT=true`。旧 Pod 已经建立了 CephFS 内核挂载，后续文件读写通过内核客户端访问 Ceph，不需要 CSI Pod 代理每次读写。因此，节点插件缺席时，既有挂载可能继续有效，新建挂载却失败。

这解释了“旧服务正常”的假象：它证明的是过去的挂载成功，无法证明现在有能力完成 Pod 重建、迁移或节点重启后的挂载恢复。此解释以该事故的内核客户端配置为前提，不应推广成所有存储驱动在插件消失后都能正常工作。

### 3.4 配置变化时间线与证据边界

| 时间（Asia/Shanghai） | 记录中的变化 |
| --- | --- |
| 2026-02-28 18:41:58 | 原 DaemonSet 模板没有控制平面容忍 |
| 2026-03-07 15:37:07 | 修订 15，仅增加 rollout restart 注解 |
| 2026-03-07 15:45:45 | 修订 16，增加控制平面 NoSchedule 容忍 |
| 2026-03-09 22:11:41 | 修订 17，重新使用没有容忍的原模板 |
| 2026-07-19 20:00:07 | 新 storage-server Pod 创建，挂载失败 |
| 2026-07-20 19:35 | 应用修复后的 Rook ConfigMap |
| 2026-07-20 20:11 | CephFS CSI DaemonSet 完成 31/31 更新 |

“容忍曾经加入，又被移除”有 ControllerRevision 历史支持；“一定是 Operator 在 3 月 9 日覆盖了某人的手动补丁”仍是推断。旧 ConfigMap 没有声明该容忍，Operator reconcile 能解释回退，但显式 rollback 或 apply 旧模板也能产生相同结果。复盘明确说明缺少可用的审计证据，不能确定操作者和命令。

第二只旧 Pod 创建于增加容忍之后；第一只创建得更早，现存证据无法还原它具体通过哪一次临时插件或操作完成挂载。另一个时间线细节是修订 17 复用了旧 ControllerRevision，不能把该对象创建时间当成此次回退时间。

### 3.5 已实施的持久修复和结果

历史调查确认，实际运行的是 **Rook v1.17.4，使用原始 YAML 管理，没有 Rook Helm release**。运维仓库另一个 `rook-ceph/` 目录记录的 Helm 与外部 Ceph 部署方式不符合这个现场。修复落到 `rook-ceph/rook-ceph-operator-config` ConfigMap，新增：

```yaml
data:
  CSI_CEPHFS_PLUGIN_TOLERATIONS: |
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
```

这里只展示新增字段。Rook v1.17.4 源码确认存在 CephFS 专用配置，它优先于通用插件容忍；通用配置作为未设置专用配置时的默认值。该次选择专用字段，避免同时改变 RBD 插件。[Rook v1.17.4 CSI 配置实现](https://github.com/rook/rook/blob/v1.17.4/pkg/operator/ceph/csi/spec.go#L504)。

复盘记录的修复结果：三个控制平面节点都有 CSI Pod 并注册 CephFS 驱动；等待中的 storage-server 自动完成挂载和启动；Deployment 完成 rollout，两个副本 Running、Ready；`/crater` 确认为 CephFS；CSI DaemonSet 最终为 31/31。无需手动删除业务 Pod。

更新过程中另有一个独立故障：`dell-63` 出现 containerd `content digest ... not found`，使 `maxUnavailable=1` 的滚动更新停在 `updated=11`、`ready=30`。维护者定向执行 `crictl pull` 后，kubelet 下一次重试成功，rollout 继续。记录不能严格区分 pull 修复内容存储还是与自动重试的时机重合；不宜把它写成污点问题或确定的因果验证。

## 4. 两个案例可以提炼的文章主线

| 对照项 | RDMA 案例 | CephFS CSI 案例 |
| --- | --- | --- |
| 业务准入规则 | 账户独占污点 | 控制平面污点 |
| 被遗漏的节点组件 | OFED 驱动组件、RDMA shared device plugin | CephFS CSI node plugin |
| 外部表现 | RDMA 资源为 0，新 RDMA 作业无法使用节点 | 业务 Pod 已调度，新挂载失败 |
| 容易误导的健康信号 | 宿主机驱动正常、DaemonSet 34/34 | PVC Bound、attached=true、旧 Pod Running |
| 修复配置入口 | NicClusterPolicy 的 spec.tolerations | 实际 Rook 安装方式支持的 ConfigMap / Helm 配置 |
| 当前材料的验证程度 | 原始故障证据和修复方案明确，缺少恢复现场输出 | 完整修复时间线和最终验证结果 |

文章可以围绕这个判断展开：**允许业务使用一个节点时，还必须确认该节点有能力提供业务依赖的网络、设备与存储服务。** 对某一依赖而言，业务可能使用的节点应被该依赖的节点组件覆盖。组件覆盖不仅取决于 toleration，也取决于硬件标签、节点亲和性、版本匹配与就绪状态。

## 5. 原始材料中需要修正的表述

1. **CR 与 CRD**：`nic-cluster-policy` 是 NicClusterPolicy 的一个自定义资源实例（CR）；CRD 定义这种资源的 schema。应修改实例或实例的声明式源文件，不能笼统写成“patch CRD”。
2. **Operator 自身与受管组件**：原笔记“operator.tolerations 只影响 Operator 本身部署的 Daemonsets”不够准确。25.1.0 的模板把它用于 Operator 自身的 Deployment；受管组件从 CR 的 spec.tolerations 取值。官方 Helm 参数表的说明容易混淆，应以该版本模板和渲染代码说明配置去向。
3. **cordon**：标准 DaemonSet Pod 会自动获得 `node.kubernetes.io/unschedulable:NoSchedule` 容忍，不能仅因节点被 cordon 就认定该 DaemonSet 被挡住。本案原始节点输出的 `Unschedulable` 还是 false；明确的阻塞是自定义账户污点。应区分 DaemonSet 模板和实际 Pod 中控制器添加的容忍。[Kubernetes DaemonSet 文档](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#taints-and-tolerations)。
4. **patch 持久性**：API patch 可以保存在集群对象中，但不等于已经写入 Git 或上游声明式配置。原笔记的 JSON merge patch 会替换整个 tolerations 数组，写成操作步骤时应保留原有条目。
5. **NoSchedule 与旧 Pod**：NoSchedule 阻止未匹配容忍的新调度，不会单凭这个 effect 驱逐已运行 Pod；NoExecute 则有不同语义。它们不能混写。[Kubernetes Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)。
6. **恢复证据与历史推断**：Rook 的配置回退触发者、第一只旧 Pod 的具体挂载过程和 RDMA 修复后的资源数值仍有缺口，正式博客应保留这些边界。
7. **更新策略**：成文时补充核对的 25.1.0 OFED 模板使用 `OnDelete`。不能把“Operator 更新模板”与“自动滚动替换旧驱动 Pod”混为一谈；新合格节点补建 Pod、旧 Pod 的替换、组件升级控制逻辑需要分别判断。[固定版本模板](https://github.com/Mellanox/network-operator/blob/v25.1.0/manifests/state-ofed-driver/0050_ofed-driver-ds.yaml#L23)。
8. **等待标签的更新时机**：25.1.0 控制器根据已分配到节点的 OFED 容器 Ready 状态维护等待标签；现有现场只证明故障时标签为 true，没有完整还原内核变更到标签变化的过程。不要写成“内核标签一变化，Operator 就必然立即置 true”。[控制器实现](https://github.com/Mellanox/network-operator/blob/v25.1.0/controllers/nicclusterpolicy_controller.go#L185)。

## 6. 可复用的只读排查顺序

先看业务 Pod 事件，区分调度阶段和节点启动阶段，再沿依赖向下查。以下命令为后续写作参考，本次没有对集群执行：

```bash
# 共用：目标节点的污点、标签、Pod 实际节点及事件
kubectl describe node <node>
kubectl -n <namespace> get pod <pod> -o wide
kubectl -n <namespace> describe pod <pod>

# RDMA：模板选择条件、目标节点覆盖、等待标签和上报资源
kubectl -n nvidia-network-operator get ds -o yaml
kubectl -n nvidia-network-operator get pods -o wide --field-selector spec.nodeName=<node>
kubectl get node <node> -L network.nvidia.com/operator.mofed.wait
kubectl get node <node> -o json
kubectl get nicclusterpolicy nic-cluster-policy -o yaml
kubectl explain nicclusterpolicy.spec.tolerations

# CSI：目标节点插件、节点驱动注册、卷声明和真实报错
kubectl -n rook-ceph get ds csi-cephfsplugin -o yaml
kubectl -n rook-ceph get pods -o wide --field-selector spec.nodeName=<node>
kubectl get csinode <node> -o yaml
kubectl -n crater-workspace get pvc crater-storage -o yaml
kubectl -n rook-ceph get configmap rook-ceph-operator-config -o yaml
```

RDMA 验证应走到目标节点上插件 Ready、等待标签释放、资源 Capacity / Allocatable 恢复，以及实际作业使用 RDMA。CSI 验证应走到目标节点驱动注册、DaemonSet rollout 完成、业务完成挂载和实际读写。声明式配置改动则应从实际管理入口追踪到最终生成的 PodSpec。

## 7. 后续写作仍需补充的材料

- RDMA patch 后目标节点的插件 Pod、标签变化、资源数量与作业验证输出。目前材料足以讲故障和修复入口，不足以讲完整恢复时间线。
- RDMA 当时的实际安装版本和生成对象版本关联，以及 source YAML 是否后来在另一分支或配置仓库持久化。目录名 25.1.0 只是本地部署材料的版本线索。
- Rook 3 月 9 日回退的审计信息。若没有额外留存，应保留“符合 Operator reconcile 的解释，但具体触发不明”的措辞。
- 正式成文时，可把内部节点名、Pod 名和仓库本地路径换成匿名示例，来源索引作为作者资料保留。

## 8. 扩展检索结果

其他故障记录、调度相关问题，以及仅包含污点容忍配置的材料，已整理到 [补充故障索引](additional-incidents.zh-CN.md)。当前明确由容忍配置引起的独立事件仍为上述两起；没有找到第三起证据充分的污点容忍故障。
