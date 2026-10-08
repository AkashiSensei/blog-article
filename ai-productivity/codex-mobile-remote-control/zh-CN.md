# （已解决）五次重连/Codex手机版一直显示等待桌面端

> [知乎原文](https://zhuanlan.zhihu.com/p/2038938826651989256) · 原文最后编辑于 2026-08-04 13:37。

之前看 Claude Code 出了手机版我就馋了很久，Cursor 虽然可以通过手机网页实现远程连接电脑上的 Cursor，但是需要启用 On-Demand Usage，要额外收费。

我一直有在手机上操控电脑继续工作的需求，之前用 QClaw 能够实现一部分，现在 Codex 能够通过手机控制之后必须立刻马上跟进。

但是在配置的时候遇到问题，手机 ChatGPT 软件一直显示等待桌面端，桌面上则是等待扫码的状态。

## 问题原因

推测手机端一直显示 `Waiting for desktop` 的本质在于桌面端 Codex 没有成功建立或维持 remote-control 所需的 WebSocket / streaming 长连接。

这个问题和每次对话先 `reconnecting` 5 次原因应该是一致的，我也是经过网友提点才意识到的这个问题的。导致 `reconncting` 的原因是 WebSocket / streaming 连接建立失败，5 次后 fallback 到 HTTP 模式继续工作。

导致这个问题的原因有很多。我的问题是之前代理没有开 TUN，为 Codex 配置的 `~/.codex/.env` 有问题，写成了：

```text
HTTP_PROXY=127.0.0.1:7897
HTTPS_PROXY=127.0.0.1:7897
```

这里缺少 URL scheme，也就是 `http://` 这样的协议头，虽然能够正常通过 HTTP 访问，但是建立 WebSocket 的时候会失败。正确格式应该类似：

```text
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
```

## 我的解决方式

我现在打开 TUN 模式后，直接删除或清空 `~/.codex/.env` 然后重启 Codex 就好了，让 TUN 在系统层接管代理。

## 其它方案

另一种方案是显式让 Codex 走代理。我自己没有使用这种方法，不过看大家有成功案例，写入 `~/.codex/.env` 或者直接用这些环境变量启动 remote-control 应该也可以：

```text
HTTP_PROXY=http://127.0.0.1:7897
HTTPS_PROXY=http://127.0.0.1:7897
ALL_PROXY=http://127.0.0.1:7897
NO_PROXY=localhost,127.0.0.1,::1
```

具体端口要根据自己的代理客户端修改。注意这个写法适合 HTTP 或 mixed 代理端口。

## 或者让它自己解决

也可以把这个帖子分析发给 Codex，让 Codex 自己检查当前代理配置，并确保 remote-control 可以成功通过代理建立长连接。

修改后记得完全退出并重启 Codex。
