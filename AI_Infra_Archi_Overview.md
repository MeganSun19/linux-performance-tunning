# AI 基础设施架构全景

> **创建日期**: 2026-04-23  
> **学习背景**: AI Infra 学习路径，从 GPU 监控到完整架构理解  
> **持续更新**: 随学习进度补充

---

## 一、NVIDIA 完整软件栈

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              应用层 (Applications)                               │
│  PyTorch | TensorFlow | JAX | vLLM | TensorRT-LLM | Triton Server | NeMo       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           AI 框架与工具 (AI Frameworks)                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │   cuDNN     │ │   cuBLAS    │ │  TensorRT   │ │    DALI     │              │
│  │ (神经网络) │ │ (线性代数) │ │ (推理优化) │ │ (数据加载) │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐              │
│  │   RAPIDS    │ │   Thrust    │ │   cuFFT     │ │  cuSPARSE   │              │
│  │ (数据科学) │ │ (并行算法) │ │  (FFT)     │ │ (稀疏矩阵) │              │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘              │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            通信层 (Communication)                                │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │                              NCCL                                         │ │
│  │              (多GPU/多节点集合通信: AllReduce, AllGather等)               │ │
│  │                    自动选择最优路径: NVLink > PCIe > Network              │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐                  │
│  │    NVSHMEM      │ │     SHARP       │ │    MPI/GLOO     │                  │
│  │  (GPU共享内存)  │ │ (网内聚合计算)  │ │  (分布式通信)   │                  │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        数据加速层 (Magnum IO)                                    │
│  ┌─────────────────────────┐ ┌─────────────────────────┐                       │
│  │    GPUDirect RDMA       │ │   GPUDirect Storage     │                       │
│  │  (GPU↔网卡零拷贝)       │ │   (GPU↔存储零拷贝)      │                       │
│  └─────────────────────────┘ └─────────────────────────┘                       │
│  ┌─────────────────────────┐ ┌─────────────────────────┐                       │
│  │    GPUDirect P2P        │ │     GPUDirect Async     │                       │
│  │   (GPU↔GPU直连)         │ │    (异步数据传输)       │                       │
│  └─────────────────────────┘ └─────────────────────────┘                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              CUDA 层 (CUDA Stack)                                │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │                         CUDA Runtime API                                  │ │
│  │         cudaMalloc / cudaMemcpy / cudaLaunchKernel / cudaStream          │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │                         CUDA Driver API                                   │ │
│  │                    cuCtxCreate / cuModuleLoad / cuLaunchKernel           │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      驱动与管理层 (Driver & Management)                          │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        NVIDIA Driver (nvidia.ko)                         │   │
│  │                      GPU 硬件抽象、内存管理、调度                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌─────────── 管理 API ──────────┐  ┌─────────── 诊断工具 ──────────┐         │
│  │                                │  │                                │         │
│  │  ┌──────────┐ ┌──────────┐   │  │  ┌──────────┐ ┌──────────┐   │         │
│  │  │   NVML   │ │   DCGM   │   │  │  │   NVVS   │ │nvidia-smi│   │         │
│  │  │ (管理库) │ │(数据中心 │   │  │  │(验证套件)│ │ (CLI)    │   │         │
│  │  │          │ │ GPU管理) │   │  │  │          │ │          │   │         │
│  │  └──────────┘ └──────────┘   │  │  └──────────┘ └──────────┘   │         │
│  │                                │  │                                │         │
│  │  ┌──────────┐ ┌──────────┐   │  │  ┌──────────┐ ┌──────────┐   │         │
│  │  │dcgm-     │ │  DCGM    │   │  │  │ Nsight   │ │ cuda-gdb │   │         │
│  │  │exporter  │ │  SDK     │   │  │  │ Systems  │ │          │   │         │
│  │  │(Prometheus)│          │   │  │  │(性能分析)│ │ (调试)   │   │         │
│  │  └──────────┘ └──────────┘   │  │  └──────────┘ └──────────┘   │         │
│  └────────────────────────────────┘  └────────────────────────────────┘         │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    NVSwitch/NVLink 管理层 (Interconnect Management)              │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                        Fabric Manager (FM)                               │   │
│  │              NVSwitch 配置、Fabric 初始化、NVLink 监控                   │   │
│  │                    (独立守护进程，DGX/HGX 必需)                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │    NVLSM     │  │    NSCQ      │  │    NVSDM     │  │     NMX      │        │
│  │   (NVLink    │  │  (NVSwitch   │  │  (NVSwitch   │  │  (多节点     │        │
│  │  Subnet Mgr) │  │ Config/Query)│  │  Data Mgr)   │  │  NVLink管理) │        │
│  │              │  │              │  │              │  │              │        │
│  │ • 拓扑发现   │  │ • 配置管理   │  │ • 数据路径   │  │ • 跨节点     │        │
│  │ • LID分配    │  │ • 状态查询   │  │ • 流量监控   │  │   Fabric     │        │
│  │ • 路由计算   │  │ • 固件管理   │  │ • 性能统计   │  │ • NMX-M/C    │        │
│  │              │  │              │  │              │  │              │        │
│  │ Hopper及以前 │  │ Hopper及以前 │  │ Blackwell+   │  │ GB200+       │        │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         网络驱动层 (Network Drivers)                             │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    MLNX_OFED / DOCA-OFED                                 │   │
│  │           InfiniBand 和 Ethernet RDMA 驱动 (Mellanox/NVIDIA)            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │     UFM      │  │    NetQ      │  │    DOCA      │  │   nvidia-fs  │        │
│  │ (InfiniBand  │  │  (Ethernet   │  │ (BlueField   │  │  (GDS驱动)   │        │
│  │  网络管理)   │  │   可观测性)  │  │  DPU SDK)    │  │              │        │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              硬件层 (Hardware)                                   │
│                                                                                  │
│  ┌─── GPU ───┐  ┌─── 互连 ───┐  ┌─── 网络 ───┐  ┌─── DPU ───┐  ┌─ 存储 ─┐    │
│  │           │  │            │  │            │  │           │  │        │    │
│  │ H100/H200 │  │  NVLink 4  │  │ Quantum    │  │ BlueField │  │ NVMe   │    │
│  │ Blackwell │  │  NVSwitch  │  │ InfiniBand │  │   DPU     │  │ NVMe-oF│    │
│  │ (GPU Die) │  │  (900GB/s) │  │ Spectrum   │  │ (网络     │  │ 并行FS │    │
│  │           │  │            │  │ Ethernet   │  │  加速)    │  │        │    │
│  │ • SM      │  │ • 单机内   │  │            │  │           │  │        │    │
│  │ • Tensor  │  │   GPU互连  │  │ • 跨节点   │  │ • RDMA    │  │ • GDS  │    │
│  │   Core    │  │ • 统一内存 │  │   通信     │  │ • 卸载    │  │   直连 │    │
│  │ • HBM     │  │   Fabric   │  │ • RoCE     │  │ • 加密    │  │        │    │
│  │           │  │            │  │ • IB       │  │           │  │        │    │
│  └───────────┘  └────────────┘  └────────────┘  └───────────┘  └────────┘    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、NVIDIA 组件速查表

