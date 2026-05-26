# DeepSpeed 分布式训练框架学习笔记

> 学习日期：2026年5月
> 实验环境：Kaggle Notebook (2× Tesla T4 GPU，每卡 15360 MiB)

---

## 1. DeepSpeed 概述

**Microsoft 开源的分布式训练框架**，核心目标是让超大模型在有限 GPU 上训练成为可能。

### 1.1 DeepSpeed ≠ ZeRO

ZeRO 是 DeepSpeed 里最出名的子模块，但 DeepSpeed 不止于此：

```
DeepSpeed
├── ZeRO（内存优化）              ← 最常用，被过度代表
├── Pipeline Parallelism          ← 流水线并行
├── Tensor Parallelism            ← 张量并行
├── Curriculum Learning           ← 训练数据课程策略
├── Sparse Attention              ← 稀疏注意力（早期研究特性）
├── Inference Engine              ← 推理优化（DeepSpeed-FastGen）
└── ZeRO-Inference                ← 推理时的内存优化
```

ZeRO 使用频率最高，是因为几乎所有人训练大模型第一个遇到的问题就是 OOM。

### 1.2 在 AI Infra 技术栈中的位置

```
应用层        微调脚本 / 预训练脚本
              │
训练接口层    HuggingFace Trainer / Accelerate
              │  deepspeed="ds_config.json"  或  fsdp="full_shard"
              │
分布式框架层  DeepSpeed ZeRO  ←二选一→  PyTorch FSDP    ← DeepSpeed 在这里
              │                                │
              └──────────────┬─────────────────┘
                             │
通信层                      NCCL（AllReduce / AllGather / ReduceScatter）
                             │
硬件层              GPU / NVLink / RoCEv2 / InfiniBand
```

DeepSpeed 做的事：在 NCCL 之上，协调多卡/多节点的**训练过程**——怎么切分权重、怎么调度梯度通信、怎么管显存。它不管推理。

### 1.3 与 vLLM 的关系

常见误解：DeepSpeed 和 vLLM 是竞争关系。实际上两者服务于完全不同的阶段：

```
[训练阶段]

  数据
   ↓
  HF Trainer（训练逻辑：epoch、eval、checkpoint、日志）
   ├─ 每个 step 调用 DeepSpeed Engine
   │     ├─ ZeRO 分片显存（优化器状态 / 梯度 / 权重）
   │     └─ NCCL 同步多卡梯度（AllReduce / ReduceScatter / AllGather）
   └─ 产出：model.safetensors

          ↓ 权重文件是两阶段唯一交接物

[推理阶段]

  model.safetensors
   ↓
  vLLM 加载权重
   ├─ PagedAttention 管理 KV Cache
   └─ Continuous Batching 处理并发请求
   ↓
  用户请求 → /v1/chat/completions → 返回结果
```

**三者分工**：

- HF Trainer = 指挥官（训练跑几轮、什么时候保存）
- DeepSpeed = 后勤（显存怎么省、多卡怎么分工）
- NCCL = 通信兵（卡和卡之间数据怎么传）

**DeepSpeed 不是在某一步，而是嵌入训练循环每个 step 内部**：

```
Step N：
  1. forward pass      → 正常跑，DeepSpeed 透明
  2. backward pass     → DeepSpeed hook 拦截，ReduceScatter 分片梯度
  3. optimizer.step()  → DeepSpeed 接管：用本卡梯度分片更新优化器状态，
                         AllGather 把更新后的权重广播给所有卡
  4. 进入 Step N+1
```

**DeepSpeed 产出权重，vLLM 消费权重**，串联关系，不是竞争。

---

## 2. ZeRO：核心内存优化

### 2.1 为什么需要 ZeRO

训练时每个参数需要存：

```
权重本身（FP16）          2 bytes
梯度（FP16）              2 bytes
优化器状态：
  FP32 主权重副本         4 bytes   ← 混合精度训练必须保留一份 FP32
  Adam momentum (FP32)    4 bytes   ← 梯度历史均值
  Adam variance (FP32)    4 bytes   ← 梯度历史方差
─────────────────────────────────
合计                     16 bytes/param
```

**FP（Floating Point）**：浮点数，后面的数字是位数。FP32 = 4 bytes，FP16/BF16 = 2 bytes。

**Adam momentum/variance**：Adam 优化器不只看当前梯度，还记录历史趋势：

