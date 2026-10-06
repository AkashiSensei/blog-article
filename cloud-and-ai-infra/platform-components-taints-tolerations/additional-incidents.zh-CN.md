# 补充检索：其他故障记录与污点容忍线索

检索日期：2026-10-01。范围包括本地 Crater 的 operations、crater-insights、storage、backup、main 下的工作树、zhejianglab 副本，以及博客仓库中的云计算与 AI Infra 材料。重复副本和翻译不作为独立事件计数。

**结果：目前明确记录了故障与污点容忍因果关系的，仍是 RDMA 独占节点和 Rook CephFS CSI 两起。没有找到第三起证据充分的独立污点容忍故障。** 以下新增记录可以用作其他博客选题，或帮助区分相似表象。

## 1. 其他实际故障或部署问题记录

| 记录 | 文档记载的现象或结论 | 材料完整程度 | 是否支持归因为污点容忍 |
| --- | --- | --- | --- |
| [2026-07-04：inspur-gpu-13 宿主机 OOM 与 NVIDIA UVM 未记账内存](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/cluster/incidents/inspur-gpu-13-oom-incident-2026-07-04.md:16) | 约 893 GiB 宿主机内存未进入 Pod cgroup 记账，触发全局 OOM；LXCFS 被杀后残留断开的 FUSE 挂载，新 Pod 持续 FailedMount。报告高度怀疑旧 vLLM 工作负载为主要触发者，但具体 CUDA/vLLM 调用点未确认。 | 完整复盘，有时间线、指标、源码分析、恢复过程和置信度分层 | 否。taint/unschedulable 仅在检查清单中出现；根因分析指向 UVM 记账与 OOM。 |
| [RDMA 卡死／驱动阻塞的 shell 记录](</Users/liuyizhou/develop/blob-article/云计算和AI Infra/RDMA卡死-驱动死锁问题解决/shell.txt:118>) | dell-gpu-32 上多只 ib_write_bw 进程为 D 状态，等待点为 mlx5r_umr_post_send_wait；2026-04-11 的内核日志反复报告任务阻塞超过 120 秒，栈涉及 mlx5_ib_reg_user_mr。后面记录了 reboot 命令。 | 原始终端记录；未找到完整根因说明或重启后验证 | 否。现有材料是已经运行的测试进程在内核驱动调用中阻塞，没有未容忍污点的证据。目录名含“问题解决”，但不能据此补写已验证的解决结论。 |
| [RDMA 大数据量测试 MR 分配失败](/Users/liuyizhou/develop/blob-article/cloud-and-ai-infra/rdma-memlock-limit/zh-CN.md:174) | 小消息测试正常，1 MiB 及以上测试失败，报 Couldn't allocate MR；文档记录容器 memlock 为 64 KiB，采用 containerd OCI 基础规范配置解除限制，并附测试输出。 | 已有完整博客、失败截图、配置与测试 | 否。文档分析指向容器内存锁定限制。 |
| [ACK 迁移：control-plane 节点选择导致无法调度](</Users/liuyizhou/develop/crater/operations/act-gpu-cluster/crater/测试集群.md:94>) | 笔记记载 ACK 托管集群不暴露 master，需要移除原 control-plane 节点选择条件，组件改用 devops 节点标签。 | 简短问题记录，没有 FailedScheduling 原始事件 | 未支持。记录明确指出的是 nodeSelector；不能因为同样涉及控制平面节点就归入 toleration 故障。 |
| [CloudNativePG / OpenEBS：WaitForFirstConsumer 与反亲和性](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/cloudnative-pg/README.md:82) | 笔记提醒 PVC 的 WaitForFirstConsumer 模式可能出现死锁，增加节点亲和性标签；另提醒 init Pod 有反亲和性要求，需要多个运维节点。 | 部署经验与风险提示，缺少事件和完整依赖还原 | 未支持。现有描述指向卷绑定、亲和性与可用节点数量。 |
| [GPU Operator / NFD：卸载 hook 无法完成](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/gpu-operator/README.md:111) | ACK 部署记录指出 prune post-delete Job 没有正确加入 imagePullSecrets，导致 Helm 删除时 hook 无法完成，随后给出模板修改。 | 明确问题与修改片段，缺少修复后输出 | 否。模板虽然也带 tolerations，但该条记录指向镜像拉取凭据。 |
| [ACK 迁移：自定义 scheduler 无法进入调度循环](</Users/liuyizhou/develop/crater/operations/act-gpu-cluster/crater/测试集群.md:130>) | 笔记记载升级 Kubernetes 1.27 后，storagecapacity 相关 API 变化导致 informer cache 同步失败，调度循环不能启动；记录了源码调整。 | 简短历史记录，缺少对应提交和日志 | 未支持。笔记指向 API 兼容性，不能把“无法调度”直接等同于污点问题。 |
| [CSI rollout 中的 containerd 镜像内容缺失](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/crater/crater-helm-charts/archive/2026-07-20-storage-server-cephfs-csi-incident/README.md:252) | 修复 CSI 容忍后，dell-63 报 content digest not found，使 maxUnavailable=1 的 rollout 暂停。定向 crictl pull 后下一次创建重试成功；记录保留因果不确定性。 | Rook 复盘中的独立次生问题，已有恢复记录 | 否。应与同一次操作中的容忍修复分开分析。 |