| 缩写 | 全称 | 层级 | 职责 |
|------|------|------|------|
| **NVML** | NVIDIA Management Library | 驱动层 | GPU 管理 API（nvidia-smi 底层） |
| **DCGM** | Data Center GPU Manager | 管理层 | 数据中心级 GPU 监控、诊断、健康检查 |
| **NVVS** | NVIDIA Validation Suite | 诊断层 | GPU 硬件验证测试套件 |
| **FM** | Fabric Manager | NVSwitch层 | NVSwitch 配置和 Fabric 初始化 |
| **NVLSM** | NVLink Subnet Manager | NVSwitch层 | NVLink 拓扑发现、路由计算 |
| **NSCQ** | NVSwitch Config & Query | NVSwitch层 | NVSwitch 配置查询（Hopper 及以前） |
| **NVSDM** | NVSwitch Data Manager | NVSwitch层 | NVSwitch 数据路径管理（Blackwell+） |
| **NMX** | NVLink Management Software | NVSwitch层 | 多节点 NVLink 管理（GB200+） |
| **NCCL** | NVIDIA Collective Communications Library | 通信层 | 多 GPU 集合通信 |
| **UFM** | Unified Fabric Manager | 网络层 | InfiniBand 网络管理 |
| **GDS** | GPUDirect Storage | I/O层 | GPU ↔ 存储零拷贝 |
| **DOCA** | Data Center Infrastructure on a Chip Architecture | DPU层 | BlueField DPU 开发框架 |

---

