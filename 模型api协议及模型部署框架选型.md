# 模型 API 协议及模型部署框架选型

本文整理当前主流的三大模型 API 协议（OpenAI Chat Completions、OpenAI Responses、Anthropic Messages）的区别，以及常见模型部署框架对这些协议的支持情况，供在 DeepSeek Harness（下称 DSH）中配置自定义模型 API 时选型参考。

## 1. 背景：DSH 中的 API 协议

DSH 的 **Settings → Models** 中：

- **DeepSeek 官方卡片**要求填写「兼容 Anthropic Messages 协议的 API 地址」——DSH 的 DeepSeek 适配器（`dsh-llm-deepseek`）使用 Anthropic Messages API 与端点通信。
- **自定义模型 API** 的「API 协议」提供三种选择，存入 `cordis.patch.yml` 的值分别是：
  - `openai-completions` — OpenAI Chat Completions（`POST /v1/chat/completions`）
  - `openai-responses` — OpenAI Responses（`POST /v1/responses`）
  - `anthropic-messages` — Anthropic Messages（`POST /v1/messages`）
- **一个提供商只使用一种协议**；同一网关若同时提供两种协议，需要建立两个提供商。
- 协议必须选网关实际使用的那一种，选错会导致请求被拒绝或行为异常。

## 2. 三大协议概览

| 维度 | OpenAI Chat Completions | OpenAI Responses | Anthropic Messages |
|---|---|---|---|
| 端点 | `POST /v1/chat/completions` | `POST /v1/responses` | `POST /v1/messages` |
| 定位 | 事实上的行业标准，兼容面最广 | OpenAI 新一代接口，原生支持推理与有状态会话 | Anthropic 原生接口，Claude 生态与 agentic 工具链（如 Claude Code）使用 |
| 认证 | `Authorization: Bearer <key>` | `Authorization: Bearer <key>` | `x-api-key` + `anthropic-version: 2023-06-01`（多数实现也接受 Bearer） |
| 系统提示 | `messages` 中 `role: "system"` 消息 | 顶层 `instructions` 字段 | 顶层 `system` 字段（字符串或 content block 数组，不在 `messages` 内），可挂 `cache_control` 做前缀缓存 |
| `max_tokens` | 可选 | `max_output_tokens`，可选 | **必填** |
| 消息内容 | 字符串或 parts 数组（`text` / `image_url`） | input items 数组（字符串或结构化 item） | **content block 数组**：`text` / `image` / `tool_use` / `tool_result` / `thinking` 等 |
| 工具定义 | `tools[].function.parameters`（JSON Schema） | `tools[].function.parameters` | `tools[].input_schema`（JSON Schema） |
| `tool_choice` | `auto` / `none` / `required` / 指定函数 | 类似 OpenAI | `{type: auto / any / tool / none}` |
| 工具结果回传 | 独立的 `role: "tool"` 消息 + `tool_call_id` | function_call_output item | 助手消息中的 `tool_use` block + 用户消息中的 `tool_result` block |
| 思考/推理 | 非标准，各家扩展（如 `reasoning_content`、`reasoning_effort`） | 原生 `reasoning` output item | 原生 `thinking` block（请求端 `thinking: {type: enabled, budget_tokens}`，响应带签名） |
| 流式事件 | `chat.completion.chunk`，`choices[].delta` 增量 | `response.created` / `response.output_text.delta` / `response.completed` 等 | `message_start` → `content_block_start` → `content_block_delta`（`text_delta` / `thinking_delta` / `input_json_delta`）→ `content_block_stop` → `message_stop` |
| 结束原因 | `finish_reason`（`stop` / `tool_calls` / `length`） | `status` / `incomplete_details` | `stop_reason`（`end_turn` / `max_tokens` / `tool_use` / `pause_turn`） |
| 有状态会话 | 无状态 | 支持（`previous_response_id`、`store`） | 无状态 |
| 附加能力 | — | — | 独立的 token 计数端点 `POST /v1/messages/count_tokens` |

## 3. Anthropic Messages 与 OpenAI 格式的核心区别

