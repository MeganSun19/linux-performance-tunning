# NCCL 集合通信学习笔记

> 学习日期：2024年
> 实验环境：Kaggle Notebook (2× Tesla T4 GPU)

---

## 1. NCCL 概述

**NVIDIA Collective Communications Library** — GPU 间高性能通信库。

### 核心价值
- 自动检测硬件拓扑（NVLink、PCIe、InfiniBand）
- 自动选择最优通信路径和算法
- 对应用层透明，一套 API 适配所有硬件配置

### 在 AI 训练栈中的位置

```
PyTorch / TensorFlow
        ↓
torch.distributed / Horovod
        ↓
      NCCL  ←── 这一层
        ↓
NVLink / PCIe / InfiniBand
        ↓
      GPU
```

### 官方资源
- 文档：https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/
- nccl-tests：https://github.com/NVIDIA/nccl-tests

---

## 2. 集合操作（Collective Operations）

### 2.1 AllReduce — 最核心

**定义**：每个 GPU 贡献数据，执行规约（如求和），**所有 GPU 都得到相同结果**。

```
GPU 0: [1, 2, 3]     ┐
GPU 1: [4, 5, 6]     ├─→ sum ─→ 所有 GPU 都得到 [10, 14, 18]
GPU 2: [5, 7, 9]     ┘
```

**应用**：数据并行训练中的**梯度同步**（最频繁的操作）

### 2.2 AllGather — 收集所有数据

**定义**：每个 GPU 贡献数据，**所有 GPU 都收到完整的拼接结果**。

```
GPU 0: [A]     ┐
GPU 1: [B]     ├─→ 所有 GPU 都得到 [A, B, C]
GPU 2: [C]     ┘
```

**应用**：FSDP 中收集完整权重

### 2.3 ReduceScatter — 规约后分散

**定义**：先规约，再把结果**分片**发给各 GPU。

```
GPU 0: [A0, A1, A2]     ┐            GPU 0: [ΣX0]
GPU 1: [B0, B1, B2]     ├─→ sum ─→  GPU 1: [ΣX1]
GPU 2: [C0, C1, C2]     ┘            GPU 2: [ΣX2]
```

**应用**：ZeRO 优化器中的梯度分散

### 2.4 Broadcast — 一对多广播

**定义**：从 root GPU 把数据**复制**到所有其他 GPU。

**应用**：初始化时广播模型参数

### 2.5 Reduce — 多对一汇聚

**定义**：所有 GPU 规约，**只有 root 得到结果**。

### 2.6 AlltoAll — 全交换

**定义**：每个 GPU 向每个其他 GPU 发送不同的数据块。

**应用**：张量并行、MoE 中的 token 路由

### 操作关系

```
AllReduce = Reduce + Broadcast
AllReduce = ReduceScatter + AllGather
```

---

## 3. 通信算法

### 3.1 Ring 算法

**思想**：GPU 组成逻辑环，数据沿环单向流动。

```
    GPU0 ──→ GPU1
     ↑         ↓
    GPU3 ←── GPU2
```

**过程**（AllReduce）：
1. **Reduce-Scatter 阶段**：数据分 chunk，沿环传递并累加，n-1 步后每个 GPU 持有一个 chunk 的完整和
2. **All-Gather 阶段**：完整 chunk 沿环传播，n-1 步后所有 GPU 拥有全部结果

**特点**：
- 步数：2(n-1)
- 带宽利用率高（~100%）
- **适合大数据量**

### 3.2 Tree 算法

**思想**：GPU 组织成树形结构，数据沿树上下传播。

```
        GPU0 (root)
       /    \
    GPU1    GPU2
   /   \
 GPU3  GPU4
```

**过程**（AllReduce）：
1. **Reduce 阶段（上行）**：叶子 → 根，逐层汇聚
2. **Broadcast 阶段（下行）**：根 → 叶子，逐层广播

**特点**：
- 步数：2×log(n)
- 带宽利用率低
- **适合小数据量**（延迟主导）

### 3.3 Collnet (SHARP)

**思想**：利用**网络交换机硬件**执行规约计算。

```
传统：GPU ──数据──→ GPU ──数据──→ GPU（GPU 计算）
SHARP：GPU ─┬─数据─→ [交换机] ─结果─→ 所有 GPU
       GPU ─┘      (交换机计算)
```

**特点**：
- 延迟接近 Tree，带宽接近 Ring
- 需要特定硬件（Mellanox Quantum 交换机）

### 算法选择

| 数据大小 | 选择 | 原因 |
|---------|------|------|
| < 几 KB | Tree | 延迟主导 |
| > 几 MB | Ring | 带宽主导 |

