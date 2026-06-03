# QwenPaw — token_ids & log_probs 支持

让 [QwenPaw](https://github.com/agentscope-ai/QwenPaw) 在运行过程中返回每个请求的 `token_ids` 和 `log_probs`，包括中间步骤（Thought、Action、Observation）。

## 快速应用

将 `scripts/` 下的文件覆盖到开源仓对应位置：

| 脚本 | 目标路径（开源仓内） |
|------|---------------------|
| `scripts/_model_response.py` | `agentscope/model/_model_response.py` |
| `scripts/_openai_model.py` | `agentscope/model/_openai_model.py` |
| `scripts/_react_agent.py` | `agentscope/agent/_react_agent.py` |
| `scripts/openai_provider.py` | `qwenpaw/providers/openai_provider.py` |

```bash
# 在 QwenPaw 开源仓根目录执行
cp scripts/_model_response.py agentscope/model/_model_response.py
cp scripts/_openai_model.py   agentscope/model/_openai_model.py
cp scripts/_react_agent.py    agentscope/agent/_react_agent.py
cp scripts/openai_provider.py qwenpaw/providers/openai_provider.py
```

## 启用方法

默认 `enable_logprobs=False`（安全）。启用需修改 `qwenpaw/providers/openai_provider.py`：

```python
# 第 192 行附近
enable_logprobs=False,   # 改为 True
```

重启 QwenPaw 后，`token_ids` 和 `log_probs` 会自动收集到每个步骤的 `Msg.metadata` 中：

```python
msg.metadata["token_ids"]   # list[list[int]]
msg.metadata["log_probs"]   # list[float]
```

## 兼容性

| 服务 | 支持 logprobs |
|------|:---:|
| vLLM | ✅ |
| OpenAI 官方 | ✅ |
| DashScope（阿里云） | ❌ |

> DashScope 不支持 logprobs，需切换到 vLLM 等兼容服务。

## 修改详情

### 1. `_model_response.py` — ChatResponse 数据结构

新增两个字段（默认值 `None`，向后兼容）：

```python
token_ids: list[list[int]] | None = field(default_factory=lambda: None)
log_probs: list[float] | None = field(default_factory=lambda: None)
```

### 2. `_openai_model.py` — OpenAI 模型调用

- 新增 `enable_logprobs` 参数（默认 `False`）
- 条件性添加 `logprobs=True, top_logprobs=1` 到 API 请求
- 流式/非流式路径均从 `choice.logprobs.content` 提取 token bytes 和 logprob
- `structured_model` 模式下自动移除 logprobs 参数（OpenAI 不支持同时使用）

### 3. `_react_agent.py` — ReAct Agent 推理循环

- `_reasoning()` 和 `_summarizing()` 方法中收集 token_ids/log_probs
- 存储到 `msg.metadata["token_ids"]` 和 `msg.metadata["log_probs"]`
- 通过 `logger.info` 输出每个步骤的 token 统计

### 4. `openai_provider.py` — QwenPaw OpenAI Provider

- `get_chat_model_instance()` 传递 `enable_logprobs=False`（默认关闭）

## 数据流

```
OpenAI API (logprobs=True, top_logprobs=1)
  │
  ▼
OpenAIChatModel.__call__()
  │
  ▼
ChatResponse.token_ids / ChatResponse.log_probs
  │
  ▼
ReActAgent._reasoning() / _summarizing()
  │
  ▼
Msg.metadata["token_ids"] / Msg.metadata["log_probs"]
```