两者语义等价但报文结构不同，不能只换 base URL 混用（除非端点做了协议转换）：

1. **系统提示位置不同**：OpenAI 把 system 放在 `messages` 数组里（`role: "system"`）；Anthropic 放在顶层 `system` 字段，`messages` 里只有 user/assistant 轮次。
2. **内容是类型化 block**：OpenAI 的 `content` 通常是字符串（或多模态 parts）；Anthropic 的 `content` 必须是带 `type` 的 block 数组（`{"type": "text", "text": "..."}`），文本、图片、工具调用、思考全部用 block 表达。
3. **`max_tokens` 必填**：Anthropic Messages 不允许省略，OpenAI 可省略。
4. **工具协议字段名不同**：工具入参 schema 在 OpenAI 叫 `parameters`，Anthropic 叫 `input_schema`；工具结果在 OpenAI 是独立 `role: "tool"` 消息，在 Anthropic 是同一条助手/用户消息里的 `tool_use` / `tool_result` block。
5. **流式事件完全不同**：OpenAI 流是一串 `chat.completion.chunk`；Anthropic 流是带块状态机的 SSE 事件序列（`message_start`、`content_block_delta` 等），客户端解析逻辑不通用。
6. **思考块一等公民**：Anthropic 协议中 `thinking` block 及其签名是协议组成部分（可与 `thinking` 请求参数联动）；OpenAI Chat Completions 的推理表达不是统一标准（Responses 协议才原生化了 `reasoning` item）。
7. **版本与缓存头**：Anthropic 要求 `anthropic-version` 头，前缀缓存通过 `cache_control` block 显式声明；OpenAI 无对应要求。

> 简明字段对照（Anthropic ↔ OpenAI）：`system` 字段 ↔ `role: system` 消息；`content` block ↔ `content` 字符串/parts；`tools[].input_schema` ↔ `tools[].function.parameters`；`tool_use`/`tool_result` block ↔ `tool_calls`/`role: tool` 消息；`stop_reason` ↔ `finish_reason`；`max_tokens` 必填 ↔ 可选。

## 4. 模型部署框架协议支持矩阵

| 部署框架 | Chat Completions | Responses | Anthropic Messages | 说明 |
|---|---|---|---|---|
| **vLLM** | ✅ `/v1/chat/completions`、`/v1/completions` | ✅ `/v1/responses`（依版本） | ✅ `/v1/messages` + `/v1/messages/count_tokens`（`vllm.entrypoints.anthropic`） | 三协议齐备；工具调用、思考、视觉依赖模型与版本 |
| **SGLang** | ✅ `/v1/chat/completions`、`/v1/completions` | ✅（Model Gateway 路由提供 `/v1/responses`，单机以版本为准） | ✅ `/v1/messages` + `count_tokens`，服务器默认注册、无需额外开关 | 复用与 OpenAI 端点相同的 chat template 与解析器 |
| **Ollama** | ✅ `/v1/chat/completions`、`/v1/completions`（另有原生 `/api/chat`） | ✅ v0.13.3+，仅无状态子集 | ✅ `/v1/messages`，**子集**：不支持 `count_tokens`、`tool_choice`、`metadata`、`cache_control`、Batches | 本地桌面/单机最易用；三协议都有 |
| **llama.cpp（llama-server）** | ✅ `/v1/chat/completions`、`/v1/completions`、`/v1/embeddings` | ✅ `/v1/responses`（内部转 chat completions，无 `previous_response_id`） | ✅ `/v1/messages` + `count_tokens`（2026-01 起，内部转换实现） | 轻量 CPU/GPU 本地推理；协议转换层齐全 |
| **LM Studio** | ✅ `/v1/chat/completions`、`/v1/completions`、`/v1/embeddings` | ✅ `/v1/responses`（支持 `previous_response_id` 状态化） | ✅ `/v1/messages` | 另有原生 `/api/v1/chat`（0.4.0+）；桌面端三协议全支持 |
| **Hugging Face TGI** | ✅ `/v1/chat/completions`（1.4.0+） | ❌ | ❌ | 另有原生 `/generate` API；选型时只能配 `openai-completions` |
| **TensorRT-LLM** | ✅ `/v1/chat/completions`、`/v1/completions` | ❌（未见提供） | ❌（源码明确不留 `/v1/messages` 路径） | 只能配 `openai-completions`，或前置协议转换层 |
| **NVIDIA NIM（vLLM 后端）** | ✅ | ✅ `/v1/responses`（含查询/取消） | ✅ `/v1/messages` + `count_tokens` | 继承 vLLM 的三协议能力 |

