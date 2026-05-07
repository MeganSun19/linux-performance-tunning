## Lustre:

Lustre: 分布式文件系统。特点：并行，分布式，高吞吐。

### Lustre 核心架构：

强解耦的三层结构：

```text
client 计算节点
｜
MDS （元数据)
｜
OSS （数据）
```

- MDS (Metadata Server)
  主要负责文件系统的目录结构和属性，包括： 文件名，目录结构，权限，文件大小，stripe信息。

- OSS (Object Storage Server) 对象存储服务器
  负责真正的数据块

- OST (Object Storage Target)
  每个OSS上挂多个磁盘，每个磁盘叫一个OST

数据实际在哪里？

一个文件 = 被切分（striping） --> 分布在多个OST上

Lustre的重点就在于striping.

eg. 100GB的文件：

strip size = 1G

Strip count = 4

数据会变成：

```text
块1 → OST1
块2 → OST2
块3 → OST3
块4 → OST4
块5 → OST1
...
```

并行文件系统（Lustre）的魔法：数据打散（Striping）。

当您往 Lustre 里存一个 100GB 的大文件时，Lustre 会把它切成 100 个 1GB 的小块，分别存放在 100 台不同的存储服务器（叫做 OSS，对象存储服务器）上。

当您的 GPU 去读取这个大文件时，不是向一台服务器要 100GB，而是同时向 100 台服务器各自要 1GB！

这就像把一个打饭窗口，瞬间变成了 100 个窗口同时递食物的超级自助餐大厅。

---

## NVMe

NVMe 固态硬盘： NVMe 是一种专门为闪存设计的协议。它的接口直接连在主板的 PCIe 总线上（电脑里的高速公路主干道）。这就好比磁悬浮列车直接开到了市中心，没有任何中转。

---

## NVMe-oF (NVMe over Fabrics)

NVMe-oF 就是利用 RDMA 技术（跑在 InfiniBand 或高速以太网线上），让计算节点去读写远程存储服务器上的 NVMe 硬盘时，感觉就像那块硬盘插在自己主板上一样快，一样没有延迟。

---

## GPUDirect Storage, 简称 GDS

传统的 IO 路径是：存储 -> CPU 内存 -> GPU 显存。GDS 技术允许通过 RDMA，让 存储 -> 网卡 -> PCIe Switch -> GPU 显存 直接传输数据，完全绕过 CPU Bounce Buffer，极大提升加载速度。

before GDS:

1. 数据从远程存储通过网卡进入计算节点。
2. 网卡把数据放到 CPU 的内存（RAM） 里。（这就是著名的 Bounce Buffer 弹跳缓冲区）。
3. CPU 看一眼数据，确认没问题。
4. 然后 CPU 再把数据从内存拷贝到 GPU 的显存（VRAM） 里。

缺点： CPU 成了最累的搬运工！这不仅增加了延迟，还会占满 PCIe 带宽（数据进一次 CPU，出一次 CPU，占用两遍带宽）。

after GDS:

1. 数据从远程存储进入网卡。
2. 网卡通过 PCIe 交换芯片，直接把数据“强行塞进” GPU 显存！CPU完全无感知。

---

## NVIDIA DGX SuperPOD 官方参考架构文档（以A100为例）

### HCA:

它是用于本机内部通信还是连接外部？

这里需要区分**“节点内”和“节点间”**的通信：

连接外部服务器（节点间）：这是 HCA 的核心职责。 HCA 的端口连接到外部的 InfiniBand 交换机（如物理机架中的叶交换机），从而让这台服务器能与 SuperPOD 中的其他成百上千台服务器对话。

本机内部通信（节点内）：主要不靠它。

虽然 HCA 插在本机内，但同一台 DGX 节点内部的 8 颗 GPU 之间如果要相互“说话”，走的是带宽更高的 NVLink（每颗 GPU 带宽达 600 GBps）和 NVSwitch。

什么时候会用到 HCA 做内部通信？ 只有当数据需要从本地存储（NVMe）或 CPU 内存传输到 GPU 显存，或者跨越不同的物理总线时，会经过 PCIe 通道。但严格来说，HCA 的主要设计目标是实现跨节点的高速互联和远程数据读取。

HCA 就是插在 PCIe 插槽上的 ConnectX 系列“超级网卡”。在 DGX A100 中，你确实能看到 8 个计算网口。它们的主要任务是作为“桥梁”，把这台机器内部强大的算力通过高速公路（InfiniBand 交换网络）与其他机器以及外部高速存储连接起来

