> 历史材料快照，收集于 2026-10-01。来源：[note.md](</Users/liuyizhou/develop/blob-article/云计算和AI Infra/K8s中RDMA节点的内核升级问题/note.md>)。
> 正文保留原记录，其中 CR/CRD、Operator 容忍配置等表述的校正见 [素材索引](../materials.zh-CN.md)。

# RDMA 污点容忍问题 - 技术笔记

_创建时间: 2026-04-06_

---

## 问题描述

Crater 集群节点有独占污点 `crater.raids.io/account`，导致 NVIDIA Network Operator 部署的 RDMA 相关 Pod 无法调度：

- `mofed-driver` Pod 无法调度
- `network.nvidia.com/operator.mofed.wait` 标签卡在 `true`
- RDMA 资源 `rdma/rdma_v100` 无法上报

---

## 临时解决方案

使用 `kubectl patch` 修改 NicClusterPolicy：

```bash
kubectl patch nicclusterpolicy nic-cluster-policy --type='merge' -p '{
  "spec": {
    "tolerations": [
      {
        "key": "crater.raids.io/account",
        "operator": "Exists",
        "effect": "NoSchedule"
      },
      {
        "key": "node.kubernetes.io/unschedulable",
        "operator": "Exists",
        "effect": "NoSchedule"
      }
    ]
  }
}'
```

---

## 持久化方案分析

### 当前部署结构

| 组件 | 部署方式 | 配置文件 |
|------|---------|---------|
| Network Operator 本身 | Helm Chart | `values.yaml` |
| NicClusterPolicy (CRD) | 独立 YAML | `NicClusterPolicy.yaml` |

**关键发现：** NicClusterPolicy 是独立的 Kubernetes CRD，不由 Helm values.yaml 模板渲染。

### 尝试过的方案

❌ 修改 `network-operator/values.yaml` 中的 `operator.tolerations`
- 只影响 Operator 本身部署的 Daemonsets
- 不会传递到 NicClusterPolicy 创建的 Pod（ofedDriver、rdmaSharedDevicePlugin）

### 正确的持久化方案

修改 `act-gpu-cluster/network-operator/25.1.0/ShareDevice/NicClusterPolicy.yaml`：

```yaml
apiVersion: mellanox.com/v1alpha1
kind: NicClusterPolicy
metadata:
  name: nic-cluster-policy
spec:
  tolerations:
    - key: "crater.raids.io/account"
      operator: "Exists"
      effect: "NoSchedule"
  ofedDriver:
    ...
  rdmaSharedDevicePlugin:
    ...
```

然后 `kubectl apply -f NicClusterPolicy.yaml`

---

## 结论

**无法通过修改 Helm values.yaml 一处配置来持久化添加污点容忍。**

原因：
1. NicClusterPolicy 是独立管理的 CRD
2. 不在 Helm 模板渲染流程中
3. 需要单独修改 NicClusterPolicy.yaml 并版本化管理

**但这仍然是"持久化"的：**
- NicClusterPolicy.yaml 在 Git 仓库中版本化管理
- 可以在部署脚本中自动 apply
- 不会因为集群重建而丢失

---

## 相关文件

- Crater 文档: `raids-lab/crater/website/content/docs/admin/more/rdma.mdx`
- 集群配置: `~/develop/crater/operations/act-gpu-cluster/network-operator/`
- 洞见笔记: `~/develop/crater/crater-insights/infrastructure/rdma.md`
