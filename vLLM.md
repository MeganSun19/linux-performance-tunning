# vLLM 推理部署学习笔记

> 学习日期：2026年5月
> 实验环境：Kaggle Notebook (2× Tesla T4 GPU，每卡 16GB)

---

## 1. 为什么需要 vLLM？

### 1.1 大模型推理的根本痛点：KV Cache

大模型生成文本是**逐 token 自回归**的——预测第 100 个词时，需要前 99 个词的 Attention 计算结果（Key 和 Value 矩阵）。

- **朴素做法**：每次都重新计算前 99 个词的 KV，极慢
- **KV Cache**：把前 99 个词的 KV 矩阵缓存在显存里，每次只算新 token 的 KV 并拼接进去

KV Cache 用**显存空间换取计算时间**，是推理加速的核心手段。

### 1.2 传统 KV Cache 的问题：显存碎片化

传统框架（如 HuggingFace transformers）给每个请求分配一块**连续的显存**。但生成长度是未知的，导致：

- **内部碎片**：提前分配大块显存，实际用不完，白白占着
- **外部碎片**：请求结束后留下大量不连续的小碎片，新请求塞不进去
- 实测：高达 **60~80% 的 KV Cache 显存被浪费**

### 1.3 vLLM 的解法：PagedAttention（分页注意力）

借鉴操作系统**虚拟内存分页**机制：

- 把 KV Cache 切成固定大小的**块（Block）**，例如每块存 16 个 token
- 物理显存上**不连续**，通过一张**页表（Block Table）**把这些块逻辑上串起来
- 每个请求只按需分配块，用完一块再申请下一块

**收益**：显存浪费率降到 **4% 以下**，省出来的显存同时处理更多并发请求。

### 1.4 Continuous Batching（持续批处理）

- **传统静态批处理**：凑齐 N 个请求一起跑，最慢的那个没算完，其他人都得等
- **Continuous Batching**：谁先算完谁先走，立刻补入新请求，GPU 利用率最大化

两者结合，vLLM 比 HuggingFace transformers 的吞吐量高 **2~4 倍**。

---

## 2. vLLM 在 AI Infra 中的位置

### 2.1 推理栈层次

```
用户请求
    ↓
负载均衡（Nginx / K8s Service）
    ↓
vLLM API Server          ← 今天学习的这层
    ↓                      （OpenAI 兼容接口 + PagedAttention + Continuous Batching）
GPU 推理引擎
    ↓
模型权重（Qwen / Llama / 内部模型）
```

### 2.2 与训练框架的对比


| 维度   | 训练（DDP/FSDP）    | 推理（vLLM）                  |
| ---- | --------------- | ------------------------- |
| 关注指标 | 吞吐量、显存占用        | TTFT（首字延迟）、TPOT（生成速度）、并发数 |
| 通信模式 | AllReduce 同步梯度  | AllReduce 同步张量并行中间结果      |
| 显存瓶颈 | 权重 + 梯度 + 优化器状态 | 权重 + KV Cache             |


---

## 3. OpenAI 兼容 API：为什么重要

### 3.1 不是聊天机器人，是行业标准接口

OpenAI 2020 年开放了这套 HTTP 接口规范，格式简单好用，行业默认跟进，成为**事实标准（de facto standard）**。现在 Anthropic、Google、Mistral、各大开源框架全部实现了同一套接口。

```
OpenAI 开放 API 规范
    ↓
行业全跟进（不是官方强制，是大家都跟着抄）
    ↓
vLLM / Ollama / LiteLLM / TGI 全部兼容
    ↓
所有工具链（LangChain / LlamaIndex / 各业务系统）直接通
```

### 3.2 对 AI Infra 工程师的实际价值

**标准化部署**：业务侧永远只看到 `/v1/chat/completions`，底下换什么模型、跑几张卡、怎么扩容，全是 Infra 团队的事。

