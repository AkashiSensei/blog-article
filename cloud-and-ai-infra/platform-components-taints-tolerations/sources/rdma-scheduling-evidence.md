# RDMA 调度现场输出摘录

收集日期：2026-10-01。以下逐行摘取已有终端记录，保留字段原文；省略无关字段，分段之间不是连续输出。没有补写修复后的结果。

## 节点状态

来源：[node.txt](</Users/liuyizhou/develop/blob-article/云计算和AI Infra/K8s中RDMA节点的内核升级问题/node.txt>)。取自第一份节点 describe 输出。

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

## 两个 DaemonSet 的 describe 输出

来源：[kcl.txt](</Users/liuyizhou/develop/blob-article/云计算和AI Infra/K8s中RDMA节点的内核升级问题/kcl.txt>)。保留名称、节点选择条件、计数与容忍。

```text
Name:           rdma-shared-dp-ds
Node-Selector:  feature.node.kubernetes.io/pci-15b3.present=true,network.nvidia.com/operator.mofed.wait=false
Desired Number of Nodes Scheduled: 34
Current Number of Nodes Scheduled: 34
Number of Nodes Scheduled with Up-to-date Pods: 34
Number of Nodes Scheduled with Available Pods: 34
Number of Nodes Misscheduled: 0
Pods Status:  34 Running / 0 Waiting / 0 Succeeded / 0 Failed
  Node-Selectors:       feature.node.kubernetes.io/pci-15b3.present=true
                        network.nvidia.com/operator.mofed.wait=false
  Tolerations:          nvidia.com/gpu:NoSchedule op=Exists
Name:           mofed-ubuntu22.04-76944977f-ds
Node-Selector:  feature.node.kubernetes.io/kernel-version.full=5.15.0-170-generic,feature.node.kubernetes.io/pci-15b3.present=true,feature.node.kubernetes.io/system-os_release.ID=ubuntu,feature.node.kubernetes.io/system-os_release.VERSION_ID=22.04
Desired Number of Nodes Scheduled: 34
Current Number of Nodes Scheduled: 34
Number of Nodes Scheduled with Up-to-date Pods: 34
Number of Nodes Scheduled with Available Pods: 34
Number of Nodes Misscheduled: 0
Pods Status:  34 Running / 0 Waiting / 0 Succeeded / 0 Failed
  Node-Selectors:       feature.node.kubernetes.io/kernel-version.full=5.15.0-170-generic
                        feature.node.kubernetes.io/pci-15b3.present=true
                        feature.node.kubernetes.io/system-os_release.ID=ubuntu
                        feature.node.kubernetes.io/system-os_release.VERSION_ID=22.04
  Tolerations:          nvidia.com/gpu:NoSchedule op=Exists
```

## 按目标节点筛选 Pod

下面两行在原记录中连续出现：前一条查询没有返回匹配 Pod，终端随即执行下一条查询。

```text
➜  crater-insights git:(devel) ✗ kubectl --kubeconfig ~/.kube/config.gpu get pods -n nvidia-network-operator -o wide | grep dell-gpu-06
➜  crater-insights git:(devel) ✗ kubectl --kubeconfig ~/.kube/config.gpu get ds -n nvidia-network-operator
```
