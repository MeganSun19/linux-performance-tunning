# 在 Linux 下模拟 RoCEv2 ECN 拥塞标记

## 1. 实验架构与原理

本实验利用软件模拟 **RDMA 高压打流 -> 交换机队列拥塞 -> 触发 ECN 标记** 的完整网络行为。

*   **Client / 模拟交换机 (Ubuntu 1)**：
    1.  作为流量发送端，同时利用 `tc` (Traffic Control) 模拟一台拥塞的交换机。
    2.  利用 **HTB** 算法人为制造一条 100Mbps 的“窄路”引发拥塞。
    3.  利用 **RED** 算法在检测到拥塞时给数据包打上 ECN (CE) 标记。
*   **Server (Ubuntu 2)**：
    1.  作为流量接收端，利用 `tcpdump` 抓包验证 IP 头部中的 ECN 标记是否被成功修改。

---

## 2. 环境准备

### 2.1 安装必要组件
```bash
sudo apt update
sudo apt install -y rdma-core ibverbs-utils perftest iproute2 tcpdump
```

### 2.2 加载 Soft-RoCE 模块并绑定网卡
```bash
# 加载内核模块
sudo modprobe rdma_rxe
# 将物理网卡 ens192 绑定到 rxe0 实例
sudo rdma link add rxe0 type rxe netdev ens192
```

### 2.3 系统清理与基础配置
```bash
# 关闭防火墙，防止 UDP 4791 (RoCEv2) 端口被拦截
sudo ufw disable
sudo iptables -F
sudo iptables -t mangle -F

# 开启内核 ECN 支持
sudo sysctl -w net.ipv4.tcp_ecn=1
```

---

## 3. 实验步骤 (Lab)

### 3.1 在 Ubuntu 1 上配置 Linux TC 队列
```bash
# 1. 清除网卡上的旧规则
sudo tc qdisc del dev ens192 root 2>/dev/null

# 2. 建立 HTB 根队列，将网卡带宽强行限制为 100Mbit（我的网卡ens192实际底层跑在10Gpbs以上）
sudo tc qdisc add dev ens192 root handle 1: htb default 10
sudo tc class add dev ens192 parent 1: classid 1:10 htb rate 100mbit

# 3. 挂载 RED 算法并开启 ECN 功能
# 规则：当队列积压超过 30KB (min) 时，根据概率开始打标记；超过 90KB (max) 时全量打标记。
sudo tc qdisc add dev ens192 parent 1:10 handle 20: red \
  limit 1000000 min 30000 max 90000 probability 0.1 \
  avpkt 1000 burst 50 ecn
```

### 3.2 建立 RDMA 流量连接

1.  **Ubuntu 2 (Server) 启动监听**：
    ```bash
    ib_write_bw -d rxe0 -R
    ```

2.  **Ubuntu 1 (Client) 持续打流**：
    ```bash
    # 使用 -R 强制通过 RDMA-CM 握手
    ib_write_bw -d rxe0 10.124.148.188 -R --run_infinitely
    ```

3.  **Ubuntu 1 (Client) 动态修改 TOS**：
    *另起一个终端执行以下命令，为 RoCEv2 数据包宣告 ECN 能力。*
    ```bash
    # 将 TOS 设置为 2 (ECT(0))，告知交换机本流支持 ECN
    sudo iptables -t mangle -A OUTPUT -p udp --dport 4791 -j TOS --set-tos 2
    ```

---

## 4. 实验验证

### 4.1 查看 Ubuntu 1 的 TC 统计
```bash
tc -s qdisc show dev ens192
```
**期望结果**：
*   `backlog`：显示有几万字节的积压。
*   `marked`：该数字持续飙升，代表交换机成功识别到拥塞并进行了标记。
*   `early`：丢包停止增长或增长极其缓慢。

### 4.2 在 Ubuntu 2 抓包分析 IP 头部
```bash
# 过滤来自 Ubuntu 1 (10.124.148.187) 的 RoCEv2 数据包
sudo tcpdump -i ens192 src host 10.124.148.187 and udp port 4791 -c 20 -vvv
```
**期望结果**：
*   输出中包含 `IP (tos 0x3, ce ...)`。
*   **0x3** 代表原来的 `0x2` (ECT(0)) 被交换机改写成了 `0x3` (CE)。
*   `ce` 明确代表 **Congestion Encountered**。

---

## 5. 常见问题 (Troubleshooting)

在实验过程中，我们总结了 7 个经典的报错与解决方法：

### 坑 1：TC RED 配置报错 `Required parameter (limit, avpkt) is missing`
*   **现象**：Ubuntu 22.04 下无法直接执行简单的 `tc ... red ecn`。
*   **原理**：现代 Linux 的 `iproute2` 要求 RED 算法必须明确平均包大小 (`avpkt`) 和突发包数量 (`burst`) 才能准确计算时间窗口。
*   **解决**：命令中补充 `avpkt 1000 burst 50`。

### 2. `TOS only valid for rdma_cm` 报错
*   **现象**：使用 `-T 104` 尝试指定 TOS 时直接报错。
*   **原理**：默认的 Out-of-Band (TCP) 握手方式无法控制底层网卡的 IP 头部。
*   **解决**：加上 `-R` 参数，强制使用 **RDMA-CM** 握手。

### 3. 连接卡死，GID 为 `00:00:00...` 全零
*   **现象**：加上 `-R` 后连接失败，控制台显示 GID 是全 0。
*   **原理**：错误手动指定了无效的 GID 索引，或防火墙拦截了 UDP 4791 端口。
*   **解决**：移除 `-x` 参数让系统自动解析 GID，并确保已执行 `sudo ufw disable`。

### 4. 吞吐量跑满 1Gbps，但 `marked` 为 0
*   **现象**：连接成功且速度极快，但 TC 统计中没有标记产生。
*   **原理**：RED 触发的条件是队列发生积压。如果发送速度等于物理链路速度，包“随来随走”不排队，则不会触发标记。
*   **解决**：使用 `tc htb` 建立限速通道，人为制造“下行狭路”导致队列拥塞。

### 5. 加入 Iptables 规则后报错 `Unexpected CM event`
*   **现象**：若在建连前就开启 Iptables 规则，`ib_write_bw` 会连接失败。
*   **原理**：Iptables 会拦截并修改建连时的控制包。Soft-RoCE 状态机非常严苛，收到被改动的握手包会直接拒绝连接。
*   **解决**：采取 **“先上车，后补票”** 策略：先让数据跑起来，再动态启用 Iptables 规则。

### 6. `marked` 为 0，但 `early` (丢包) 飙升
*   **现象**：发生了拥塞，但全部变成丢包，没有 ECN 标记。
*   **原理**：ECN 位于 TOS 字段最后 2 bit。如果是 `104` (结尾 `00`)，代表不支持 ECN。RED 算法检测到包不支持 ECN 时，只能选择直接丢弃。
*   **解决**：设置 **TOS 2** (结尾 `10`)，宣告自己是 ECT (ECN Capable Transport)。

### 7. Tcpdump 抓到的依然是 `tos 0x0`
*   **现象**：TC 显示有大量标记，但抓包依然看不到 `tos 0x3`。
*   **原理**：视角盲区。抓到了接收端发回来的 **ACK 确认包**。对方没配 Iptables，所以回程包 TOS 依然是 0。
*   **解决**：使用 `src host` 过滤，精准狙击带有 ECN 标记的“去程业务包”。