```python
# 原来连 OpenAI
client = OpenAI(api_key="sk-xxx")

# 换成本地 vLLM，代码一行不用改
client = OpenAI(
    api_key="any-string",                    # vLLM 不校验，随便填
    base_url="http://localhost:8000/v1"      # 指向本地服务
)
```

```
研究员/业务团队              AI Infra 团队
──────────────              ─────────────
"我要用 Llama3"        →    vllm serve Llama3
"我要换 Qwen"          →    vllm serve Qwen       ← 业务代码零改动
"我要用内网私有模型"   →    vllm serve InternalModel
```

### 3.3 两种使用姿势


| 模式                    | 用法                          | 适用场景           |
| --------------------- | --------------------------- | -------------- |
| **Offline Inference** | `LLM.generate()` Python API | 批量处理、数据标注、离线评测 |
| **API Server**        | `vllm serve` 起 HTTP 服务      | 生产部署、对接业务系统    |


---

## 4. 张量并行（Tensor Parallelism）

### 4.1 原理

把模型的**权重矩阵按列/行切分**到多张 GPU，每张卡只存和计算一部分，最后通过 AllReduce 合并结果。

```
单卡（TP=1）：
GPU 0 存全部权重 [W]，计算 x·W

双卡（TP=2）：
GPU 0 存 [W 左半]，计算 x·W_left
GPU 1 存 [W 右半]，计算 x·W_right
    ↓ AllReduce
合并结果
```

### 4.2 适用场景

- **小模型（<7B）**：单卡放得下，TP=2 没必要，反而因为通信开销略微变慢
- **大模型（70B+）**：单卡装不下，TP=8 把权重分到 8 张卡，才能跑起来

```
70B 模型，FP16 → 单卡需要 140GB
8 张 A100(80GB) + TP=8 → 每卡只需 ~17.5GB  ✅
```

### 4.3 实验结果分析


| 模式      | GPU 0     | GPU 1     | 总计        |
| ------- | --------- | --------- | --------- |
| 单卡 TP=1 | 13395 MiB | 3 MiB     | 13398 MiB |
| 双卡 TP=2 | 13545 MiB | 13545 MiB | 27090 MiB |


**双卡总占用反而更多**，原因：

1. **权重确实对半切了**（省了约 1.5GB/卡）
2. **KV Cache 没有切分**，每张卡都要保留完整 KV Cache
3. **通信 buffer**，AllReduce 需要额外显存

结论：对 1.5B 小模型，TP=2 意义不大；对单卡塞不下的大模型，TP 是唯一出路。

---

## 5. 动手实验记录

### 5.1 实验环境

- **平台**：Kaggle Notebook
- **硬件**：GPU T4 ×2（PCIe 连接，无 NVLink）
- **模型**：`Qwen/Qwen2.5-1.5B-Instruct`（FP16，约 3GB 权重）

### 5.2 Kaggle 安装踩坑

**问题**：直接 `pip install vllm` 后运行报错：

```
/usr/bin/ld: cannot find -lcuda: No such file or directory
```

**原因**：vLLM 启动时用 `ninja` JIT 编译 CUDA 内核，链接器找 `libcuda.so` 时只搜索标准路径，但 Kaggle 上 `libcuda.so` 在 `/usr/local/nvidia/lib64/`，不在搜索路径里。

**修复**：创建软链接把 `libcuda.so` 放到链接器能找到的地方：

```bash
!ln -sf /usr/local/nvidia/lib64/libcuda.so /usr/local/cuda-12.8/targets/x86_64-linux/lib/libcuda.so
!ln -sf /usr/local/nvidia/lib64/libcuda.so.1 /usr/local/cuda-12.8/targets/x86_64-linux/lib/libcuda.so.1
!ldconfig
```

**同时加环境变量**：

```python
import os
os.environ["VLLM_COMPILE_LEVEL"] = "0"   # 关掉所有编译优化
```

### 5.3 Lab 1：离线批量推理（最终脚本）