## 三、业界通用 AI Infra 架构（厂商无关）

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              应用层 (Applications)                               │
│                    PyTorch / TensorFlow / JAX / Hugging Face                    │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         编译优化层 (Compiler & Optimizer)                        │
│         XLA | TorchCompile | ONNX Runtime | TVM | Triton | MLIR                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          加速库层 (Accelerated Libraries)                        │
│              BLAS | DNN | FFT | Sparse | Collective Communication              │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         编程模型层 (Programming Model)                           │
│           CUDA | HIP | SYCL | OpenCL | oneAPI | Triton Language                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           运行时层 (Runtime)                                     │
│      CUDA Runtime | ROCm | Level Zero | Neuron SDK | XLA Runtime               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          驱动层 (Driver)                                         │
│         GPU Driver | RDMA Driver | Storage Driver | DPU Driver                 │
└─────────────────────────────────────────────────────────────────────────────────┘
                                        │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          硬件层 (Hardware)                                       │
│                  GPU | TPU | NPU | ASIC | DPU | SmartNIC                        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 四、各层 NVIDIA vs 替代方案对比

### 4.1 硬件层

| 维度 | NVIDIA | AMD | Intel | Google | AWS |
|------|--------|-----|-------|--------|-----|
| **产品** | H100/H200/Blackwell | MI300X | Gaudi 3 | TPU v5/v6 | Trainium/Inferentia |
| **显存** | 80-192GB HBM3 | 192GB HBM3 | 128GB HBM2e | 专有 | 专有 |
| **互连** | NVLink 900GB/s | Infinity Fabric | 以太网 | 专有 | 专有 |
| **生态** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **成本** | 💰💰💰 | 💰💰 | 💰💰 | 💰💰💰 | 💰 |

### 4.2 编程模型层

| 模型 | 厂商 | 跨平台 | 生态成熟度 |
|------|------|--------|-----------|
| **CUDA** | NVIDIA | ❌ 仅 NVIDIA | ⭐⭐⭐⭐⭐ |
| **HIP** | AMD | ✅ AMD + NVIDIA | ⭐⭐⭐⭐ |
| **SYCL** | Khronos | ✅ 全平台 | ⭐⭐⭐ |
| **oneAPI** | Intel | ✅ 全平台 | ⭐⭐⭐ |
| **Triton** | OpenAI | ✅ NVIDIA + AMD | ⭐⭐⭐⭐ |

### 4.3 通信库层

| 库 | NVIDIA | AMD | Intel | 开源 |
|----|--------|-----|-------|------|
| **集合通信** | NCCL | RCCL | oneCCL | Gloo |
| **RDMA** | GPUDirect | ROCm RDMA | oneAPI L0 | 原生 verbs |
| **MPI** | NCCL+MPI | RCCL+MPI | Intel MPI | OpenMPI |

### 4.4 监控管理层

| 功能 | NVIDIA | AMD | Intel | 通用 |
|------|--------|-----|-------|------|
| **GPU 管理** | NVML/DCGM | ROCm SMI | oneAPI L0 | - |
| **集群管理** | Base Command | - | - | Slurm/K8s |
| **可观测性** | dcgm-exporter | rocm-smi | - | Prometheus |

---

## 五、企业选择非 NVIDIA 的典型场景

| 场景 | 推荐方案 | 原因 |
|------|----------|------|
| **推理成本优化** | AWS Inferentia2 | 成本降低 80%+ |
| **大规模训练(成本)** | AMD MI300X | 性能相当，成本低 30-40% |
| **最高性能训练** | Google TPU v6e | 硬件软件协设计 |
| **避免厂商锁定** | SYCL + Triton | 代码可跨平台 |
| **企业 IT 集成** | Intel Gaudi + oneAPI | CPU+GPU 统一 |

---

## 六、DCGM 深入学习

> **学习日期**: 2026-04-23  
> **实验环境**: dcgm-fake-gpu-exporter (Docker Compose)  
> **学习目标**: 理解 DCGM 架构，掌握 GPU 监控栈

### 6.1 DCGM 核心概念

#### DCGM vs NVML vs nvidia-smi

```
┌─────────────────────────────────────────────────────────────┐
│                      用户/应用                              │
└───────────────┬─────────────────────┬───────────────────────┘
                │                     │
    ┌───────────▼───────────┐ ┌───────▼───────┐
    │        DCGM           │ │  nvidia-smi   │
    │   (数据中心级管理)    │ │   (CLI工具)   │
    │                       │ │               │
    │ • 多GPU批量管理       │ │ • 单机查看    │
    │ • 后台持续监控        │ │ • 即时查询    │
    │ • 健康诊断            │ │ • 简单操作    │
    │ • 策略告警            │ │               │
    │ • Prometheus导出      │ │               │
    └───────────┬───────────┘ └───────┬───────┘
                │                     │
                └──────────┬──────────┘
                           │
                    ┌──────▼──────┐
                    │    NVML     │
                    │  (底层API)  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Driver    │
                    └─────────────┘
```