- momentum（一阶矩）= 记住"过去往哪个方向走"
- variance（二阶矩）= 记住"过去走得有多剧烈"
- 对 AI Infra 工程师的意义：Adam 为每个参数额外存两个 FP32 数，共 8 bytes/param

**7B 模型训练显存估算**：

```
7,000,000,000 × 16 bytes = 112 GB（仅权重+梯度+优化器，不含激活值）
```

**为什么混用 FP16 和 FP32**：纯 FP16 训练数值不稳定（梯度太小会下溢到 0），所以用混合精度——前向/反向用 FP16，优化器状态保 FP32。

### 2.2 ZeRO 三阶段

**ZeRO = Zero Redundancy Optimizer**，逐步消除冗余：

```
                    params(2B)   grads(2B)   opt states(12B)   per-param total
─────────────────────────────────────────────────────────────────────────────
无分片（DDP）       全量          全量          全量               16 bytes
ZeRO-1             全量          全量          1/N                4 + 12/N bytes
ZeRO-2             全量          1/N           1/N                2 + 14/N bytes
ZeRO-3             1/N           1/N           1/N                16/N bytes
```

- **ZeRO-1**：只分片优化器状态（Adam momentum + variance + FP32主权重）
- **ZeRO-2**：分片优化器状态 + 梯度
- **ZeRO-3**：分片所有（权重 + 梯度 + 优化器状态），内存随卡数**线性扩展**

**通俗比喻**：

- ZeRO-1 = 分账单（只分"零花钱"）
- ZeRO-2 = 分工（再分"做的事"）
- ZeRO-3 = 分身（连"是谁"都分）

### 2.3 ZeRO vs FSDP


| 维度              | PyTorch FSDP     | DeepSpeed ZeRO    |
| --------------- | ---------------- | ----------------- |
| 出身              | PyTorch 官方内置     | Microsoft 开源，独立安装 |
| 配置方式            | Python API       | JSON 配置文件         |
| **优化器状态分片**     | ❌ 不支持            | ✅ ZeRO-1/2/3 核心特性 |
| **CPU/NVMe 卸载** | ❌                | ✅ ZeRO-Offload    |
| 安装              | 零依赖，装 PyTorch 就有 | 需额外安装，有 C++ 编译    |
| 调试难度            | 相对容易             | 较复杂，报错有时不友好       |


**工程选型**：

```
模型 < 7B，卡数 < 8           → FSDP，够用省事
模型 7B~70B，卡数 8~64        → 两者都行，DeepSpeed 更省显存
模型 > 70B，或需要 CPU 卸载   → DeepSpeed，FSDP 做不到
```

**ZeRO-2 优势量化**（7B 模型，8 卡）：

```
FSDP：优化器状态不分片 → 每卡都要存 7B × 12 bytes = 84 GB
ZeRO-2：84 GB / 8 卡 = 10.5 GB/卡  → 节省 87%
```

### 2.4 显存占用速查


| 场景              | bytes/param | 说明         |
| --------------- | ----------- | ---------- |
| 推理（FP16）        | 2           | 只有权重       |
| 训练（FP16，无优化器分片） | 16          | 权重+梯度+Adam |
| ZeRO-2，N 卡      | 2 + 14/N    | 权重全量，其余分片  |
| ZeRO-3，N 卡      | 16/N        | 全部分片       |


**口诀**：推理记 2，训练记 16；N 卡 ZeRO-3 就除以 N。

---

## 3. HuggingFace 生态关系

### 3.1 Transformers vs Trainer

```
pip install transformers  →  同一个包，两个层次

transformers（模型库）
├── AutoModel / AutoTokenizer     ← 模型定义、权重加载
├── BertModel / LlamaModel / ...  ← 各种具体模型架构
└── Trainer                       ← 训练循环封装（住在这里）
```

- **Transformers**：模型库，告诉你"Llama 3 架构长什么样"
- **Trainer**：Transformers 里的一个类，封装了完整训练循环（loss、梯度、checkpoint、日志）

### 3.2 Trainer 作为统一入口

```python
# 切换分布式策略只改一行，业务代码不动
TrainingArguments(deepspeed="ds_config.json")   # 用 DeepSpeed
TrainingArguments(fsdp="full_shard")             # 用 FSDP
TrainingArguments()                              # 单卡
```

---

## 4. 动手实验记录

### 4.1 实验环境