### SU (scalable unit)

- Compute Nodes: 20台NVIDIA DGX A100 系统，每台A100 配备8个A100 GPU，两个CPU，以及用于计算、存储和管理的各种网卡（HCA）
- Networking Switches：
    1. Compute Fabric： 8台 NVIDIA Quantum QM8790 HDR 200G infiniband Leaf 交换机。每个DGX A100 系统通过8张HCA 分别连到这8台Leaf交换机，形成“轨道优化”拓扑。
    2. Storage Fabric： 2台 NVIDIA Quantum QM8790叶交换机
    3. In-Band Management： 2台NVIDIA Specturm SN4600 100 Ge 交换机
    4. Out-of-Band Managment: 2台NVIDIA AS 4610 1 Ge 交换机
- Managment&Physical infra
    1. Delicated Management Rack: 每个SU拥有一个专用机架，用于集中放置所有叶交换机
    2. Managment Servers: 这些服务器运行NVIDIA Base Command Manager,负责系统配置、负载均衡、监控和作业调度（Slurm）等关键服务。
    3. UFM Appliances: 每个SU通常配置2个NVIDIA Unified Fabric Manager. 分别用于管理compute fabric 和storage fabric的infiniband网络。
- Storage Hierachy
    1. RAM(系统内存)：每节点约2TB DDR4内存。
    2. Internal Storage: 每节点配备30TB 的NVMe固态硬盘，提供超55GBps的本地读取带宽。

总结：一个SU将20台算力巅峰的服务器与多达14台各型交换机通过一个专用的管理机架紧密集成，单SU可提供高达48 AI PFLOPS的计算性能。

---

## 单台 NVIDIA DGX SuperPOD 可扩展单元 (SU) 技术架构与连线指南

1. 架构概述与 SU 定义

在 NVIDIA DGX SuperPOD 参考架构中，可扩展单元 (Scalable Unit, 简称 SU) 是构建高性能 AI 计算基础设施的核心模块化组件。

* SU 标准定义： 单个 SU 由 20 台 NVIDIA DGX A100 系统组成。选择 20 个节点作为标准规模，旨在“优化计算性能与成本，同时最大限度地减少系统瓶颈”，使其成为支持复杂 AI 工作负载的理想构建块。
* 计算能力： 这种标准化、模块化的设计不仅实现了卓越的平衡性，还赋予了单台 SU 高达 48 AI PFLOPS 的峰值计算能力。

2. 机架硬件布局 (Hardware Layout)

为确保最优的散热效率、简化线缆管理并提供高效的空间分配，单台 SU 采用以下物理布局：

* 5 个计算机架 (Compute Racks)： 20 台 DGX A100 系统采用平均分布逻辑，每台计算机架放置 4 台节点。
* 1 个管理机架 (Management Rack)： 整个 SU 的网络中枢。该机架集中放置了所有的计算网络叶交换机、存储网络叶交换机、带内/带外管理交换机以及管理服务器，有效缩短了节点到核心交换层的链路长度。

3. 计算网络 (Compute Fabric) 详解

计算网络是实现超大规模分布式训练的物理基础，其设计必须能够支撑极高的通信吞吐量。

* 硬件配置： 单个 SU 必须配置 8 台计算叶交换机 (Leaf Switches) 和 5 台计算脊交换机 (Spine Switches)。所有交换机型号均指定为 NVIDIA Quantum QM8790 HDR 200Gb/s InfiniBand 交换机。
* 轨道优化 (Rail-Optimized) 连线逻辑： 这是 SuperPOD 架构获得极低延迟的关键。每台 DGX A100 系统上的 8 个 HCA 网卡（HCA 1-8）必须严格一一对应地连接至 8 台计算叶交换机（Leaf 1-8）。
* 设计逻辑： 这种 8 平面（8-plane）轨道优化设计确保了特定 GPU 对（例如 GPU 0 和 1）之间的流量始终在同一个物理“轨道”内交换，从而在 140 节点甚至更大规模的集群中维持极高性能。

20 节点轨道优化连接矩阵

