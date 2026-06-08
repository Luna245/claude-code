# 04 - Prompt Caching 完全指南

> Prompt Caching 是 Anthropic 提供的**前缀缓存**机制。本章覆盖：开关、配置、客户端/服务端行为、失效条件、成本模型。

---

## 一、Prompt Cache 是什么

**一句话**：把请求中**稳定不变的前缀**对应的**计算结果**（KV 状态）缓存起来，下次同样前缀直接复用，跳过 prefill 计算。

### 它**不是**什么

- ❌ 不是"Claude 记住了对话"——它不影响模型行为
- ❌ 不是省 token 数量——前缀该传还得传
- ❌ 不是省网络流量——HTTP body 完整发送

### 它**是**什么

- ✅ 省**计算时间**（prefill 跳过）
- ✅ 省**钱**（命中部分按 10% 价格计费）
- ✅ 省**延迟**（首 token 响应时间显著下降）

---

## 二、Claude Code 里的开关

### 默认状态：自动开启

Claude Code 装好就在用了，无需配置。

### 怎么关掉

环境变量（粒度由粗到细）：

```bash
# 全局关闭（优先级最高）
export DISABLE_PROMPT_CACHING=1

# 按模型分级关闭
export DISABLE_PROMPT_CACHING_OPUS=1
export DISABLE_PROMPT_CACHING_SONNET=1
export DISABLE_PROMPT_CACHING_HAIKU=1
```

或写进 `~/.claude/settings.json`：
```json
{
  "env": {
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

关闭后 Claude Code 启动时会显示警告横幅。

### TTL（缓存存活时间）

Claude Code 用户**不能选**——服务端根据你的认证方式自动决定：

| 用户类型 | TTL |
|---------|-----|
| Max 订阅 | 1 小时 |
| Pro 订阅 | 5 分钟 |
| API key | 5 分钟 |
| 超出套餐走 credits | 自动降到 5 分钟 |

Max 用户的 1 小时 TTL **不另外收费**（因为按套餐算）。

### 什么时候你**会想关掉**

- 用 AWS Bedrock / Vertex AI / 第三方网关，遇到 `cache_control.scope` 兼容性错误
- 做成本测试，想看真实未缓存的开销
- 调试缓存命中率问题
- 用 `--resume` 恢复会话报 400 错误

---

## 三、API 调用时怎么开（开发者层面）

API 默认**不开**，必须手动加 `cache_control` 标记。

### 模式 1：自动缓存（推荐）

在请求顶层加一个 `cache_control`，系统自动把断点应用到最后一个可缓存 block：

```python
response = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    cache_control={"type": "ephemeral"},   # 顶层一个就够
    system="You are a code reviewer...",
    messages=[...]
)
```

第一轮之后，几乎 100% 的输入 token 都从缓存读取，**对话代码就是普通的消息列表**，不需要在单个 block 上加特殊标记。

### 模式 2：显式断点（精细控制）

在静态部分的最后一个 block 上打 `cache_control`：

```python
client.messages.create(
    model="claude-opus-4-8",
    system=[
        {"type": "text", "text": "You are an AI assistant..."},
        {
            "type": "text",
            "text": "<整本 Pride and Prejudice 的文本>",
            "cache_control": {"type": "ephemeral"}   # ← 断点
        }
    ],
    messages=[
        {"role": "user", "content": "Analyze the major themes."}
    ]
)
```

### 最多 4 个断点

适合分层缓存：

```
[ system prompt ]                ← 断点 1（很少变）
[ 工具定义 ]                     ← 断点 2（偶尔变）  
[ RAG 文档 / 大文件内容 ]         ← 断点 3（按文档变）
[ 历史对话稳定部分 ]              ← 断点 4（每轮变前的位置）
[ 当前用户消息 ]                 ← 不缓存
```

### 显式指定 TTL

```python
"cache_control": {"type": "ephemeral", "ttl": "5m"}   # 默认
"cache_control": {"type": "ephemeral", "ttl": "1h"}   # 长缓存，写入贵些
```

**经验值**：5 分钟 TTL 在第 2 次请求后回本，1 小时 TTL 在第 3 次回本。只在调用间隔超过 10 分钟时用 1 小时。

---

## 四、能缓存的"目标"和顺序

缓存按这个顺序累积：

```
缓存范围按顺序匹配：