| 工具 | 定位 | 使用场景 |
|------|------|----------|
| **nvidia-smi** | CLI 工具 | 手动查看、简单脚本 |
| **NVML** | C API 库 | 自己写监控程序 |
| **DCGM** | 数据中心管理框架 | 集群监控、健康检查、告警 |

### 6.2 DCGM 架构

#### 运行模式

```
┌─────────────── 嵌入式模式 ───────────────┐
│                                          │
│  ┌────────────────────────────────────┐  │
│  │         你的应用程序               │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │   libdcgm.so (DCGM 共享库)   │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
│                                          │
│  优点: 简单，无需额外进程                │
│  缺点: 应用崩溃则监控中断                │
└──────────────────────────────────────────┘

┌─────────────── 独立模式（推荐）──────────┐
│                                          │
│  ┌──────────────┐    ┌──────────────┐   │
│  │ nv-hostengine│    │   你的应用   │   │
│  │  (守护进程)  │◄───│              │   │
│  │              │    │   dcgmi      │   │
│  │  libdcgm.so  │    │ dcgm-exporter│   │
│  └──────────────┘    └──────────────┘   │
│                                          │
│  优点: 独立运行，应用崩溃不影响监控      │
│  推荐: 生产环境使用                      │
└──────────────────────────────────────────┘
```

#### 监控栈数据流

```
┌─────────────┐     scrape      ┌────────────┐     query     ┌─────────┐
│dcgm-exporter│◀──────────────│ Prometheus │◀─────────────│ Grafana │
│   :9400     │   /metrics     │   :9090    │              │  :3000  │
└──────┬──────┘                └────────────┘              └─────────┘
       │ 读取
┌──────▼──────┐
│    DCGM     │
│  (GPU数据)  │
└─────────────┘
```

#### 核心功能

| 功能 | 说明 | 命令示例 |
|------|------|----------|
| **发现** | 列出所有 GPU | `dcgmi discovery -l` |
| **监控** | 实时查看指标 | `dcgmi dmon` |
| **诊断** | 健康检查 | `dcgmi diag -r 1/2/3` |
| **分组** | 批量管理 GPU | `dcgmi group -c mygroup` |
| **策略** | 设置告警阈值 | `dcgmi policy --set` |
| **统计** | 作业级监控 | `dcgmi stats` |

### 6.3 DCGM 关键指标

#### 真实 DCGM 指标命名（DCGM_FI_* 前缀）

| 指标 | 含义 | 监控重点 |
|------|------|----------|
| `DCGM_FI_DEV_GPU_UTIL` | GPU 利用率 % | 训练是否跑满 |
| `DCGM_FI_DEV_FB_USED` | 显存使用 MB | OOM 预警 |
| `DCGM_FI_DEV_FB_FREE` | 显存空闲 MB | 资源规划 |
| `DCGM_FI_DEV_FB_TOTAL` | 显存总量 MB | 基准参考 |
| `DCGM_FI_DEV_GPU_TEMP` | GPU 温度 °C | 散热问题 |
| `DCGM_FI_DEV_POWER_USAGE` | 功耗 W | 电力规划 |
| `DCGM_FI_DEV_SM_CLOCK` | SM 时钟 MHz | 是否降频 |
| `DCGM_FI_DEV_MEM_CLOCK` | 显存时钟 MHz | 内存带宽 |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | 内存拷贝利用率 % | 数据传输瓶颈 |

#### Fake Exporter 简化命名（实验用）

| Fake Exporter | 真实 DCGM |
|---------------|-----------|
| `dcgm_gpu_utilization` | `DCGM_FI_DEV_GPU_UTIL` |
| `dcgm_fb_used` | `DCGM_FI_DEV_FB_USED` |
| `dcgm_fb_free` | `DCGM_FI_DEV_FB_FREE` |
| `dcgm_fb_total` | `DCGM_FI_DEV_FB_TOTAL` |
| `dcgm_gpu_temp` | `DCGM_FI_DEV_GPU_TEMP` |
| `dcgm_power_usage` | `DCGM_FI_DEV_POWER_USAGE` |
| `dcgm_mem_copy_utilization` | `DCGM_FI_DEV_MEM_COPY_UTIL` |

### 6.4 dcgm-exporter + Prometheus + Grafana 实验

#### 实验环境

