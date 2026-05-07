```text
megsun@MEGSUN-M-KHXH AI % docker run -it --rm nvcr.io/nvidia/pytorch:26.03-py3 bash

=============
== PyTorch ==
=============

NVIDIA Release 26.03 (build 286725636)
PyTorch Version 2.11.0a0+a6c236b
Container image Copyright (c) 2025, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
Copyright (c) 2014-2024 Facebook Inc.
Copyright (c) 2011-2014 Idiap Research Institute (Ronan Collobert)
Copyright (c) 2012-2014 Deepmind Technologies    (Koray Kavukcuoglu)
Copyright (c) 2011-2012 NEC Laboratories America (Koray Kavukcuoglu)
Copyright (c) 2011-2013 NYU                      (Clement Farabet)
Copyright (c) 2006-2010 NEC Laboratories America (Ronan Collobert, Leon Bottou, Iain Melvin, Jason Weston)
Copyright (c) 2006      Idiap Research Institute (Samy Bengio)
Copyright (c) 2001-2004 Idiap Research Institute (Ronan Collobert, Samy Bengio, Johnny Mariethoz)
Copyright (c) 2015      Google Inc.
Copyright (c) 2015      Yangqing Jia
Copyright (c) 2013-2016 The Caffe contributors
All rights reserved.

Various files include modifications (c) NVIDIA CORPORATION & AFFILIATES.  All rights reserved.

GOVERNING TERMS: The software and materials are governed by the NVIDIA Software License Agreement
(found at https://www.nvidia.com/en-us/agreements/enterprise-software/nvidia-software-license-agreement/)
and the Product-Specific Terms for NVIDIA AI Products
(found at https://www.nvidia.com/en-us/agreements/enterprise-software/product-specific-terms-for-ai-products/).

WARNING: The NVIDIA Driver was not detected.  GPU functionality will not be available.
   Use the NVIDIA Container Toolkit to start this container with GPU support; see
   https://docs.nvidia.com/datacenter/cloud-native/ .

NOTE: The SHMEM allocation limit is set to the default of 64MB.  This may be
   insufficient for PyTorch.  NVIDIA recommends the use of the following flags:
   docker run --gpus all --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 ...

root@21378d374290:/workspace# env | grep NVIDIA
NVIDIA_VISIBLE_DEVICES=all
NVIDIA_REQUIRE_CUDA=cuda>=9.0
NVIDIA_DRIVER_CAPABILITIES=compute,utility,video
NVIDIA_PRODUCT_NAME=PyTorch
NVIDIA_CPU_ONLY=1
NVIDIA_BUILD_ID=286725636
NVIDIA_PYTORCH_VERSION=26.03
root@21378d374290:/workspace# pip list | grep -E "torch|nccl|transformer|apex"
apex                             0.1
torch                            2.11.0a0+a6c236b9fd.nv26.3.46836102
torch_tensorrt                   2.11.0a0
torchao                          0.17.0+gitd9881220
torchdata                        0.11.0
torchtitan                       0.2.1+git71517cf6
torchvision                      0.25.0a0+b7d91027.nv26.3.46836102
transformer_engine               2.13.0+28777046
root@21378d374290:/workspace# find /usr -name "libnccl*"
/usr/share/doc/libnccl-dev
/usr/share/doc/libnccl2
/usr/lib/aarch64-linux-gnu/libnccl.so
/usr/lib/aarch64-linux-gnu/libnccl.so.2
/usr/lib/aarch64-linux-gnu/libnccl.so.2.29.7
root@21378d374290:/workspace# ls /usr/local/cuda/bin
__nvcc_device_query  compute-sanitizer  cuda-gdb          cuda-gdb-python3.10-tui  cuda-gdb-python3.12-tui  cuda-gdb-python3.9-tui  cudafe++   fatbinary  nvcc          nvdisasm  nvprune  tileiras
bin2c                crt                cuda-gdb-minimal  cuda-gdb-python3.11-tui  cuda-gdb-python3.8-tui   cuda-gdbserver          cuobjdump  nsys       nvcc.profile  nvlink    ptxas
root@21378d374290:/workspace# nvcc --version
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2026 NVIDIA Corporation
Built on Mon_Mar_02_09:52:54_PM_PST_2026
Cuda compilation tools, release 13.2, V13.2.51
Build cuda_13.2.r13.2/compiler.37434383_0
```

这份 Output 非常精彩！它完美展现了一个**生产级 AI 容器**的内部构造。虽然你是在 Mac 上运行（没有 NVIDIA 显卡），但容器里的每一个细节都映射了你学习计划表里的知识点。

我为你把这个输出拆解为五个核心维度，结合你的学习计划进行深度解释：

### 1. 启动警告（系统调优的伏笔）