┌─ tools（工具定义）─────────────┐
│                                │
├─ system（系统提示）────────────┤   断点放在哪里
│                                │   决定缓存到哪里
├─ messages（历史对话）──────────┤
│   ...                          │
│   ← cache_control 在这里        │
└────────────────────────────────┘
   ↓
   断点之后的内容动态、不缓存
```

**关键规则**：
- 缓存以**前缀字节**为单位，必须**逐字节相同**才命中
- 最小可缓存长度阈值：大多数模型是 1024 token，Haiku 4.5 是 4096 token——低于阈值即使打了 `cache_control` 也不会真缓存
- 匹配顺序：tools → system → messages，遇到不同就停

---

## 五、什么会让缓存**失效**

| 操作 | 后果 |
|------|------|
| system prompt 改一个字 | 整个缓存失效 |
| 工具定义增减 / 改描述 | 整个缓存失效 |
| 加/减图片（任意位置） | 整个缓存失效 |
| 改 `tool_choice` | 整个缓存失效 |
| 改 `thinking` 开关或 budget | 缓存失效 |
| TTL 到期（5min/1h 没新请求） | 自动过期 |
| 历史里前面的消息被编辑/删除 | 从那一点之后全失效 |
| MCP server 重启（工具定义可能变） | 缓存失效 |

**隐藏的失效源**：
- 在 system prompt 里塞当前时间戳
- 用户名/会话 ID 等动态变量放在 system 里
- 频繁切换 extended thinking 开关

---

## 六、客户端 vs 服务端的行为变化

### 客户端（Claude Code / SDK / 你的代码）

```
请求构造时：
  ├─ 在 prompt 的"稳定前缀"末尾加 cache_control 标记
  ├─ 选择 TTL（5m / 1h）
  └─ 其他内容照常发送
  
请求发送：
  ├─ 完整 prompt 还是要全部发出去（不省网络流量）
  └─ HTTP body 大小不变

响应收到：
  └─ 多了缓存指标字段：
     {
       "cache_creation_input_tokens": N,   ← 这次写入缓存的 token
       "cache_read_input_tokens": M,       ← 这次命中缓存读取的 token
       "input_tokens": K,                  ← 真正新计算的 token
       "output_tokens": L
     }
```

### 服务端

```
收到请求：
  ├─ 1. 用 cache_control 之前的内容计算缓存键
  │     （cryptographic hash of prompts up to the cache control point）
  ├─ 2. 查缓存
  │     ├─ 命中 → 直接复用之前的 KV 状态，跳过 prefill
  │     └─ 未命中 → 重新 prefill 整个前缀，并写入缓存
  ├─ 3. 计算断点之后的动态部分
  ├─ 4. 生成回复
  └─ 5. 返回结果 + 缓存指标
```

---

## 七、三种计费 token

| Token 类型 | 价格（相对基础输入价） | 何时产生 |
|-----------|---------------------|---------|
| `input_tokens`（普通输入） | 100% | 没缓存的部分 |
| `cache_creation_input_tokens` | **125%**（5m）/ **200%**（1h） | 第一次写入缓存 |
| `cache_read_input_tokens` | **10%** | 命中缓存的部分 |
| `output_tokens` | 100% | 模型输出（不受缓存影响） |

### 经济模型

```
不缓存的成本（每次请求）：
  N × 100% = 100N