- **平台**：Kaggle Notebook
- **硬件**：2× Tesla T4（PCIe，每卡 15360 MiB）
- **NCCL**：2.25.1，CUDA 12.8

### 4.2 Lab 1：单卡 DeepSpeed 引擎（最终脚本）

```python
%%writefile train_lab1.py
import torch, torch.nn as nn, deepspeed, argparse

parser = argparse.ArgumentParser()
parser.add_argument('--local_rank', type=int, default=-1)
parser = deepspeed.add_config_arguments(parser)
args = parser.parse_args()

model = nn.Linear(128, 128)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4)

model_engine, optimizer, _, _ = deepspeed.initialize(
    args=args,
    model=model,
    optimizer=optimizer,
    model_parameters=model.parameters()
)

for step in range(10):
    x = torch.randn(16, 128).to(model_engine.device)
    loss = model_engine(x).mean()
    model_engine.backward(loss)
    model_engine.step()
    if model_engine.global_rank == 0:
        print(f"step {step}  loss {loss.item():.4f}")
```

```python
%%writefile ds_config.json
{
  "train_micro_batch_size_per_gpu": 16,
  "zero_optimization": { "stage": 2 }
}
```

```bash
%%bash
deepspeed --num_gpus=1 train_lab1.py --deepspeed_config ds_config.json
```

**输出**：

```
step 0  loss 0.0014
step 1  loss 0.0049
...
step 9  loss 0.0167
```

loss 在 0 附近震荡正常——无真实 label，无收敛目标，跑完不报错即成功。

### 4.3 Lab 2：双卡 ZeRO-2 显存观察（最终脚本）

脚本同 Lab 1，训练结束前加显存查询：

```python
# 训练循环末尾加（进程还活着时查）
if model_engine.global_rank == 0:
    import subprocess
    result = subprocess.run(
        ["nvidia-smi", "--query-gpu=index,memory.used,memory.total", "--format=csv"],
        capture_output=True, text=True
    )
    print(result.stdout)
```

```bash
%%bash
deepspeed --num_gpus=2 train_lab1.py --deepspeed_config ds_config.json
```

**输出**：

```
index, memory.used [MiB], memory.total [MiB]
0, 2169 MiB, 15360 MiB
1, 2169 MiB, 15360 MiB
```

两卡完全均分 ✅。2169 MiB 大部分是 DeepSpeed 框架开销（NCCL buffer、CUDA context），模型本身（nn.Linear 128×128 = 16K 参数）极小。

**与 vLLM TP=2 对比**：


| 实验               | 机制          | GPU 0     | GPU 1     | 分的是什么      |
| ---------------- | ----------- | --------- | --------- | ---------- |
| vLLM TP=2        | 张量并行（推理）    | 13545 MiB | 13545 MiB | 权重矩阵按列切分   |
| DeepSpeed ZeRO-2 | 优化器状态分片（训练） | 2169 MiB  | 2169 MiB  | 梯度+优化器状态均分 |


注意：ZeRO-2 的权重每卡还是全量存的；ZeRO-3 才连权重也分片。

### 4.4 Lab 3：HuggingFace Trainer + ZeRO-2 微调（最终脚本）

```python
%%writefile train_lab3.py
from transformers import AutoModelForSequenceClassification, AutoTokenizer, TrainingArguments, Trainer
from datasets import load_dataset

model_name = "distilbert-base-uncased"
model = AutoModelForSequenceClassification.from_pretrained(model_name, num_labels=2)
tokenizer = AutoTokenizer.from_pretrained(model_name)

dataset = load_dataset("imdb")
def tokenize(ex):
    return tokenizer(ex["text"], truncation=True, padding="max_length", max_length=128)
dataset = dataset.map(tokenize, batched=True)

training_args = TrainingArguments(
    output_dir="./output",
    num_train_epochs=2,
    per_device_train_batch_size=8,
    deepspeed="ds_z2.json",
    logging_steps=10,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=dataset["train"].select(range(400)),
)
trainer.train()
```

```python
%%writefile ds_z2.json
{
  "train_micro_batch_size_per_gpu": "auto",
  "gradient_accumulation_steps": "auto",
  "zero_optimization": { "stage": 2 },
  "optimizer": {
    "type": "AdamW",
    "params": { "lr": "auto", "weight_decay": "auto" }
  }
}
```