| 节点网卡 | 目标交换机 | 逻辑描述 (GPU 关联) |
| --- | --- | --- |
| Node 1-20 (HCA 1) | Compute Leaf 1 | 轨道 1 连线：关联 GPU 1 & 2 |
| Node 1-20 (HCA 2) | Compute Leaf 2 | 轨道 2 连线：关联 GPU 1 & 2 |
| Node 1-20 (HCA 3) | Compute Leaf 3 | 轨道 3 连线：关联 GPU 3 & 4 |
| Node 1-20 (HCA 4) | Compute Leaf 4 | 轨道 4 连线：关联 GPU 3 & 4 |
| Node 1-20 (HCA 5) | Compute Leaf 5 | 轨道 5 连线：关联 GPU 5 & 6 |
| Node 1-20 (HCA 6) | Compute Leaf 6 | 轨道 6 连线：关联 GPU 5 & 6 |
| Node 1-20 (HCA 7) | Compute Leaf 7 | 轨道 7 连线：关联 GPU 7 & 8 |
| Node 1-20 (HCA 8) | Compute Leaf 8 | 轨道 8 连线：关联 GPU 7 & 8 |

* 拓扑结构： 叶交换机与 5 台脊交换机之间通过 Fat-Tree (胖树) 拓扑实现无阻塞连接，确保 SU 内部具备全双工双向带宽。

4. 存储网络 (Storage Fabric) 架构

存储网络专门用于处理模型训练期间的高密度 I/O，确保单节点吞吐量超过 40 GBps。

* 硬件规格： 根据单 SU (20 节点) 标准配置，存储网络由 4 台存储叶交换机 (Storage Leaf Switches) 和 2 台存储脊交换机 (Storage Spine Switches) 组成（型号为 QM8790）。
* 硬件冗余设计： 每台 DGX A100 系统使用两个存储端口进行连接。为实现硬件级的高可用性，这两条链路分别来自 两个独立的 ConnectX-6 双端口 HCA。这种配置确保了即便单张 HCA 卡发生故障，存储访问依然可以维持。
* 性能保障： 存储网络通过 RDMA 通信实现极低延迟，支持训练任务直接从远程存储以超过 16 GBps (每 GPU 2 GBps) 的速率读取数据。

5. 管理网络配置 (Management Networks)

管理网络分为带内和带外两部分，用于集群调度、监控及底层控制。

1. 带内管理网络 (In-band Management):
  * 配置 2 台 NVIDIA Spectrum SN4600 交换机，运行 NVIDIA Cumulus Linux。
  * 每台 DGX A100 提供 2 个 100GbE 接口，用于 Slurm 调度、文件系统元数据访问及容器镜像拉取。
2. 带外管理网络 (Out-of-Band Management):
  * 配置 2 台 NVIDIA AS4610 交换机，同样运行 Cumulus Linux。
  * 连接至每个节点的 BMC 接口，负责电源管理、环境监控及底层固件维护。

6. 关键连线矩阵总结 (Summary Matrix)

以下是单台可扩展单元 (SU) 在 20 节点标准配置下的网络硬件需求汇总：

| 网络类型 | 交换机数量 (单 SU) | 交换机型号 | 每节点连接数 |
| --- | --- | --- | --- |
| 计算网络 (Compute) | 8 叶 + 5 脊 | NVIDIA Quantum QM8790 | 8 (HDR InfiniBand) |
| 存储网络 (Storage) | 4 叶 + 2 脊 | NVIDIA Quantum QM8790 | 2 (来自 2 张 HCA) |
| 带内管理 (In-band) | 2 叶 | NVIDIA Spectrum SN4600 | 2 (100GbE) |
| 带外管理 (OOB) | 2 叶 | NVIDIA AS4610 | 1 (BMC 接口) |

7. 部署与优化注意事项

* 自适应路由 (Adaptive Routing)： 这是计算网络的核心设计要求。必须启用该功能以动态优化分布式训练中的通信路径，绕过局部拥塞。
* UFM 平台部署： 必须配置 NVIDIA Unified Fabric Manager (UFM) Enterprise。它结合了实时遥测与 AI 分析，对于监控 InfiniBand 网络的运行状况至关重要。
* 布线一致性： 严格遵循 SuperPOD 参考架构中的电缆长度和类型建议，以维持纳秒级的一致延迟表现。

专家提示： 20 节点 SU 虽然是一个完整且独立的功能单元，但其设计的精髓在于卓越的可扩展性。通过在第二层网络增加脊交换机和核心组 (Core Groups)，单 SU 架构可以无缝扩展至 140 节点甚至数百个节点，且无需更改现有的节点级布线，是支撑未来算力增长的标准化底座。
