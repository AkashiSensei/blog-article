# 云计算和 AI Infra

更准确地说可能是面向 AI 的云原生平台工程和系统实践，而非全栈的 AI Infra，对特征工程、训练框架等实现细节涉及较少，侧重各厂商 GPU、NPU 资源管理、容器化开发环境、工作流/任务编排等平台能力，致力于为 AI 工作者提供简单易用的计算资源。

这里的文章可能大多都是依托于 Crater 的，它是一个我们项目组维护的一个面向 Kubernetes 的AI 开发平台。

## 文章

- [K8s-containerd 环境下为 RDMA 修改 Pod 内存锁定上限](rdma-memlock-limit/zh-CN.md) / [English](rdma-memlock-limit/en-US.md)
- [从 RDMA 资源消失到 CSI 挂载失败：平台组件的污点与容忍](platform-components-taints-tolerations/zh-CN.md) / [English](platform-components-taints-tolerations/en-US.md)