> `WARNING: The NVIDIA Driver was not detected.`
> `NOTE: The SHMEM allocation limit is set to the default of 64MB... NVIDIA recommends... --ipc=host`

- **解释：** 容器检测到你本地没有 NVIDIA 驱动。
- **知识点关联（Phase 1 性能优化）：** 那个 `SHMEM` 的提示极其重要。在多显卡训练（NCCL）时，GPU 之间需要通过共享内存（Shared Memory）交换数据。如果以后你在 DGX 上跑容器，**必须加上 `--ipc=host`**。否则，你的训练会在开始几秒后因为共享内存不足直接崩掉。

---

### 2. 环境变量（身份证明）

> `NVIDIA_PYTORCH_VERSION=26.03`
> `NVIDIA_PRODUCT_NAME=PyTorch`
> `NVIDIA_CPU_ONLY=1`

- **解释：**
  - `26.03` 是 NVIDIA 的版本号逻辑（YY.MM）。
  - `NVIDIA_CPU_ONLY=1` 是因为没检测到驱动，容器自动切换到了“只读模式”。
- **知识点关联（NGC Catalog）：** 每一个来自 NGC 的镜像都有这些环境变量，方便脚本判断当前处于什么环境。

---

### 3. 核心软件包（你的“核武器库”）

> `apex 0.1`
> `torch 2.11.0...nv26.3...`
> `transformer_engine 2.13.0`
> `torchtitan 0.2.1`

- **重点解释：**
  - **`nv26.3` 后缀：** 注意看 `torch` 的版本，它不是官方社区版，而是 **NVIDIA 深度优化版**。它内置了针对 H100 架构的算子优化。
  - **`transformer_engine`：** **这是 H100 的灵魂。** H100 最大的卖点是 **FP8 训练**，而 FP8 的逻辑全部封装在这个 `transformer_engine` 里。没有它，H100 就发挥不出全部实力。
  - **`apex`：** 这是 NVIDIA 开发的混合精度训练工具库，也是 Phase 2 学习分布式训练的基础。
- **知识点关联（Phase 2 分布式训练）：** 以后你学 DDP 和 FSDP 时，就会用到这些库。

---

### 4. NCCL 库（集群通信的血液）

> `/usr/lib/aarch64-linux-gnu/libnccl.so.2.29.7`

- **重点解释：**
  - **`aarch64`：** 因为你的 Mac 是 M 芯片（ARM 架构），Docker 自动拉取了 ARM 版本的镜像。如果是 DGX（x86），路径会是 `x86_64-linux-gnu`。
  - **`libnccl`：** 看到这个文件了吗？这就是你计划表里 **04-18** 要学的 **NCCL (NVIDIA Collective Communications Library)**。
- **知识点关联（04-18 NCCL 集合通信）：** 它是多机多卡通信的底层协议。你不需要自己去官网上找 `.so` 文件，NVIDIA 已经帮你精准地放在了容器里。

---

### 5. CUDA 工具链（开发者的铲子）

> `nvcc`, `nvlink`, `nsys`, `ptxas`
> `Cuda compilation tools, release 13.2`

- **重点解释：**
  - **`nvcc`：** CUDA 的编译器。
  - **`nvlink`：** 这里居然有一个名为 `nvlink` 的可执行文件！它用于在编译阶段处理 GPU 之间的互联链接逻辑。
  - **`nsys`：** 这是 **NVIDIA Nsight Systems**。以后你要分析集群性能瓶颈（比如为什么网络慢），就靠这个 `nsys`。
  - **`CUDA 13.2`：** 这是非常超前的版本（目前的生产主流多为 12.x）。这说明 26.03 镜像确实是面向未来的。
- **知识点关联（04-25 NVLink/NVSwitch）：** 当你学到物理连接时，你会回想起在容器里已经见过这些控制工具。

---

### 总结：你现在的学习产出

你刚才的操作已经证明了你已经掌握了 **“AI 环境审计”** 的能力。

**你应该在你的 04-13 笔记里写下：**

1. **NGC 镜像的真相：** 它不只是 PyTorch，它是一个高度集成的 **“全栈 AI 运行环境”**，包含了针对 H100 优化的 Transformer Engine 和 NCCL。
2. **版本依赖：** 镜像版本（如 26.03）决定了 CUDA（13.2）和 PyTorch 的配套关系，这种“官方全家桶”极大地降低了环境部署的故障率。
3. **跨平台性：** 即使在 Mac ARM 架构上，依然可以拉取镜像进行环境查看和脚本预开发。

**下一步建议：**

你可以试着在容器里运行 `python`，然后输入：

```python
import torch
print(torch.__version__)
print(torch.cuda.is_available()) # 应该是 False
```

虽然它是 False，但你已经通过“外科手术”式的方式，看清了一个 AI 超级计算环境的五脏六腑。这比单纯看文档强百倍！
