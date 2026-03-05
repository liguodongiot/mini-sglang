# Mini-SGLang API 文档

Mini-SGLang 是一个轻量级（约 5,000 行）高性能 LLM 推理框架，支持多种模型和注意力后端。

## 目录

- [Python API](#python-api)
  - [LLM 类](#llm-类)
  - [SamplingParams 类](#samplingparams-类)
  - [Engine 类](#engine-类)
  - [Scheduler 类](#scheduler-类)
- [HTTP API](#http-api)
  - [端点概览](#端点概览)
  - [POST /generate](#post-generate)
  - [POST /v1/chat/completions](#post-v1chatcompletions)
  - [GET /v1/models](#get-v1models)
- [数据模型](#数据模型)
- [配置选项](#配置选项)

---

## Python API

### LLM 类

离线批处理推理的主要接口。

```python
from minisgl.llm import LLM
from minisgl.core import SamplingParams

llm = LLM(
    model_path="meta-llama/Llama-2-7b-chat-hf",
    dtype=torch.bfloat16,
)
```

#### 构造函数参数

| 参数 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `model_path` | `str` | Required | 模型权重路径，可以是本地文件夹或 Hugging Face repo ID |
| `dtype` | `torch.dtype` | `torch.bfloat16` | 模型权重的数据类型 |
| `**kwargs` | - | - | 其他参数传递给 `SchedulerConfig` |

#### 方法

##### `generate(prompts, sampling_params)`

执行批量推理。

**参数：**

- `prompts`: `List[str] | List[List[int]]` - 输入提示文本或 token ID 列表
- `sampling_params`: `SamplingParams | List[SamplingParams]` - 采样参数

**返回：**

```python
List[Dict[str, str | List[int]]]
```

每个元素包含：
- `text`: 生成的文本
- `token_ids`: 生成的 token ID 列表

**示例：**

```python
from minisgl.llm import LLM
from minisgl.core import SamplingParams

llm = LLM(model_path="meta-llama/Llama-2-7b-chat-hf")

# 单个提示
params = SamplingParams(temperature=0.7, max_tokens=256)
results = llm.generate(["Hello, how are you?"], params)
print(results[0]["text"])

# 批量提示
params_list = [
    SamplingParams(temperature=0.7, max_tokens=256),
    SamplingParams(temperature=0.9, max_tokens=128),
]
results = llm.generate(
    ["What is the capital of France?", "What is 2+2?"],
    params_list
)
```

---

### SamplingParams 类

控制文本生成的采样参数。

```python
from minisgl.core import SamplingParams

params = SamplingParams(
    temperature=0.7,
    top_k=50,
    top_p=0.9,
    max_tokens=512,
    ignore_eos=False,
)
```

#### 属性

| 属性 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `temperature` | `float` | 0.0 | 采样温度。值为 0 时使用贪婪解码 |
| `top_k` | `int` | -1 | Top-K 采样。-1 表示禁用 |
| `top_p` | `float` | 1.0 | Nucleus 采样阈值 |
| `ignore_eos` | `bool` | False | 是否忽略 EOS token |
| `max_tokens` | `int` | 1024 | 最大生成 token 数 |

#### 属性 (只读)

- `is_greedy`: `bool` - 是否使用贪婪解码（temperature <= 0 或 top_k == 1，且 top_p == 1.0）

---

### Engine 类

底层推理引擎，管理模型加载、前向传播和采样。

**注意：** 通常不需要直接使用 `Engine` 类，推荐使用 `LLM` 类或 `Scheduler`。

```python
from minisgl.engine import Engine, EngineConfig
from minisgl.distributed import DistributedInfo

config = EngineConfig(
    model_path="meta-llama/Llama-2-7b-chat-hf",
    tp_info=DistributedInfo(0, 1),
    dtype=torch.bfloat16,
)

engine = Engine(config)
```

#### 构造函数参数

| 参数 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `config` | `EngineConfig` | Required | 引擎配置 |

---

### Scheduler 类

请求调度器，管理推理请求的调度和执行。

```python
from minisgl.scheduler import Scheduler, SchedulerConfig
from minisgl.distributed import DistributedInfo

config = SchedulerConfig(
    model_path="meta-llama/Llama-2-7b-chat-hf",
    tp_info=DistributedInfo(0, 1),
    dtype=torch.bfloat16,
)

scheduler = Scheduler(config)
```

---

## HTTP API

Mini-SGLang 提供 FastAPI 服务，支持 OpenAI 风格的 API。

### 端点概览

| 方法 | 路径 | 描述 |
|------|------|------|
| POST | `/generate` | 简单生成接口 |
| POST | `/v1/chat/completions` | OpenAI 风格的聊天完成接口 |
| GET | `/v1/models` | 获取可用模型列表 |
| GET | `/v1` | API 根路径健康检查 |

### 启动服务器

```bash
python -m minisgl \
    --model-path meta-llama/Llama-2-7b-chat-hf \
    --host 0.0.0.0 \
    --port 1919
```

---

### POST /generate

简单文本生成接口。

**请求体：**

```json
{
    "prompt": "Hello, how are you?",
    "max_tokens": 256,
    "ignore_eos": false
}
```

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `prompt` | `string` | Yes | 输入提示文本 |
| `max_tokens` | `integer` | Yes | 最大生成 token 数 |
| `ignore_eos` | `boolean` | No | 是否忽略 EOS token，默认 false |

**响应：**

流式响应（Server-Sent Events）

```
data: Hello, I'm doing well! How can I help you today?
data: [DONE]
```

---

### POST /v1/chat/completions

OpenAI 风格的聊天完成接口。

**请求体：**

```json
{
    "model": "llama-2-7b",
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is Python?"}
    ],
    "max_tokens": 256,
    "temperature": 0.7,
    "top_p": 0.9,
    "top_k": 50,
    "stream": true,
    "stop": [],
    "presence_penalty": 0.0,
    "frequency_penalty": 0.0,
    "ignore_eos": false
}
```

| 字段 | 类型 | 必填 | 描述 |
|------|------|------|------|
| `model` | `string` | Yes | 模型名称（用于兼容性） |
| `messages` | `array` | No* | 聊天消息列表 |
| `prompt` | `string` | No* | 原始提示文本（与 messages 二选一） |
| `max_tokens` | `integer` | No | 最大生成 token 数，默认 16 |
| `temperature` | `float` | No | 采样温度，默认 1.0 |
| `top_p` | `float` | No | Nucleus 采样阈值，默认 1.0 |
| `top_k` | `integer` | No | Top-K 采样，默认 -1（禁用） |
| `stream` | `boolean` | No | 是否流式输出，默认 false |
| `stop` | `array` | No | 停止词列表 |
| `presence_penalty` | `float` | No | 存在惩罚，默认 0.0 |
| `frequency_penalty` | `float` | No | 频率惩罚，默认 0.0 |
| `ignore_eos` | `boolean` | No | 是否忽略 EOS，默认 false |

**注意：** `messages` 和 `prompt` 必须提供其中之一。

**响应（非流式）：**

```json
{
    "id": "cmpl-0",
    "object": "text_completion",
    "created": 1234567890,
    "model": "llama-2-7b",
    "choices": [
        {
            "index": 0,
            "text": "Python is a high-level programming language...",
            "finish_reason": "stop"
        }
    ]
}
```

**响应（流式）：**

```
data: {"id":"cmpl-0","object":"text_completion.chunk","choices":[{"delta":{"role":"assistant","content":"Python"},"index":0,"finish_reason":null}]}

data: {"id":"cmpl-0","object":"text_completion.chunk","choices":[{"delta":{"content":" is"},"index":0,"finish_reason":null}]}

data: {"id":"cmpl-0","object":"text_completion.chunk","choices":[{"delta":{},"index":0,"finish_reason":"stop"}]}

data: [DONE]
```

---

### GET /v1/models

获取可用模型列表。

**响应：**

```json
{
    "object": "list",
    "data": [
        {
            "id": "meta-llama/Llama-2-7b-chat-hf",
            "object": "model",
            "created": 1234567890,
            "owned_by": "mini-sglang",
            "root": "meta-llama/Llama-2-7b-chat-hf"
        }
    ]
}
```

---

## 数据模型

### Req

推理请求数据结构。

```python
from minisgl.core import Req, SamplingParams
import torch

req = Req(
    input_ids=torch.tensor([1, 2, 3], dtype=torch.int32, device="cpu"),
    table_idx=0,
    cached_len=0,
    output_len=512,
    uid=0,
    sampling_params=SamplingParams(),
    cache_handle=None,
)
```

| 属性 | 类型 | 描述 |
|------|------|------|
| `input_ids` | `torch.Tensor` | 输入 token ID 列表（CPU tensor） |
| `table_idx` | `int` | 请求在页面表中的索引 |
| `cached_len` | `int` | 已缓存的 token 长度 |
| `device_len` | `int` | 当前设备上的 token 长度 |
| `max_device_len` | `int` | 设备上最大 token 长度 |
| `output_len` | `int` | 请求的最大输出长度 |
| `uid` | `int` | 请求唯一标识符 |
| `sampling_params` | `SamplingParams` | 采样参数 |
| `cache_handle` | `BaseCacheHandle` | KV 缓存句柄 |

### Batch

推理批次数据结构。

```python
from minisgl.core import Batch, Req, Batch

batch = Batch(
    reqs=[req1, req2],
    phase="prefill"  # 或 "decode"
)
```

| 属性 | 类型 | 描述 |
|------|------|------|
| `reqs` | `List[Req]` | 批次中的请求列表 |
| `phase` | `Literal["prefill", "decode"]` | 推理阶段 |
| `input_ids` | `torch.Tensor` | 批次输入 token |
| `out_loc` | `torch.Tensor` | 输出位置 |
| `padded_reqs` | `List[Req]` | 填充后的请求列表 |
| `attn_metadata` | `BaseAttnMetadata` | 注意力元数据 |

---

## 配置选项

### 命令行参数

| 参数 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `--model-path` | `str` | Required | 模型路径或 Hugging Face repo ID |
| `--dtype` | `str` | auto | 数据类型：auto, float16, bfloat16, float32 |
| `--tensor-parallel-size` | `int` | 1 | Tensor 并行度 |
| `--max-running-requests` | `int` | 256 | 最大运行请求数 |
| `--max-seq-len-override` | `int` | None | 最大序列长度覆盖 |
| `--memory-ratio` | `float` | 0.9 | GPU 显存用于 KV 缓存的比例 |
| `--dummy-weight` | `bool` | False | 使用随机权重（用于测试） |
| `--disable-pynccl` | `bool` | False | 禁用 PyNCCL tensor 并行 |
| `--host` | `str` | 127.0.0.1 | 服务器监听地址 |
| `--port` | `int` | 1919 | 服务器监听端口 |
| `--cuda-graph-max-bs` | `int` | None | CUDA Graph 最大批大小 |
| `--num-tokenizer` | `int` | 0 | Tokenizer 进程数，0 表示与 detokenizer 共享 |
| `--max-prefill-length` | `int` | 8192 | Prefill 最大 chunk 大小 |
| `--num-pages` | `int` | None | KVCache 页面数覆盖 |
| `--attention-backend` | `str` | auto | 注意力后端：flashattn, flashinfer, auto |
| `--cache-type` | `str` | radix | KV 缓存管理策略：radix, naive |
| `--shell-mode` | `bool` | False | 交互式 Shell 模式 |

### 环境变量

| 变量 | 值 | 描述 |
|------|-----|------|
| `LOG_LEVEL` | DEBUG, INFO, WARNING, ERROR | 日志级别 |
| `LOG_PID` | 0 或 1 | 是否在日志中包含进程 ID |
| `MINISGL_DISABLE_OVERLAP_SCHEDULING` | - | 禁用重叠调度 |

---

## 使用示例

### Python 离线推理

```python
import torch
from minisgl.llm import LLM
from minisgl.core import SamplingParams

llm = LLM(
    model_path="meta-llama/Llama-2-7b-chat-hf",
    dtype=torch.bfloat16,
)

params = SamplingParams(
    temperature=0.7,
    top_p=0.9,
    max_tokens=256,
)

results = llm.generate(
    ["Explain quantum computing in simple terms:", "What is machine learning?"],
    [params, params]
)

for result in results:
    print(f"Generated: {result['text']}")
    print(f"Token IDs: {result['token_ids'][:10]}...")
```

### HTTP API 调用

```bash
# 聊天完成
curl -X POST http://localhost:1919/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ],
    "max_tokens": 256,
    "temperature": 0.7
  }'

# 流式聊天完成
curl -X POST http://localhost:1919/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b",
    "messages": [{"role": "user", "content": "Hello!"}],
    "stream": true
  }'
```

### 交互式 Shell 模式

```bash
python -m minisgl --model-path meta-llama/Llama-2-7b-chat-hf --shell-mode
```

在 Shell 模式中：
- 输入文本直接发送推理
- `/reset` - 重置对话历史
- `/exit` - 退出