**注意**：n=2 时 Ring 和 Tree 步数相同（都是 2 步），差异不明显。n≥8 时差异才显著。

---

## 4. 性能指标

### 4.1 两种带宽

| 指标 | 定义 | 用途 |
|------|------|------|
| **algbw** | `数据量 / 时间` | 应用层看到的带宽 |
| **busbw** | 考虑通信模式后的硬件利用率 | **与硬件峰值对比** |

### 4.2 busbw 计算公式

| 操作 | 公式 | n=2 系数 | n=8 系数 |
|------|------|---------|---------|
| AllReduce | `algbw × 2(n-1)/n` | 1.0 | 1.75 |
| AllGather | `algbw × (n-1)/n` | 0.5 | 0.875 |
| ReduceScatter | `algbw × (n-1)/n` | 0.5 | 0.875 |
| Broadcast | `algbw × 1` | 1.0 | 1.0 |

**n=2 时 AllReduce 的 busbw = algbw**（系数为 1）

### 4.3 性能评估标准

| busbw vs 硬件峰值 | 评估 |
|------------------|------|
| 90%+ | 优秀 |
| 70-90% | 良好 |
| 50-70% | 一般 |
| < 50% | 需优化 |

### 4.4 不同硬件的预期带宽

| 硬件 | 连接方式 | 预期 busbw |
|------|---------|-----------|
| T4 (Kaggle) | PCIe Gen3 | 4-12 GB/s |
| V100 (DGX-1) | NVLink 2.0 | 100-150 GB/s |
| A100 (DGX A100) | NVLink 3.0 | 400-500 GB/s |
| H100 (DGX H100) | NVLink 4.0 | 700-800 GB/s |

---

## 5. 关键环境变量

### 调试

```bash
# 查看通信路径选择
export NCCL_DEBUG=INFO

# 更详细（性能下降）
export NCCL_DEBUG=TRACE

# 指定子系统
export NCCL_DEBUG_SUBSYS=INIT,COLL,NET
```

### 算法控制

```bash
# 强制使用特定算法
export NCCL_ALGO=RING   # 或 TREE, COLLNET
```

### 网络配置

```bash
# 指定网络接口
export NCCL_SOCKET_IFNAME=eth0

# 指定 InfiniBand 设备
export NCCL_IB_HCA=mlx5_0

# 禁用 InfiniBand（调试用）
export NCCL_IB_DISABLE=1
```

---

## 6. Out-of-place vs In-place

| 模式 | 含义 | 内存 |
|------|------|------|
| Out-of-place | 输入输出用不同缓冲区 | 2× |
| In-place | 输入输出用同一缓冲区 | 1× |

**实际训练用 in-place**（省内存），看 nccl-tests 结果时**看 in-place 列**。

---

## 7. nccl-tests 实验

### 7.1 环境准备（Kaggle）

```bash
# 1. 新建 Notebook，Settings → Accelerator → GPU T4 ×2
# 2. Settings → Internet → ON（需手机验证）

# 克隆并编译
!git clone https://github.com/NVIDIA/nccl-tests.git /kaggle/working/nccl-tests
!cd /kaggle/working/nccl-tests && make -j$(nproc)

# 验证 GPU
!nvidia-smi
```

**注意**：Kaggle 切换加速器会重启 session，`/kaggle/working/` 内容会清空。

### 7.2 实验 1：基础 AllReduce 测试

**目的**：了解输出格式，观察带宽随数据量变化

**命令**：
```bash
!cd /kaggle/working/nccl-tests && ./build/all_reduce_perf -b 8 -e 128M -f 2 -g 2
```

**参数说明**：
- `-b 8`：最小数据 8 bytes
- `-e 128M`：最大数据 128 MB
- `-f 2`：每次翻倍
- `-g 2`：使用 2 个 GPU

**输出示例**：
```
#       size         count      type   redop    root     time   algbw   busbw  #wrong
           8             2     float     sum      -1    16.13    0.00    0.00       0
        1024           256     float     sum      -1    16.73    0.06    0.06       0
     1048576        262144     float     sum      -1   300.81    3.49    3.49       0
   134217728      33554432     float     sum      -1  33403.7    4.02    4.02       0
```

**解读**：
- 小数据（8B-1KB）：time ≈ 16μs（延迟恒定）
- 大数据（128MB）：busbw ≈ 4 GB/s（带宽饱和）
- n=2 时 algbw = busbw

### 7.3 实验 2：NCCL_DEBUG 查看通信路径

**目的**：了解 NCCL 检测到的拓扑和选择的路径

**命令**：
```bash
!cd /kaggle/working/nccl-tests && NCCL_DEBUG=INFO ./build/all_reduce_perf -b 1M -e 1M -g 2 2>&1 | head -50
```

