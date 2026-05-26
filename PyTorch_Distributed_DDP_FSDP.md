# PyTorch 分布式训练：DDP 与 FSDP 学习笔记

> 学习日期：2026年5月
> 实验环境：Kaggle Notebook (2× Tesla T4 GPU)

---

## 1. 概念层次架构

在 AI Infra 中，DDP/FSDP 和 NCCL 不是同一层的东西。先搞清楚它们的位置：

```
并行策略层：DDP / FSDP / 张量并行 / 流水线并行
                ↓
分布式框架层：torch.distributed
                ↓
集合通信层：AllReduce / AllGather / ReduceScatter
                ↓
通信库层：NCCL / Gloo / MPI
                ↓
硬件互联层：NVLink / PCIe / InfiniBand
```

**DDP 和 FSDP 是"并行策略"**，它们调用 NCCL 提供的集合操作来实现功能。

---

## 2. 集合通信操作

以下操作对应 NCCL 底层，DDP/FSDP 的所有通信都建立在这几个原语之上。

### 2.1 AllReduce —— "同步全局梯度"

所有 GPU 交出自己的数据，混合求和，**所有人拿回相同的结果**。

```
GPU 0: [1,2,3]  ─┐
GPU 1: [4,5,6]  ─┼─→ sum → 所有 GPU 都得到 [5,7,9]
GPU 2: [0,0,0]  ─┘
```

**用于**：DDP 反向传播后同步梯度

### 2.2 AllGather —— "凑齐完整权重"

每个 GPU 拿出自己的一块，**所有人拿回拼好的完整数据**。

```
GPU 0: [A]  ─┐
GPU 1: [B]  ─┼─→ 所有 GPU 都得到 [A, B, C]
GPU 2: [C]  ─┘
```

**用于**：FSDP 前向/反向前把分片权重临时拼成完整权重

### 2.3 ReduceScatter —— "算完梯度，各回各家"

先全局求和，然后把结果切块，**每个 GPU 只拿回属于自己的那一份**。

```
GPU 0: [A0,A1,A2]  ─┐               GPU 0: [ΣX0]
GPU 1: [B0,B1,B2]  ─┼─→ sum ─→ 切  GPU 1: [ΣX1]
GPU 2: [C0,C1,C2]  ─┘               GPU 2: [ΣX2]
```

**用于**：FSDP 反向传播后，把梯度分散回各个持有者

### 2.4 关键等式

```
AllReduce = ReduceScatter + AllGather
```

这条等式说明：DDP 用的 AllReduce，本质上是 FSDP 所用两个操作的合并。
FSDP 只做第一步（ReduceScatter），**省掉了 AllGather 这一步的网络传输**。

---

## 3. DDP（Distributed Data Parallel）

### 3.1 核心机制

每张卡保存**完整模型**。通信只发生在"算完梯度之后"，进行一次 AllReduce。

```
训练一个 Step 的时间线：

[前向传播] → [反向传播] → [AllReduce 同步梯度] → [优化器更新]
                                  ↑
                           唯一的通信点
```

### 3.2 显存构成（"老三样"）

DDP 的每张卡必须存下：

- **权重 (Parameters)**：1× 模型大小
- **梯度 (Gradients)**：1× 模型大小
- **优化器状态 (Optimizer States, Adam)**：2× 模型大小（动量 + 方差）

合计：约 **4× 模型参数量**的显存被锁死，这就是 DDP 的根本瓶颈。

### 3.3 梯度桶重叠（Gradient Bucket Overlap）—— DDP 的核心优化

DDP 的精华不只是"AllReduce"，而是它的**通信与计算重叠机制**：

- 反向传播是从最后一层逐层往回算的
- 每当某一层的梯度算完，就立刻放入一个 Bucket（桶）
- Bucket 满了就触发该 Bucket 的 AllReduce，**不等所有梯度都算完**
- 这样，当前一层的梯度在做 AllReduce 传输时，下一层的梯度还在 GPU 上继续计算

结果：**AllReduce 通信和反向传播计算高度并行，通信开销几乎被隐藏在计算中**。