```python
import os
os.environ["VLLM_COMPILE_LEVEL"] = "0"

from vllm import LLM, SamplingParams

llm = LLM(
    model="Qwen/Qwen2.5-1.5B-Instruct",
    dtype="float16",
    gpu_memory_utilization=0.85,
    max_model_len=2048,
    enforce_eager=True,
)

sampling_params = SamplingParams(temperature=0.7, max_tokens=128)

prompts = [
    "What is PagedAttention in vLLM?",
    "Explain GPU memory in one sentence.",
    "What does tensor parallel mean?",
]

outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    print(f">>> {output.prompt}")
    print(f"    {output.outputs[0].text}")
    print()
```

**输出**（1.5B 小模型能力有限，答非所问属正常）：

```
>>> What is PagedAttention in vLLM?
    PagedAttention in vLLM is a class that provides a method to perform
    attention mechanism...

>>> Explain GPU memory in one sentence.
    GPU memory is a type of computer storage that allows for fast and
    efficient storage and processing of data.

>>> What does tensor parallel mean?
    Tensor parallel is a term in mathematics and physics that refers to
    the property of a tensor...（答成了广义相对论）
```

**显存占用（Lab 1 基准）**：

```
!nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv
index, memory.used [MiB], memory.total [MiB]
0, 13395 MiB, 15360 MiB
1, 3 MiB, 15360 MiB
```

> ⚠️ `torch.cuda.memory_allocated()` 看不到 vLLM 的显存，因为 vLLM 用自己的显存管理器绕过了 PyTorch allocator。必须用 `nvidia-smi` 才能看到真实占用。

### 5.4 Lab 2：API Server + curl 调用（最终脚本）

**启动 Server**：

```python
import subprocess, time, urllib.request, os

os.environ["VLLM_COMPILE_LEVEL"] = "0"

server = subprocess.Popen(
    [
        "python", "-m", "vllm.entrypoints.openai.api_server",
        "--model", "Qwen/Qwen2.5-1.5B-Instruct",
        "--dtype", "float16",
        "--gpu-memory-utilization", "0.85",
        "--max-model-len", "2048",
        "--enforce-eager",
        "--port", "8000",
    ],
    stdout=open("/tmp/vllm.log", "w"),
    stderr=subprocess.STDOUT,
)

for i in range(60):
    time.sleep(3)
    try:
        urllib.request.urlopen("http://localhost:8000/health")
        print(f"✅ Server 就绪（{(i+1)*3}s）")
        break
    except:
        print(f"等待中... {(i+1)*3}s", end="\r")
```

**curl 调用**：

```python
import subprocess
result = subprocess.run([
    "curl", "-s", "http://localhost:8000/v1/chat/completions",
    "-H", "Content-Type: application/json",
    "-d", '{"model":"Qwen/Qwen2.5-1.5B-Instruct","messages":[{"role":"user","content":"What is KV Cache?"}],"max_tokens":100}'
], capture_output=True, text=True)
print(result.stdout)
```

**实际输出**（模型把 "KV Cache" 误判为钓鱼链接，属小模型能力问题）：

```json
{
  "model": "Qwen/Qwen2.5-1.5B-Instruct",
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "I'm sorry, I can't answer this question. This might be a scam attempt..."
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 34,
    "completion_tokens": 52,
    "total_tokens": 86
  }
}
```

**关键字段解读**：

- `finish_reason: "stop"` — 正常结束，不是被长度截断
- `prompt_tokens / completion_tokens` — 输入输出各消耗多少 token，计费和性能分析的依据
- 这套 JSON 格式和 OpenAI API **完全一致**

**关闭 Server**（Kaggle kernel 重启后 `server` 变量丢失，用系统命令）：

```bash
!pkill -f "vllm.entrypoints.openai.api_server"
```

### 5.5 Lab 3：双卡张量并行（最终脚本）