使用 [dcgm-fake-gpu-exporter](https://github.com/saiakhil2012/dcgm-fake-gpu-exporter) 模拟 GPU 监控栈。

```bash
# 克隆仓库
git clone https://github.com/saiakhil2012/dcgm-fake-gpu-exporter.git
cd dcgm-fake-gpu-exporter/deployments

# 启动完整监控栈
docker-compose -f docker-compose-demo.yml up -d

# 访问
# Metrics:    http://localhost:9400/metrics
# Prometheus: http://localhost:9090
# Grafana:    http://localhost:3000 (admin/admin)
```

#### 实验输出

**Metrics 端点** (`curl localhost:9400/metrics`):
```
dcgm_gpu_utilization{gpu="1",device="nvidia1"} 12.0
dcgm_gpu_utilization{gpu="2",device="nvidia2"} 24.0
dcgm_gpu_utilization{gpu="3",device="nvidia3"} 58.0
dcgm_gpu_utilization{gpu="4",device="nvidia4"} 50.0

dcgm_gpu_temp{gpu="1",device="nvidia1"} 45.0
dcgm_gpu_temp{gpu="2",device="nvidia2"} 49.0
dcgm_gpu_temp{gpu="3",device="nvidia3"} 60.0
dcgm_gpu_temp{gpu="4",device="nvidia4"} 62.0

dcgm_fb_used{gpu="1",device="nvidia1"} 4401.0
dcgm_fb_used{gpu="2",device="nvidia2"} 5066.0
dcgm_fb_used{gpu="3",device="nvidia3"} 7191.0
dcgm_fb_used{gpu="4",device="nvidia4"} 5612.0

dcgm_power_usage{gpu="1",device="nvidia1"} 125.0
dcgm_power_usage{gpu="2",device="nvidia2"} 156.0
dcgm_power_usage{gpu="3",device="nvidia3"} 213.0
dcgm_power_usage{gpu="4",device="nvidia4"} 194.0
```

**Prometheus 查询示例**:
```promql
# 所有 GPU 利用率
dcgm_gpu_utilization

# 平均利用率
avg(dcgm_gpu_utilization)

# 最高利用率
max(dcgm_gpu_utilization)

# 利用率超过 50% 的 GPU
dcgm_gpu_utilization > 50

# 显存使用率（已用/总量 %）
dcgm_fb_used / dcgm_fb_total * 100

# 过去5分钟平均
avg_over_time(dcgm_gpu_utilization[5m])
```

#### 模拟 GPU 数据解读

| GPU | 利用率 | 温度 | 显存使用 | 功耗 |
|-----|--------|------|----------|------|
| nvidia1 | 12% | 45°C | 4401 MB | 125W |
| nvidia2 | 24% | 49°C | 5066 MB | 156W |
| nvidia3 | 58% | 60°C | 7191 MB | 213W |
| nvidia4 | 50% | 62°C | 5612 MB | 194W |

### 6.5 学习检查点

完成本模块后应能：
- [x] 说出 DCGM → dcgm-exporter → Prometheus → Grafana 的数据流
- [x] 解释 GPU_UTIL vs FB_USED 的区别
- [x] 在 Prometheus 中查询 GPU 指标
- [x] 理解 DCGM 指标命名规范（DCGM_FI_* 前缀）
- [x] 区分嵌入式模式和独立模式

### 6.6 生产环境部署要点

#### 真实环境部署（有 GPU）

```bash
# 启动 dcgm-exporter（需要 GPU）
docker run -d --gpus all --cap-add SYS_ADMIN -p 9400:9400 \
  nvcr.io/nvidia/k8s/dcgm-exporter:4.5.2-4.8.1-distroless

# 验证指标
curl localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL
```

#### Kubernetes 部署

```bash
# 使用 Helm
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update
helm install dcgm-exporter gpu-helm-charts/dcgm-exporter
```

#### 关键配置

| 配置 | 推荐值 | 说明 |
|------|--------|------|
| `scrape_interval` | 15s | Prometheus 采集间隔 |
| `DCGM_EXPORTER_INTERVAL` | 1000ms | DCGM 指标更新间隔 |
| Grafana Dashboard | ID 12239 | 官方推荐 Dashboard |

---

## 七、参考资源

| 资源 | 链接 |
|------|------|
| DCGM 官方文档 | https://docs.nvidia.com/datacenter/dcgm/latest/ |
| DCGM Getting Started | https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/getting-started.html |
| dcgm-exporter GitHub | https://github.com/NVIDIA/dcgm-exporter |
| dcgm-fake-gpu-exporter | https://github.com/saiakhil2012/dcgm-fake-gpu-exporter |
| Grafana Dashboard #12239 | https://grafana.com/grafana/dashboards/12239 |
| NVIDIA AI Enterprise | https://docs.nvidia.com/ai-enterprise/ |
| GPU Operator | https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/ |
| Fabric Manager | https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/feature-overview.html |

---

**最后更新**: 2026-04-24
