# 在 Linux 下模拟 RoCEv2 PFC (QoS 队列隔离)

## 1. 实验目的

利用 Linux `tc prio` 机制构建“多车道”网络，将普通 TCP 流量导向“慢车道”，并将 RoCE 流量识别并优先放入“VIP 专属不丢包快车道”。

*   **测试环境**：两台 Ubuntu 主机（Ubuntu 1 作为 Client，Ubuntu 2 作为 Server）。
*   **技术说明**：Soft-RoCE (RXE) 无法完整模拟真实硬件的 PFC (802.1Qbb) 行为。真实的 PFC 作用于数据链路层（L2），依赖物理网卡和交换机的 ASIC 芯片发送 Pause Frame。本实验旨在模拟 **QoS 优先级分类**与**多队列隔离**。

---

## 2. 实验步骤

### 2.1 环境清理
```bash
# 清除旧的队列规则与 mangle 表规则
sudo tc qdisc del dev ens192 root 2>/dev/null
sudo iptables -t mangle -F OUTPUT
```

### 2.2 创建“多车道”优先级队列 (Ubuntu 1)
使用 Linux 原生的 `prio` 机制创建 3 个波段（Band 0, 1, 2），Band 0 拥有最高优先级。

```bash
# 1. 建立 3 个优先级车道（1:1 最高优先，1:2 默认，1:3 最低优先）
sudo tc qdisc add dev ens192 root handle 1: prio bands 3

# 2. 配置引流规则：匹配 IP 头中 TOS 为 104 (0x68) 的包，送入 1:1 队列
sudo tc filter add dev ens192 protocol ip parent 1: prio 1 u32 match ip tos 0x68 0xff flowid 1:1
```
> **注**：默认情况下，Linux 会将未匹配的普通流量放入 Band 1 (1:2)。

### 2.3 制造背景干扰流量
1.  **Ubuntu 2 (Server)**: `iperf3 -s`
2.  **Ubuntu 1 (Client)**: `iperf3 -c 10.124.148.188 -t 3600`

### 2.4 启动 RoCE 业务流量
此时 RoCE 流量的 TOS 仍为默认值 0，正与 iperf3 流量挤在同一个通道。
1.  **Ubuntu 2 (Server)**: `ib_write_bw -d rxe0 -R`
2.  **Ubuntu 1 (Client)**: `ib_write_bw -d rxe0 10.124.148.188 -R --run_infinitely`

### 2.5 为 RoCE 流量打上优先级标签 (Ubuntu 1)
在流量运行过程中，另开终端执行以下命令，强行篡改后续 RoCE 数据包的 TOS：
```bash
# 将 RoCE (UDP 4791) 的 TOS 设置为 104，触发 tc filter 规则
sudo iptables -t mangle -A OUTPUT -p udp --dport 4791 -j TOS --set-tos 104
```

---

## 3. 实验验证 (Ubuntu 1)

执行统计查看命令：
```bash
tc -s class show dev ens192
```

**期待结果**：
*   **1:1 (Band 0)**：`Sent` 字节数持续增加，代表 RoCE 业务流已成功进入专属快车道。
*   **1:2 (Band 1)**：字节数巨大，代表 iperf3 垃圾流量及之前未标记的 RoCE 包在此排队。

---

## 4. 与真实物理硬件 PFC 的区别

在搭载 ConnectX-6/7 网卡和智能交换机的真实 AI 数据中心中，硬件 PFC 与本软件实验存在本质区别：

1.  **无损传输 (Lossless) vs 简单队列隔离**：
    *   **本实验**：若发送过猛导致队列溢出，Linux 内核仍会丢弃 (Drop) 报文。
    *   **真实硬件**：交换机在硬件队列水位达到警戒线时，会向网卡发送 **802.1Qbb Pause 帧**。网卡收到后物理性停止相应优先级队列的发送，确保 **零丢包**。

2.  **标签注入时机 (Data Plane vs Control Plane)**：
    *   **本实验**：只能在连接建立后用 `iptables` 动态篡改，提前配置会破坏握手协议。
    *   **真实硬件**：网卡驱动原生支持 `cma_roce_tos` 参数，能智能识别并仅在数据传输阶段打标，不影响连接协商。

3.  **计算载体 (CPU vs ASIC)**：
    *   **本实验**：依赖主机 CPU 处理中断和系统内存 (RAM) 缓存，高负载下会产生软中断风暴。
    *   **真实硬件**：由交换机和网卡的 **ASIC 芯片** 在片上高速缓存中处理，纳秒级延迟且不消耗主机 CPU 资源。

4.  **工作层次 (L3 vs L2)**：
    *   **本实验**：基于 IP 层 (L3) 的 TOS 字段。
    *   **真实硬件**：PFC 通过 DSCP (L3) 映射到 VLAN Tag 中的 **802.1p (PCP)** 字段，在以太网 MAC 层 (L2) 直接工作。