```python
import os
os.environ["VLLM_COMPILE_LEVEL"] = "0"

from vllm import LLM, SamplingParams

llm_tp2 = LLM(
    model="Qwen/Qwen2.5-1.5B-Instruct",
    dtype="float16",
    gpu_memory_utilization=0.85,
    max_model_len=2048,
    enforce_eager=True,
    tensor_parallel_size=2,
    distributed_executor_backend="mp",   # Kaggle 必须用 multiprocessing
)

sampling_params = SamplingParams(temperature=0.7, max_tokens=100)
outputs = llm_tp2.generate(["Explain tensor parallelism in one sentence."], sampling_params)
print(outputs[0].outputs[0].text)
```

**显存占用（Lab 3 双卡）**：

```
!nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv
index, memory.used [MiB], memory.total [MiB]
0, 13545 MiB, 15360 MiB
1, 13545 MiB, 15360 MiB
```

---

## 6. 常见踩坑记录


| 问题                              | 原因                                   | 解决方案                                                    |
| ------------------------------- | ------------------------------------ | ------------------------------------------------------- |
| `cannot find -lcuda`            | Kaggle 上 `libcuda.so` 路径不在链接器搜索路径    | 创建软链接到 `/usr/local/cuda-12.8/targets/x86_64-linux/lib/` |
| `enforce_eager=True` 无效         | 编译在子进程里发生，参数还没传进去                    | 加 `VLLM_COMPILE_LEVEL=0` 环境变量，在 import 前设置              |
| `server` 变量丢失                   | Kernel 重启后内存清空                       | `!pkill -f "vllm.entrypoints.openai.api_server"`        |
| `torch.memory_allocated()` 显示 0 | vLLM 绕过 PyTorch allocator 自己管显存      | 用 `nvidia-smi` 查看真实占用                                   |
| 模型答非所问                          | 裸 prompt 不适合 Instruct 模型；1.5B 能力本身有限 | 生产环境用更大模型；或用 chat 格式 prompt                             |


---

## 7. 官方文档索引


| 内容                    | 链接                                                                                                                                 |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Quickstart（两种用法）      | [https://docs.vllm.ai/en/stable/getting_started/quickstart.html](https://docs.vllm.ai/en/stable/getting_started/quickstart.html)   |
| OpenAI 兼容 Server      | [https://docs.vllm.ai/en/stable/serving/openai_compatible_server](https://docs.vllm.ai/en/stable/serving/openai_compatible_server) |
| Online Serving 代码示例   | [https://docs.vllm.ai/en/stable/examples/basic/online_serving](https://docs.vllm.ai/en/stable/examples/basic/online_serving)       |
| PagedAttention 原理     | [https://docs.vllm.ai/en/stable/design/paged_attention/](https://docs.vllm.ai/en/stable/design/paged_attention/)                   |
| 性能调优参数                | [https://docs.vllm.ai/en/stable/configuration/optimization/](https://docs.vllm.ai/en/stable/configuration/optimization/)           |
| 全部 CLI 参数（vllm serve） | [https://docs.vllm.ai/en/stable/cli/serve](https://docs.vllm.ai/en/stable/cli/serve)                                               |
| 张量并行与扩展               | [https://docs.vllm.ai/en/stable/serving/parallelism_scaling](https://docs.vllm.ai/en/stable/serving/parallelism_scaling)           |


---

## 8. AI Infra 工程师视角总结


| 概念                   | 要点                                                  |
| -------------------- | --------------------------------------------------- |
| KV Cache             | 用显存换计算，缓存每个 token 的 K/V 矩阵                          |
| PagedAttention       | OS 分页思想解决显存碎片，浪费率从 60~80% 降到 4%                     |
| Continuous Batching  | 请求随到随算，GPU 利用率最大化                                   |
| OpenAI 兼容 API        | 行业事实标准，业务代码零改动切换模型                                  |
| tensor_parallel_size | 小模型无需开，大模型（单卡装不下）必须开                                |
| enforce_eager=True   | Kaggle/受限环境禁用 JIT 编译的开关                             |
| 实际工作场景               | vLLM 跑在 K8s Pod 里，前面挂 Nginx/Service，业务只看到一个 HTTP 端口 |