```bash
%%bash
deepspeed --num_gpus=2 train_lab3.py --deepspeed_config ds_z2.json
```

**输出**：

```
{'loss': '0.6605', 'grad_norm': '2.325', 'epoch': '0.4'}
{'loss': '0.05798', 'grad_norm': '0.5922', 'epoch': '0.8'}
{'loss': '0.02041', 'grad_norm': '0.2821', 'epoch': '1.2'}
{'loss': '0.01391', 'grad_norm': '0.2525', 'epoch': '1.6'}
{'loss': '0.01108', 'grad_norm': '0.2222', 'epoch': '2.0'}
{'train_runtime': '12.62', 'train_samples_per_second': '63.4', 'train_loss': '0.1528'}
```

loss 从 0.66 降到 0.01，真实收敛（有 IMDB 情感分类 label）。

**LOAD REPORT 说明**（不是报错）：

- `UNEXPECTED`：预训练权重里有 MLM 相关层（vocab_transform 等），分类任务不需要，自动丢弃
- `MISSING`：分类头（classifier、pre_classifier）是新加的，随机初始化，训练就是为了学这些

**Writing model shards**：训练结束时 DeepSpeed 把分布在两张卡的参数重新合并，保存完整权重文件。

---

## 5. 关键 API 参数说明

### 5.1 `deepspeed.initialize()`

