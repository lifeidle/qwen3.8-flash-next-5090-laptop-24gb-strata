# 2026-10-28 · 服务端 SSE 心跳：双并发下客户端 60 秒超时的根治

> 结论先行：双并发时一个窗口反复报
> **"SSE stream error: Timeout after 60000ms (origin: downstream) (4028)"** ——
> 这不是模型出错，是**客户端在流式连接上连续 60 秒没收到任何字节后主动断开**。
> 修复：server.py 的流式输出加**服务端心跳**——源静默 10 秒即发 `: keep-alive` 注释行。
> 实测最大静默间隔从 60 秒+ 压到 **1.4 秒**，超时从机制上不可能再触发。

---

## 一、问题：双并发下的三个静默窗口

单路时解码流永不中断，这个错误从不出现。双并发时出现三个"零字节窗口"：

| 静默窗口 | 机制 | 时长 |
|---|---|---|
| **① 另一路的长文档读入** | 两路共享 prefill 带宽（~2000 tok/s 合计），另一路灌 13 万 token 大文档时独占读入 50-90 秒，本路解码窗口停摆 | 50-90 s |
| **② 排队等槽位** | batch2 只有两个槽，都被长任务占着时，新请求 FIFO 排队，期间一个字节都不发 | 分钟级 |
| **③ 超长深度思考** | IQ3_S 的 xhigh 思考可达 6000+ token；若客户端不把思考增量算作活动，就是长静默 | 60-120 s |

**为什么会这样**：解码是内存带宽受限的——另一路的 prefill 要占满读入带宽，本路的解码
窗口在这期间停摆。这是带宽受限推理的物理特性，不是 bug。单路永远不触发，因为自己
的解码流从不中断。

**为什么客户端会断**：多数客户端（Cherry Studio、OneAPI 网关、各类 Agent 框架）对
SSE 流有**无字节超时**（60 秒是常见默认值）。静默超过它 → 主动断开并报错。

## 二、修复：服务端心跳（两层递进）

server.py 的流式输出（OpenAI `/v1/chat/completions` 与 Anthropic `/v1/messages` 两条路径）
包一层心跳：一个 feeder 线程把引擎事件搬进队列，消费端以 10 秒超时等待。

**第一版（SSE 注释行）**：源静默满 10 秒 → HTTP 层写 `": keep-alive\n\n"`。SSE 协议里
注释行会被所有标准解析器忽略——本意是"不是数据但算收到字节"。

**实测发现注释行不够**：部署后用户的客户端仍然超时——**有些客户端的超时计时器只统计
data 事件，SSE 注释行不重置它**。第二版升级为**数据帧心跳**：

- OpenAI 路径：`data: {"choices":[{"index":0,"delta":{},"finish_reason":null}]}`——
  空 delta 的标准块，无内容、不破坏任何解析
- Anthropic 路径：`event: ping\ndata: {"type": "ping"}`——Anthropic 官方流自带的
  保活事件，客户端按协议跳过

两种都是**真实 data 事件**——任何计时器实现（按字节、按行、按事件）都会重置。

```python
def heartbeated(chunks, interval=10.0):
    """源静默 interval 秒即产出 None（HTTP 层转 ": keep-alive" 注释）。"""
    q = queue.Queue()
    DONE = object()
    def feed():
        try:
            for c in chunks:
                q.put(c)
        except BaseException as e:
            q.put(e)
        finally:
            q.put(DONE)
    threading.Thread(target=feed, daemon=True).start()
    try:
        while True:
            try:
                c = q.get(timeout=interval)     # 源静默 interval 秒 → None → 数据帧心跳
            except queue.Empty:
                yield None
                continue
            if c is DONE:
                break
            if isinstance(c, BaseException):
                raise c
            yield c
    finally:
        chunks.close()
```

覆盖的静默窗口：**排队等待**（FIFO 阻塞在生成器开头，心跳照发）、**另一路的长 prefill**
（同上）、**工具执行的长间隔**（MCP 轮次之间）。异常也经队列透传（EngineDied 等
错误处理路径不变）。

## 三、实测验证

一路灌 13 万 token 长文档（制造最恶劣的读入静默），另一路同时开流式窗口，
用原始 SSE 读取器逐行记录：

| 指标 | 结果 |
|---|---|
| keep-alive 注释到达 | **2 次**（在另一路 prefill 期间） |
| 本路流最大静默间隔 | **1.4 秒** |
| 60 秒客户端超时 | **机制上不可能再触发**（最大静默 << 60） |

客户端侧双保险：把流式超时从 60 s 调到 **300 s**（Cherry Studio：设置 → 模型提供商 →
超时；OneAPI：渠道超时配置）。

## 四、给部署者的提醒

- 这个修复在 server.py（`heartbeated()` 包装器 + 两处接入），升级引擎包**不影响**它
  （引擎与 server 是两层）
- 客户端把超时调大仍是好习惯：排队分钟级的场景（三路以上）心跳也救不了"槽位不足"，
  只能救"连接被误断"