以上是本地文档记载的结论，本次没有复现这些故障或核对当前集群状态。

## 2. 有污点容忍配置，但不是独立故障复盘的材料

| 文档 | 找到的内容 | 可用于博客的角度 |
| --- | --- | --- |
| [OpenEBS README](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/openebs/README.md:30) | localpv-provisioner 与测试 Pod 都配置 control-plane nodeSelector 和 NoSchedule 容忍 | 可以用作“业务与存储相关组件需要分别配置准入规则”的配置实例；没有缺容忍导致失败的现场。 |
| [Prometheus Stack README](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/prometheus-stack/README.md:14) | Grafana、kube-state-metrics 子 Chart 的节点选择与 control-plane 容忍示例 | 可以讲配置所在子 Chart 的层级；记录中的 Grafana Service 不好使是另一个问题，没有污点归因。 |
| [Ingress NGINX README](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/ingress-nginx/README.md:23) | 指定节点、清空 controller.tolerations，以及被注释的控制平面容忍参数 | 可作为待核对的部署配置线索；目标节点当时是否带污点、是否真的调度失败均未记录。 |
| [BuildKit README](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/buildkit/README.md:119) | rootless StatefulSet 选择控制平面节点，并显式配置容忍 | 可作为平台服务部署在控制平面节点上的实例；没有独立容忍故障记录。 |
| [Flannel 配置](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/cluster/README.md:280) | DaemonSet 使用 operator: Exists、effect: NoSchedule 的容忍，未指定 key | 可对比基础网络组件与前两个故障中受限的平台组件；这份配置没有给出新增事故。 |
| [CNPG 同节点告警 runbook](/Users/liuyizhou/develop/crater/operations/act-gpu-cluster/cloudnative-pg/cluster/docs/runbooks/CNPGClusterInstancesOnSameNode.md:26) | 通用排查步骤提醒检查可用节点污点和 affinity | 属于 Chart 携带的通用手册，不证明本地集群发生过这起污点事故。 |

## 3. 对当前博客选题的意义

RDMA 与 Rook 两起已有案例足以支撑“业务准入与节点基础设施覆盖不一致”的主线。其他材料适合充当配置对照，不能写成未经证实的第三起亲历事故。

如果扩展成平台组件可用性的系列文章，OOM 后 LXCFS 挂载失败是较强的下一篇候选：它同样表现为业务无法使用节点上的基础设施，但节点组件缺席的原因是 OOM。RDMA MR 限制、驱动阻塞、ACK 调度器兼容性则可以分别展示相似现象在不同层面的根因。