**官方文档**：[https://deepspeed.readthedocs.io/en/latest/initialize.html](https://deepspeed.readthedocs.io/en/latest/initialize.html)


| 参数                 | 必须/可选               | 说明                                  |
| ------------------ | ------------------- | ----------------------------------- |
| `model`            | ✅ 必须                | PyTorch 模型                          |
| `model_parameters` | ✅ 必须（有 optimizer 时） | 传哪些参数给优化器                           |
| `optimizer`        | ZeRO-2/3 必须         | 不传则用 DummyOptim，ZeRO 无法 hook 会报错    |
| `args`             | 推荐                  | 包含 `--deepspeed_config` 路径的命令行 args |
| `lr_scheduler`     | 可选                  | 不传则从 config JSON 自动创建               |
| `config`           | 可选                  | 直接传 dict，替代 JSON 文件                 |


**ZeRO-2 为什么必须传 optimizer**：

```
ZeRO 需要 hook 进 optimizer.step()
才能在 step 时做分片的 AllGather / ReduceScatter
不传 → DummyOptim → ZeRO 无法 hook → AssertionError
```

### 5.2 `ds_config.json` 核心字段

**官方文档（最全）**：[https://www.deepspeed.ai/docs/config-json/](https://www.deepspeed.ai/docs/config-json/)

```json
{
  // 批处理（必须三选一或全用 auto）
  "train_micro_batch_size_per_gpu": 16,
  "gradient_accumulation_steps": 1,
  "train_batch_size": 32,              // = micro_batch × grad_accum × GPU数

  // ZeRO（核心）
  "zero_optimization": {
    "stage": 2,                        // 0/1/2/3
    "overlap_comm": true,              // 通信和计算重叠，提速
    "contiguous_gradients": true,      // 梯度内存连续，提速
    "reduce_bucket_size": 5e8,         // 梯度聚合桶大小
    "allgather_bucket_size": 5e8,
    "offload_optimizer": {             // ZeRO-Offload：优化器卸载到 CPU
      "device": "cpu",
      "pin_memory": true
    },
    "offload_param": {                 // ZeRO-3 专用：权重卸载到 CPU
      "device": "cpu"
    }
  },

  // 精度
  "fp16": { "enabled": true },         // T4 用这个
  "bf16": { "enabled": true },         // A100/H100 用这个，更稳定

  // 优化器（或用 "auto" 让 HF Trainer 填充）
  "optimizer": {
    "type": "AdamW",
    "params": { "lr": "auto", "weight_decay": "auto" }
  },

  // 其他
  "gradient_clipping": 1.0,            // 梯度裁剪
  "steps_per_print": 10
}
```

### 5.3 `TrainingArguments` 常用字段

**官方文档**：[https://huggingface.co/docs/transformers/main_classes/trainer#transformers.TrainingArguments](https://huggingface.co/docs/transformers/main_classes/trainer#transformers.TrainingArguments)


| 参数                            | 必须/可选      | 说明                         |
| ----------------------------- | ---------- | -------------------------- |
| `output_dir`                  | ✅ 必须       | checkpoint 保存路径            |
| `deepspeed`                   | 可选         | 指向 ds_config.json          |
| `num_train_epochs`            | 可选，默认 3    | 训练轮数                       |
| `per_device_train_batch_size` | 可选，默认 8    | 每卡 batch size              |
| `gradient_accumulation_steps` | 可选，默认 1    | 等效扩大 batch size            |
| `max_steps`                   | 可选         | 设了则覆盖 epochs               |
| `fp16` / `bf16`               | 可选         | 混合精度                       |
| `learning_rate`               | 可选，默认 5e-5 | 学习率                        |
| `warmup_steps`                | 可选         | 学习率 warmup                 |
| `save_strategy`               | 可选         | steps / epoch / no         |
| `logging_steps`               | 可选，默认 500  | 多少步打印一次                    |
| `report_to`                   | 可选         | none / tensorboard / wandb |


`**"auto"` 字段的意义**：ds_config.json 里写 `"auto"` 的字段，DeepSpeed 会从 `TrainingArguments` 自动读取，避免两处重复配置、数值不一致。

---

## 6. 踩坑记录


| 坑                                          | 原因                               | 解法                                         |
| ------------------------------------------ | -------------------------------- | ------------------------------------------ |
| `unrecognized arguments: ds_config.json`   | `--deepspeed` 是 flag，路径要单独传      | 改用 `--deepspeed_config ds_config.json`     |
| `zero stage 2 requires an optimizer`       | optimizer 定义了但没传进 `initialize()` | 加 `optimizer=optimizer` 参数                 |
| `push_range() unexpected keyword argument` | nvtx 版本与 DeepSpeed 不兼容           | `pip install nvtx --upgrade`               |
| `mat1 and mat2 must have the same dtype`   | 开了 fp16 后模型是 Half，输入还是 Float     | config 去掉 fp16，或输入加 `.half()`              |
| `nvidia-smi` 显示 0 MiB                      | 训练结束后进程退出，显存已释放                  | 在脚本内、进程退出前用 subprocess 查                   |
| Kaggle `pip install` 偶发失败                  | 启用 GPU 后网络偶发隔离                   | 禁用再重新启用 Internet，或加 `--no-build-isolation` |


---

## 7. 官方文档索引


| 内容                           | 链接                                                                                                                             |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| DeepSpeed 官网                 | [https://www.deepspeed.ai/](https://www.deepspeed.ai/)                                                                         |
| Getting Started              | [https://www.deepspeed.ai/getting-started/](https://www.deepspeed.ai/getting-started/)                                         |
| ZeRO Tutorial                | [https://www.deepspeed.ai/tutorials/zero/](https://www.deepspeed.ai/tutorials/zero/)                                           |
| **Config JSON（最重要）**         | [https://www.deepspeed.ai/docs/config-json/](https://www.deepspeed.ai/docs/config-json/)                                       |
| `deepspeed.initialize()` API | [https://deepspeed.readthedocs.io/en/latest/initialize.html](https://deepspeed.readthedocs.io/en/latest/initialize.html)       |
| HF Transformers DeepSpeed 集成 | [https://huggingface.co/docs/transformers/deepspeed](https://huggingface.co/docs/transformers/deepspeed)                       |
| HF TrainingArguments         | [https://huggingface.co/docs/transformers/main_classes/trainer](https://huggingface.co/docs/transformers/main_classes/trainer) |
| GitHub 官方示例                  | [https://github.com/microsoft/DeepSpeed/tree/master/tests/unit](https://github.com/microsoft/DeepSpeed/tree/master/tests/unit) |


---

## 8. AI Infra 工程师视角总结


| 概念             | 要点                                 |
| -------------- | ---------------------------------- |
| DeepSpeed 定位   | 分布式训练框架，坐在 NCCL 上、HF Trainer 下     |
| ZeRO vs FSDP   | 同层竞品，DS 能分片优化器状态+CPU卸载，FSDP 更稳定零依赖 |
| 16 bytes/param | 训练显存估算基准，AI Infra 必会               |
| `"auto"` 字段    | ds_config 里的关键设计，与 Trainer 自动联动    |
| config-json    | AI Infra 最需要熟悉的一份文档，所有性能调优都在这里     |
| 实际工作场景         | DS 训练产出权重 → 上传模型库 → vLLM 加载服务，串联关系 |