**关键输出**：
```
NCCL INFO NET/IB : No device found.          ← 没有 InfiniBand
NCCL INFO Using network Socket               ← 回退到 Socket
NCCL INFO nRanks 2 nNodes 1 localRanks 2 MNNVL 0  ← 没有 NVLink
NCCL INFO Channel 00 : 0[0] -> 1[1] via SHM/direct/direct  ← 使用共享内存
NCCL INFO Connected all rings, use ring      ← 使用 Ring 算法
```

**解读**：
- Kaggle T4 没有 NVLink、没有 InfiniBand
- GPU 间通过共享内存（SHM）通信
- NCCL 自动选择 Ring 算法

### 7.4 实验 3：Ring vs Tree 算法对比

**目的**：验证算法差异，理解 n=2 时的特殊情况

**命令**：
```bash
# Ring 算法
!cd /kaggle/working/nccl-tests && NCCL_ALGO=RING ./build/all_reduce_perf -b 8 -e 64K -f 2 -g 2 -n 100

# Tree 算法
!cd /kaggle/working/nccl-tests && NCCL_ALGO=TREE ./build/all_reduce_perf -b 8 -e 64K -f 2 -g 2 -n 100
```

**输出对比（in-place time, μs）**：

| 数据大小 | Ring | Tree | 差异 |
|---------|------|------|------|
| 8 B | 14.49 | 17.00 | Ring +17% |
| 1 KB | 14.94 | 18.20 | Ring +18% |
| 4 KB | 16.77 | 23.69 | Ring +29% |
| 16 KB | 23.56 | 62.93 | Ring +63% |
| 64 KB | 45.72 | 75.48 | Ring +39% |

**Avg busbw**：Ring 0.30 GB/s vs Tree 0.16 GB/s

**结论**：
- n=2 时 **Ring 全面更快**
- 原因：n=2 时两种算法步数相同（都是 2 步），Ring 实现更优化
- **理论上的 Tree 优势（小数据延迟低）需要 n≥8 才能体现**

### 7.5 实验 4：确认算法生效（DEBUG + 算法）

**目的**：验证环境变量确实改变了算法

**命令**：
```bash
!cd /kaggle/working/nccl-tests && NCCL_DEBUG=INFO NCCL_ALGO=RING ./build/all_reduce_perf -b 1M -e 1M -g 2 2>&1 | head -60
```

**关键输出（Ring）**：
```
NCCL INFO NCCL_ALGO set by environment to RING
NCCL INFO Enabled NCCL Func/Proto/Algo Matrix:
     Function |    Tree    Ring
    AllReduce |      0       1    ← Ring=1, Tree=0
NCCL INFO Connected all rings, use ring
```

**关键输出（Tree）**：
```
NCCL INFO NCCL_ALGO set by environment to TREE
NCCL INFO Enabled NCCL Func/Proto/Algo Matrix:
     Function |    Tree    Ring
    AllReduce |      1       0    ← Tree=1, Ring=0
NCCL INFO Connected all trees
```

---

## 8. 与 PyTorch 的集成

```python
import torch.distributed as dist

# 初始化时指定 NCCL 后端
dist.init_process_group(backend="nccl")

# 以下操作都走 NCCL
dist.all_reduce(tensor)       # → ncclAllReduce
dist.all_gather(tensor_list, tensor)  # → ncclAllGather
dist.broadcast(tensor, src=0) # → ncclBroadcast
```

**生产环境用 DistributedDataParallel**（多进程 + NCCL），不用 DataParallel。

---

## 9. AI Infra 工程师需要掌握的程度

### 必须知道
- 6 种集合操作的用途
- PCIe vs NVLink 性能差距（10-100 倍）
- busbw 代表硬件利用率
- NCCL_DEBUG=INFO 排查问题

### 了解即可
- busbw 计算公式（工具自动算）
- Ring vs Tree 复杂度推导
- 具体带宽数值计算

### 实际工作场景
1. **训练慢了**：跑 nccl-tests，看 busbw 是否达到硬件预期
2. **采购硬件**：知道 NVLink vs PCIe 的性能量级
3. **看代码**：知道 all_reduce 是梯度同步，all_gather 是收集权重

---

## 10. 总结

| 概念 | 要点 |
|------|------|
| NCCL | GPU 间高性能通信库，自动选择最优路径 |
| AllReduce | 最核心操作，用于梯度同步 |
| Ring vs Tree | 大数据用 Ring，小数据用 Tree |
| busbw | 看硬件利用率，与峰值对比 |
| NVLink vs PCIe | 性能差 10-100 倍 |
| 实际工作 | 会跑 nccl-tests，会解读结果 |
