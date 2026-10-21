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

### 4.1 vLLM 三种协议的部署命令

一条 `vllm serve` 命令即可同时暴露三种协议的路由（默认端口 8000，无需为每种协议单独启动）：

```sh
# 安装（需对应 CUDA 环境）
pip install vllm

# 启动：Chat Completions、Responses、Anthropic Messages 同时可用
vllm serve Qwen/Qwen3-8B \
  --host 0.0.0.0 --port 8000 \
  --served-model-name qwen3-8b \
  --api-key sk-local \
  --enable-auto-tool-choice --tool-call-parser qwen3 \
  --reasoning-parser qwen3
```

说明：

- `--host 0.0.0.0` 是 **vLLM 自己的监听参数**，允许通配符（与 DSH 拒绝通配符是两回事）；只想本机访问就用 `--host 127.0.0.1`。
- `--served-model-name` 固定对外的模型 ID（请求 `model` 字段用它），`--api-key` 要求请求带 `Authorization: Bearer sk-local`。
- **工具调用**需要 `--enable-auto-tool-choice --tool-call-parser <解析器>`，**思考内容**需要 `--reasoning-parser <解析器>`；解析器按模型选择（如 `qwen3`、`glm4`、`deepseek_v3`），DSH 作为 agent 使用时这两项必备。
- Docker 等价启动：

  ```sh
  docker run --gpus all -p 8000:8000 \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    vllm/vllm-openai:latest \
    --model Qwen/Qwen3-8B --served-model-name qwen3-8b --api-key sk-local \
    --enable-auto-tool-choice --tool-call-parser qwen3 --reasoning-parser qwen3
  ```

启动后逐条验证三种协议（都通了再往 DSH 里配）：

```sh
# 1) OpenAI Chat Completions → DSH 选 openai-completions
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Authorization: Bearer sk-local" -H "Content-Type: application/json" \
  -d '{"model":"qwen3-8b","messages":[{"role":"user","content":"hi"}]}'

# 2) OpenAI Responses → DSH 选 openai-responses
curl http://127.0.0.1:8000/v1/responses \
  -H "Authorization: Bearer sk-local" -H "Content-Type: application/json" \
  -d '{"model":"qwen3-8b","input":"hi"}'

# 3) Anthropic Messages → DSH 选 anthropic-messages
curl http://127.0.0.1:8000/v1/messages \
  -H "x-api-key: sk-local" -H "anthropic-version: 2023-06-01" -H "Content-Type: application/json" \
  -d '{"model":"qwen3-8b","max_tokens":128,"messages":[{"role":"user","content":"hi"}]}'
```

DSH 中对应填写（同一 vLLM 可建三个提供商，Provider ID 各不相同）：

| 验证通过的端点 | DSH「API 协议」 | Base URL 示例 |
|---|---|---|
| `/v1/chat/completions` | `openai-completions` | `http://192.168.0.213:8000/v1` |
| `/v1/responses` | `openai-responses` | `http://192.168.0.213:8000/v1` |
| `/v1/messages` | `anthropic-messages` | 以 curl 实测能通的地址为准（Anthropic SDK 惯例为服务器根 `http://192.168.0.213:8000`） |

版本注意——三种协议的最低 vLLM 版本：

| 协议 | 最低版本 | 说明 |
|---|---|---|
| `/v1/chat/completions` | 所有版本 | vLLM 的基础能力 |
| `/v1/responses` | **v0.10.0**（2025-07） | v0.10.x 需加 `--enable-responses-api` 才注册路由；之后版本默认注册 |
| `/v1/messages` | **v0.11.1**（2025-11-18） | Anthropic 路由 PR #22627/#27882 于 2025-11-04 确认合入后的首个发布；v0.11.2 / v0.12.0 起必然包含 |

判定与升级：

```sh
vllm --version                 # < v0.11.1 则 /v1/messages 一定没有
vllm serve --help=responses    # 有 --enable-responses-api 说明 /v1/responses 需显式开启
```

- **最权威的判定是启动日志的 `Available routes` 列表**：出现 `/v1/messages`、`/v1/responses` 才算三协议齐备；路由缺失时先查版本/开关，再考虑升级（`pip install -U vllm`，升级前确认 CUDA/驱动兼容）。
- 版本不满足期间，可先用已就绪的 `/v1/chat/completions` 在 DSH 配 `openai-completions` 正常使用，升级后再补建另外两个提供商。
- 新版另有可调参数 `--anthropic-disabled-thinking-effort`（仅作用于 `/v1/messages` 的 `thinking: disabled` 请求）。
- 各路由的支持细节（工具调用、思考、多模态）依模型与版本而定，以实际 curl 结果为准。

环境栈匹配与升级约束（以 CUDA 12.4 + torch 2.7.0 + Python 3.12 这类旧栈为例）：

- **旧栈通常对应旧版 vLLM**：CUDA 12.4 + torch 2.7.0 是 vLLM v0.8.x–v0.9.x 时代的默认组合，此时 `/v1/responses`、`/v1/messages` 都不存在——这正是启动日志路由缺失的根本原因，加开关无济于事，只能升级 vLLM。
- **升级时 torch 与 CUDA 变体由 vLLM wheel 接管**：pip wheel 自带匹配的 torch 与 CUDA 运行库，本机旧 torch（如 2.7.0）和 CUDA toolkit（如 12.4）不构成障碍；**不要**手动 pin 旧 torch，建议新建干净的 conda/venv 环境安装。
- **真正的硬约束是 NVIDIA 驱动**（`nvidia-smi` 查驱动版本与上限 CUDA 版本）：

  | 目标 vLLM | CUDA 变体 | 驱动要求（Linux） |
  |---|---|---|
  | ≥ v0.11.1（`/v1/messages` 最低要求） | cu126 | ≥ 560.28 |
  | | cu128（该版本默认） | ≥ 570 |
  | 更高版本（默认 cu129 及以后） | cu129+ | 12.x 小版本兼容一般可用，以实际启动为准 |

- 升级后重新验证启动日志 `Available routes`，三协议齐备后再在 DSH 补建提供商。驱动无法升级时，停在旧版并用 `openai-completions` 是唯一稳妥方案。

### 4.2 只支持 OpenAI 协议的框架，如何对接 Anthropic 客户端

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