图例：✅ 官方文档确认提供；❌ 官方未提供（不代表第三方桥接不可用）。

### 4.1 只支持 OpenAI 协议的框架，如何对接 Anthropic 客户端

- **协议转换 sidecar**：如 `siliconflow/anthropic-adapter`，把 `/v1/messages` 翻译为后端的 `/v1/chat/completions`，任何 OpenAI 兼容后端（vLLM、SGLang、TGI 等）都能对外提供 Anthropic 协议，覆盖流式、工具调用与思考映射。
- **网关层转换**：LiteLLM 等聚合网关可在统一入口对外暴露多种协议，详见其官方文档。
- 反方向（Anthropic 后端对 OpenAI 客户端）同理由网关完成。

## 5. 在 DSH 中的选型建议

1. **先确认端点实际说哪种协议，再选 DSH 的 API 协议**，两者必须一致：
   - 自建 vLLM / SGLang / Ollama / llama.cpp / LM Studio：三种协议基本都能对上，按客户端偏好选即可；
   - TGI / TensorRT-LLM：只能选 `openai-completions`；
   - DeepSeek 官方卡片或 Claude 生态端点：Messages 协议（`anthropic-messages`）。
2. **一个端点一种协议建一个提供商**；同一网关暴露两种协议就建两个提供商（Provider ID 不同）。
3. **OpenAI 协议下网关拒绝请求**（key 和 URL 都对仍失败）时，优先在 `cordis.patch.yml` 路由上加兼容开关：`compat.supportsDeveloperRole: false`、`compat.maxTokensField: max_tokens`；详见[模型配置指南](docs/user/guide/providers.md#request-compatibility)。
4. **推理等级**：`reasoning_effort` 是 OpenAI 风格参数；Anthropic 协议走 `thinking` block。自定义模型需在 Models 页开启 **Reasoning levels** 并按端点文档填 API 值。
5. **本地部署快速验证**：先用 `curl` 直接打端点（如 `POST /v1/messages` 或 `/v1/chat/completions`）确认可用，再填入 DSH，避免把协议选错误判为 DSH 问题。

## 6. 参考链接

- DSH 模型配置：[docs/user/guide/providers.md](docs/user/guide/providers.md)
- DSH 公开部署（反向代理）：[docs/user/guide/public-deployments.md](docs/user/guide/public-deployments.md)
- vLLM Anthropic 端点：<https://docs.vllm.ai/en/stable/api/vllm/entrypoints/anthropic>
- SGLang Anthropic 兼容 API：<https://docs.sglang.io/docs/basic_usage/anthropic_api>
- Ollama OpenAI 兼容：<https://docs.ollama.com/api/openai-compatibility>；Anthropic 兼容：<https://docs.ollama.com/api/anthropic-compatibility>
- llama.cpp server API：<https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md>
- LM Studio OpenAI/Anthropic 兼容：<https://lmstudio.ai/docs/developer/openai-compat>、<https://lmstudio.ai/docs/developer/anthropic-compat/messages>
- TGI API 参考：<https://huggingface.co/docs/text-generation-inference/reference/api_reference>
- Anthropic Messages API：<https://docs.anthropic.com/en/api/messages>
- OpenAI Chat Completions / Responses：<https://platform.openai.com/docs/api-reference/chat>、<https://platform.openai.com/docs/api-reference/responses>

> 注：各框架协议支持随版本演进，表中结论基于 2026-10 各项目官方文档；实际选型前请以所用版本的官方文档为准。