```
无重叠（朴素版）：
─[反向传播全部]──────────────────[AllReduce]──────→

有重叠（DDP 实际）：
─[反向层1]─[AllReduce层1]─[反向层2]─[AllReduce层2]─→
            ↑ 同时发生        ↑ 同时发生
```

### 3.4 优化器状态理解

**优化器**：负责"根据梯度来更新参数"的模块。不同优化器维护不同状态：


| 优化器  | 维护状态    | 显存倍率 |
| ---- | ------- | ---- |
| SGD  | 无（或仅动量） | 0~1× |
| Adam | 动量 + 方差 | 2×   |


**为什么 DDP 里优化器各自独立，但结果一致？**
因为所有 GPU 的梯度经过 AllReduce 后完全相同，用相同梯度更新相同参数，得到的结果也一定相同。

---

## 4. FSDP（Fully Sharded Data Parallel）

### 4.1 核心思想：彻底切碎"老三样"

FSDP 对应的理论是 ZeRO-3。它把 DDP 里每张卡都要完整保存的三件东西，全部按 GPU 数量 N 等分：


| 数据    | DDP（每卡） | FSDP（每卡） |
| ----- | ------- | -------- |
| 权重    | 完整 1×   | 1/N ×    |
| 梯度    | 完整 1×   | 1/N ×    |
| 优化器状态 | 完整 2×   | 1/N × 2  |


理想情况下，N 张卡的 FSDP 使单卡显存降至 DDP 的 1/N。

### 4.2 FSDP 训练生命周期（一个 Step）

**① 待机状态（平时）**
每张卡只存 1/N 的权重碎片，极度省显存。

**② 前向传播前：AllGather 借参数**

```
[AllGather 借来所有分片] → [临时拼出完整权重] → [前向计算] → [立刻释放借来的权重]
```

显存在此时短暂飙升（因为临时持有完整权重），计算完即释放。

**③ 反向传播：再次 AllGather + ReduceScatter 还梯度**

```
[AllGather 借参数] → [反向计算完整梯度] → [ReduceScatter 散发梯度] → [每卡只留 1/N 梯度]
```

**④ 优化器更新**

```
[用 1/N 梯度] → [更新 1/N 权重] → [产生 1/N 优化器状态]
```

全程不需要通信，各卡独立完成。

**⑤ 回到待机状态**
每卡只剩 1/N 权重碎片，显存恢复到最低水位。

### 4.3 FSDP 为什么用 ReduceScatter 而不用 AllReduce？

**问**：既然要同步梯度，直接 AllReduce 让大家都拿到全量梯度不是更方便？

**答**：因为 FSDP 每张卡只更新 1/N 的权重，完整的梯度给它不仅用不上，还会撑爆显存。
ReduceScatter 做了两件事：

1. 全局求和（全部梯度汇聚）
2. 立刻切块（每张卡只拿走属于自己负责的那 1/N）

省了一半网络传输，也解决了显存爆炸的问题。这就是 AllReduce = ReduceScatter + AllGather 等式的实际价值：FSDP 砍掉了 AllGather 这一步。

---

## 5. DDP vs FSDP 对比总览


| 特性        | DDP               | FSDP                                       |
| --------- | ----------------- | ------------------------------------------ |
| 每卡存储      | 完整模型              | 1/N 模型                                     |
| 通信次数/Step | 1 次               | 多次（前向+反向各一次 AllGather，反向后一次 ReduceScatter） |
| 主要通信操作    | AllReduce         | AllGather + ReduceScatter                  |
| 单卡显存      | 4× 参数量（老三样）       | 约 4×/N 参数量                                 |
| 适用场景      | 单卡装得下的模型          | 单卡装不下，但集群总显存够的超大模型                         |
| 通信开销      | 低（只有一次 AllReduce） | 较高（频繁 AllGather）                           |


---

## 6. 分布式训练脚本的"八股文"结构

PyTorch 分布式训练必须写成独立的 `.py` 文件，用 `torchrun` 启动，原因是：