启用缓存的成本（10 次请求，前缀稳定）：
  第 1 次：N × 125%（写入） = 125N
  第 2-10 次：N × 10% × 9 = 90N
  总计：215N

对比：不缓存 10 次 = 1000N
缓存节省：约 78%
```

**结论**：只要前缀复用 ≥ 2 次，就回本。

---

## 八、怎么观察缓存效果

### 看 API 响应里的 usage 字段

**第一次请求**：
```json
{
  "usage": {
    "cache_creation_input_tokens": 188086,
    "cache_read_input_tokens": 0,
    "input_tokens": 21,
    "output_tokens": 393
  }
}
```

**第二次同样前缀**：
```json
{
  "usage": {
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 188086,   ← 命中了
    "input_tokens": 21,
    "output_tokens": 393
  }
}
```

### Claude Code 里

直接看 `/cost` 或 `/status`，能看到 cache hit / miss 的统计。

---

## 九、Prompt Cache 工作原理示意

```
第一次请求（缓存写入）：

客户端发送                服务端处理              缓存存储
─────────                ─────────              ─────────
[A][B][C]_[D][E]    →   全量 prefill           Key = hash(A,B,C)
带 cache_control                                Value = KV(A,B,C)
在 C 上                                         TTL = 5min

──────────────────────────────────────────────────────

第二次请求（缓存命中）：

客户端发送                服务端处理              缓存命中
─────────                ─────────              ─────────
[A][B][C]_[D][F]    →   计算 Key = hash(A,B,C)  命中！
（C 之前一字不差）           查缓存
                          复用 KV(A,B,C)
                          只 prefill [D][F]
                          → 总 prefill 量大减
                          → 延迟显著下降
                          → 计费 [D][F] 全价 + [A][B][C] 的 10%
```

---

## 十、实战检查清单

### 想最大化省钱

- ✅ 把最静态的内容放最前面（system → tools → 长文档 → 历史 → 当前消息）
- ✅ 用自动缓存模式（顶层一个 `cache_control` 就行）
- ✅ 保持 prompt 字节稳定，**别动不动改 system 里一个标点**
- ✅ 评估请求间隔，决定 5m 还是 1h TTL
- ✅ 多轮对话时复用同一个会话，别频繁开新会话

### 想精细控制

- 用显式断点，最多 4 个
- 在分层稳定性的边界打断点（system 后、工具后、文档后）
- 监控 `cache_read_input_tokens` / 总输入 token 的比例

### 会破坏缓存的常见操作

- ❌ 每次请求往 system prompt 里塞当前时间
- ❌ 在 prompt 中间插入用户名/会话 ID 这种变量（应该放到 user message 里）
- ❌ 频繁切换 thinking 开关
- ❌ 不必要地改工具描述
- ❌ 用 `--resume` 之后改了 CLAUDE.md（恢复的会话和当前 CLAUDE.md 拼起来字节不一致）

---

## 十一、Bedrock / Vertex 等第三方平台

非官方 API 可能不完全兼容 `cache_control`：

### 常见报错
```
ValidationException: system.1.cache_control.***.scope: Extra inputs are not permitted
```

### 解法

设置 settings.json：
```json
{
  "env": {
    "CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS": "1",
    "DISABLE_PROMPT_CACHING": "1"
  }
}
```

代价：失去缓存优化，长对话成本变高。

---

## 一句话总结

> **Claude Code 默认开启**，用 `DISABLE_PROMPT_CACHING=1` 关掉；**API 默认不开**，靠 `cache_control: {type: "ephemeral"}` 标记开启。缓存以**前缀字节哈希**为键，按 tools→system→messages 顺序匹配，命中部分**只收 10% 的钱**、跳过 prefill 计算，但**网络传输和上下文 token 数都省不了**——这是个**算力和钱的优化，不是带宽和 token 数的优化**。

---

下一章 → [05 - 速查与实战](./05-速查与实战.md)：常用命令、配置模板、踩坑案例。