**分布式训练本质是多进程编程**。`torchrun --nproc_per_node=2` 会拉起 2 个独立 Python 进程，并给每个进程注入 `LOCAL_RANK` 环境变量（进程 A 得到 `"0"`，进程 B 得到 `"1"`）。

标准脚本的 **5 个固定步骤**：

```python
# 步骤 1：建联认亲
def setup():
    dist.init_process_group("nccl")              # 建立 NCCL 通信组
    local_rank = int(os.environ["LOCAL_RANK"])    # 读取 torchrun 注入的进程编号
    torch.cuda.set_device(local_rank)            # 绑定到对应 GPU，禁止串台
    return local_rank

# 步骤 2：模型上卡
model = nn.Linear(10, 10).to(local_rank)

# 步骤 3：穿上并行机甲
model = DDP(model, device_ids=[local_rank])      # 或 FSDP(model)

# 步骤 4：标准训练连招
optimizer.zero_grad()             # 擦黑板（清空旧梯度，否则累加出错）
output = model(input)             # 做题（前向传播）
loss = criterion(output, target)  # 对答案（算误差）
loss.backward()                   # 找原因（反向传播，DDP 这里触发 AllReduce）
optimizer.step()                  # 订正（更新权重）

# 步骤 5：只让 rank 0 打印/保存（防止重复 N 次）
if local_rank == 0:
    print("Done!")

# 步骤 6：和平分手
dist.destroy_process_group()
```

---

## 7. 动手实验：DDP vs FSDP 显存对比

### 7.1 实验环境

- **平台**：Kaggle Notebook
- **硬件**：GPU T4 ×2（双卡，PCIe 连接，无 NVLink）
- **模型**：10 层 4096×4096 线性层（参数量约 1.68 亿，方便凸显差距）
- **目标**：实验验证 FSDP 的显存切分效果

### 7.2 实验脚本 (`compare_minimal.py`)

```python
import os
import argparse
import torch
import torch.nn as nn
import torch.optim as optim
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP

def setup():
    dist.init_process_group("nccl")
    local_rank = int(os.environ["LOCAL_RANK"])
    torch.cuda.set_device(local_rank)
    return local_rank

def cleanup():
    dist.destroy_process_group()

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--mode', type=str, choices=['ddp', 'fsdp'], default='ddp')
    args = parser.parse_args()

    local_rank = setup()

    # 10 层 4096x4096 线性层
    model = nn.Sequential(*[nn.Linear(4096, 4096) for _ in range(10)]).to(local_rank)

    if args.mode == 'ddp':
        model = DDP(model, device_ids=[local_rank])
    else:
        model = FSDP(model)

    optimizer = optim.Adam(model.parameters(), lr=1e-3)
    criterion = nn.MSELoss()
    dummy_input = torch.randn(32, 4096).to(local_rank)
    dummy_target = torch.randn(32, 4096).to(local_rank)

    torch.cuda.reset_peak_memory_stats(local_rank)

    optimizer.zero_grad()
    output = model(dummy_input)
    loss = criterion(output, dummy_target)
    loss.backward()
    optimizer.step()

    peak_mem = torch.cuda.max_memory_allocated(local_rank) / (1024 * 1024)

    if local_rank == 0:
        print(f"训练模式: {args.mode.upper()}")
        print(f"GPU 0 峰值显存占用: {peak_mem:.2f} MB")

    cleanup()

if __name__ == "__main__":
    main()
```

### 7.3 运行命令

```bash
# DDP 模式
!torchrun --standalone --nproc_per_node=2 compare_minimal.py --mode ddp

# FSDP 模式
!torchrun --standalone --nproc_per_node=2 compare_minimal.py --mode fsdp
```

### 7.4 实验输出

**DDP 模式：**

```
训练模式: DDP
GPU 0 峰值显存占用: 3860.19 MB
```

**FSDP 模式：**

```
训练模式: FSDP
GPU 0 峰值显存占用: 2259.80 MB
```

### 7.5 结果分析


| 模式   | 峰值显存    | 说明                             |
| ---- | ------- | ------------------------------ |
| DDP  | 3860 MB | 完整权重 + 完整梯度 + 完整 Adam 状态 + 激活值 |
| FSDP | 2260 MB | 权重/梯度/优化器各切 1/2，节省约 1.6 GB     |


**节省了哪些显存？**
模型参数量 ≈ 1.68 亿，FP32 下：

- 参数：1.68亿 × 4B ≈ 672 MB
- 梯度：672 MB
- Adam 状态：672 × 2 = 1344 MB
- 老三样合计：约 2688 MB

FSDP 将这 2688 MB 切成了 2 份，每张卡只负担 1344 MB。其余差值来自激活值（不切分）和 PyTorch 显存碎片。

**关键结论**：2 张卡节省约 1/2 的老三样。若是 8 张卡则节省 7/8；若是 1000 张卡，老三样显存几乎可以忽略不计。这就是 FSDP/ZeRO-3 能训练千亿参数大模型的底层基石。

---

## 8. 常见运行时警告解释

### `OMP_NUM_THREADS` 警告

```
Setting OMP_NUM_THREADS environment variable for each process to be 1 in default...
```

- **含义**：`torchrun` 默认把每个进程能用的 CPU 线程数限制为 1，防止多进程抢 CPU 导致系统过载。
- **玩具实验**：忽略即可。
- **生产环境调优**：
  ```bash
  export OMP_NUM_THREADS=$(( 机器CPU核数 / GPU数量 ))
  # 例：64核 CPU + 8卡 GPU → OMP_NUM_THREADS=8
  ```

### `socket.cpp: hostname cannot be retrieved` 警告

```
[c10d] The hostname of the client socket cannot be retrieved. err=-3
```

- **含义**：Kaggle 容器网络配置问题，无法解析本机 hostname。
- **影响**：完全不影响训练，NCCL 会自动回退到通过 IP 直连。忽略即可。

---

## 9. Key take aways

### Should know

- DDP 和 FSDP 是并行策略，NCCL 是底层通信库，两者不是同一层
- DDP 底层走 AllReduce，FSDP 底层走 AllGather + ReduceScatter
- FSDP 的四个生命周期状态（待机 → AllGather → 计算 → ReduceScatter）
- `torchrun` 是多进程启动器，不是 Python 内置，`LOCAL_RANK` 是它注入的
- 只有 `rank 0` 负责打印日志和保存模型（行规）

### better to know

- 梯度桶的具体 Bucket 大小调优（默认 25MB，生产环境会调整）
- FSDP 的 `ShardingStrategy` 参数（FULL_SHARD、SHARD_GRAD_OP 等）
- ZeRO-1/2/3 精确显存公式推导

### 实际工作场景

1. **模型太大单卡装不下**：从 DDP 切换到 FSDP，调整 `ShardingStrategy`
2. **FSDP 训练慢**：检查跨节点通信，确认 TP 是否跑在 NVLink 上而非跨机
3. **调试分布式训练**：用 `NCCL_DEBUG=INFO` + `rank 0` 日志定位问题
4. **性能 Benchmark**：用 `torch.cuda.max_memory_allocated` 统计峰值显存

---

## 10. 总结


| 概念                                    | 要点                                          |
| ------------------------------------- | ------------------------------------------- |
| DDP                                   | 完整模型复制，AllReduce 同步梯度，通信次数少                 |
| FSDP                                  | 老三样全切 1/N，AllGather 借参数 + ReduceScatter 还梯度 |
| AllReduce = ReduceScatter + AllGather | FSDP 只做前半步，省掉后半步通信                          |
| 梯度桶重叠                                 | DDP 的核心优化，通信与计算并行，降低实际等待时间                  |
| torchrun                              | 多进程启动器，自动注入 LOCAL_RANK 等环境变量                |
| OMP_NUM_THREADS                       | 生产环境必须调优，值 = CPU核数 / GPU数                   |
| 适用场景                                  | DDP 适合中小模型，FSDP 适合单卡塞不下的超大模型                |


