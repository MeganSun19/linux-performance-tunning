# Slurm

## 1. 概述

- **定义**：Slurm 是一个开源、容错且具有高度可扩展性的集群管理和作业调度系统，专为各种规模的 Linux 集群设计，运行时不需要修改内核，相对独立。
- **三大核心功能**：
  1. **资源分配**：在特定时间内为用户分配计算节点（资源）的独占或非独占访问权限。
  2. **作业执行框架**：提供在分配的节点上启动、执行和监控工作（通常是并行作业）的框架。
  3. **争用仲裁**：通过管理待处理作业的队列，解决对资源的争用问题。

---

## 2. 架构及管理实体

### 架构组件

- **slurmctld**：中央管理守护进程，运行在管理节点上（可选备份节点以实现容错），负责监控资源和作业状态。
- **slurmd**：运行在每个计算节点上的守护进程，负责执行作业并提供容错的分层通信。

> 下列命令可在集群中任意位置运行，用于提交、管理和监控作业。

### 管理实体

- **Nodes**：集群中的计算资源。
- **Partitions**：将节点分组成逻辑集合（可能重叠），相当于作业队列；每个分区有特定限制（如作业大小、时间限制、用户权限等）。
- **Jobs**：指定时间内分配给用户的资源配额。
- **Job steps**：作业内的一组（通常是并行的）任务；一个作业可包含多个顺序或并行执行的作业步。

---

## 3. 常用命令

### 作业管理

#### `sbatch`

`is used to submit a job script for later execution`；脚本通常包含一个或多个 `srun` 以启动并行任务（`the script typically contains one or more srun commands`）。

```bash
adev0: cat my.script
#!/bin/sh
#SBATCH --time=1
/bin/hostname
srun -l /bin/hostname
srun -l /bin/pwd

adev0: sbatch -n4 -w "adev[9-10]" -o my.stdout my.script
sbatch: Submitted batch job 469

adev0: cat my.stdout
adev9
0: adev9
1: adev9
2: adev10
3: adev10
0: /home/jette
1: /home/jette
2: /home/jette
3: /home/jette
```

```bash
adev0: sbatch test
srun: jobid 473 submitted

adev0: squeue
JOBID PARTITION NAME USER ST TIME  NODES NODELIST(REASON)
  473 batch     test jill R  00:00 1     adev9

adev0: scancel 473

adev0: squeue
JOBID PARTITION NAME USER ST TIME  NODES NODELIST(REASON)
```

#### `srun`

实时提交作业或发起作业步，可指定资源：最大/最小节点数、处理器数、内存等。

无已有作业分配时，`srun` 会向调度器**新申请**资源；例如在两节点 CPU 分区上：

```bash
[root@slurmctld data]# srun -N 2 hostname
c1
c2

[root@slurmctld data]# srun -N 2 -n 2 hostname
c1
c2
```

`--ntasks` 与选项之间**必须有空格**，否则会被解析成一个参数（如 `-n2hostname` 会报错）。

#### `salloc`

实时分配资源，通常用于分配资源并启动一个 shell，再在该 shell 中运行 `srun`。

```bash
tux0: salloc -N1024 bash
$ sbcast a.out /tmp/joe.a.out
Granted job allocation 471
$ srun /tmp/joe.a.out
Result is 3.14159
$ srun rm /tmp/joe.a.out
$ exit
salloc: Relinquishing job allocation 471
```

#### `scancel`

取消挂起或运行中的作业 / 作业步。

### 状态监控与信息查询

#### `sinfo`

`reports the state of partitions and nodes managed by Slurm`。

```bash
adev0: sinfo
PARTITION AVAIL  TIMELIMIT NODES  STATE NODELIST
debug*       up      30:00     2  down* adev[1-2]
debug*       up      30:00     3   idle adev[3-5]
batch        up      30:00     3  down* adev[6,13,15]
batch        up      30:00     3  alloc adev[7-8,14]
batch        up      30:00     4   idle adev[9-12]
```

#### `squeue`

`reports the state of jobs or job steps`。默认先按优先级列出运行中的作业，再列出排队中的作业。

```bash
adev0: squeue
JOBID PARTITION  NAME  USER ST  TIME NODES NODELIST(REASON)
65646     batch  chem  mike  R 24:19     2 adev[7-8]
65647     batch   bio  joan  R  0:09     1 adev14
65648     batch  math  phil PD  0:00     6 (Resources)
```

- `R`：运行中（Running）；`PD`：排队中（Pending）；`NODELIST` 会显示挂起原因。

#### `sacct`

`report job or job steps accounting info about the active or completed jobs`。

作业很短时 `squeue` 往往已是空表，用 `sacct` 仍能看到 **JobID / 分区 / 状态 / 退出码**，以及 `**.batch`、`.0`、`.1`** 等作业步行，例如：

```bash
[root@slurmctld data]# sacct
JobID           JobName  Partition    Account  AllocCPUS      State ExitCode 
------------ ---------- ---------- ---------- ---------- ---------- -------- 
19           multi_ste+        cpu       root          2  COMPLETED      0:0 
19.batch          batch                  root          1  COMPLETED      0:0 
19.0           hostname                  root          2  COMPLETED      0:0 
19.1                pwd                  root          1  COMPLETED      0:0 
20                 steps        cpu       root          2  COMPLETED      0:0 
20.0           hostname                  root          4  COMPLETED      0:0 
20.1           hostname                  root          2  COMPLETED      0:0 
6              gpu-test        gpu       root          0    PENDING      0:0 
```

最后一行常为 `**make run-examples**` 提交的 `**gpu_test**`：分区无 GPU 节点时会长期 `**PENDING**`，可用 `**scancel 6**` 取消。

#### `scontrol`

`is the admin tool used to view and/or modify Slurm state`；通常仅 root 可执行（`can only be executed as root`）。

```bash
adev0: scontrol show partition
PartitionName=debug TotalNodes=5 TotalCPUs=40 RootOnly=NO
   Default=YES OverSubscribe=FORCE:4 PriorityTier=1 State=UP
   MaxTime=00:30:00 Hidden=NO
   MinNodes=1 MaxNodes=26 DisableRootJobs=NO AllowGroups=ALL
   Nodes=adev[1-5] NodeIndices=0-4

PartitionName=batch TotalNodes=10 TotalCPUs=80 RootOnly=NO
   Default=NO OverSubscribe=FORCE:4 PriorityTier=1 State=UP
   MaxTime=16:00:00 Hidden=NO
   MinNodes=1 MaxNodes=26 DisableRootJobs=NO AllowGroups=ALL
   Nodes=adev[6-15] NodeIndices=5-14

adev0: scontrol show node adev1
NodeName=adev1 State=DOWN* CPUs=8 AllocCPUs=0
   RealMemory=4000 TmpDisk=0
   Sockets=2 Cores=4 Threads=1 Weight=1 Features=intel
   Reason=Not responding [slurm@06/02-14:01:24]

adev0: scontrol show job
JobId=65672 UserId=phil(5136) GroupId=phil(5136)
   Name=math
   Priority=4294901603 Partition=batch BatchFlag=1
   AllocNode:Sid=adev0:16726 TimeLimit=00:10:00 ExitCode=0:0
   StartTime=06/02-15:27:11 EndTime=06/02-15:37:11
   JobState=PENDING NodeList=(null) NodeListIndices=
   NumCPUs=24 ReqNodes=1 ReqS:C:T=1-65535:1-65535:1-65535
   OverSubscribe=1 Contiguous=0 CPUs/task=0 Licenses=(null)
   MinCPUs=1 MinSockets=1 MinCores=1 MinThreads=1
   MinMemory=0 MinTmpDisk=0 Features=(null)
   Dependency=(null) Account=(null) Requeue=1
   Reason=None Network=(null)
   ReqNodeList=(null) ReqNodeListIndices=
   ExcNodeList=(null) ExcNodeListIndices=
   SubmitTime=06/02-15:27:11 SuspendTime=None PreSusTime=0
   Command=/home/phil/math
   WorkDir=/home/phil
```

#### `sview`

图形界面工具，用于查看 partitions、nodes 与 jobs。

### 其他工具

- `**sbcast**`：`is used to transfer a file from local disk to local disk on the nodes allocated to a job`；可用于无盘节点或相对共享文件系统提升性能。
- `**sprio**`：`is used to display a detailed view of the components affecting a job's priority`。
- `**sstat**`：`is used to get information about the resources utilized by a running job or job step`。

---

## 4. 作业与作业步：核心区别


| 维度       | 作业 (Job)                       | 作业步 (Job Step)                          |
| -------- | ------------------------------ | --------------------------------------- |
| **本质属性** | 资源的分配（**Allocation**）①。        | 任务的执行（**Execution**）①。                  |
| **资源范围** | 由调度器根据分区约束和优先级分配的一组节点①。        | 在作业已分配的节点范围内进行配置，可以占用全部节点，也可以只占用部分节点①②。 |
| **执行方式** | 在队列中排队等待分配资源①③。                | 在已获得的资源中，可以**顺序执行**或**并行执行**②④。         |
| **管理开销** | 调度器（slurmctld）的管理开销相对较高⑤。      | 管理开销远低于独立作业，适合处理大量相关的小型工作⑤。             |
| **启动命令** | 通常通过 `sbatch` 或 `salloc` 申请⑥⑦。 | 通常在作业脚本或分配的 Shell 中通过 `srun` 发起④⑧。      |


---

## 5. 实践记录：giovtorres/slurm-docker-cluster（Mac）

以下是在本地 Mac 上用 [giovtorres/slurm-docker-cluster](https://github.com/giovtorres/slurm-docker-cluster) 做练习时的流程与结论摘要（Slurm 25.11.x，`cpu` 分区节点 `c1`、`c2`）。

### 5.1 宿主机 vs 容器：命令在哪里执行


| 场景                                                                                         | 位置                                   | 说明                                                                          |
| ------------------------------------------------------------------------------------------ | ------------------------------------ | --------------------------------------------------------------------------- |
| `make up` / `make down` / `make run-examples` / `make jobs` / `make scale-cpu-workers N=5` | **Mac 上仓库根目录**                       | `Makefile` 在宿主机；容器内没有这些 target，会出现 `No rule to make target 'run-examples'`。 |
| `make shell`                                                                               | 宿主机执行，进入 **slurmctld 容器**            | 等价于 `docker exec -it slurmctld bash`，工作目录常见为 `/data`。                       |
| `sbatch` / `srun` / `squeue` / `sacct` / `scontrol`                                        | **容器内**（或 `docker exec slurmctld …`） | 与 Slurm 集群交互。                                                               |


`make run-examples` 会把示例脚本拷到容器（如 `/data/examples/`）并在容器里 `sbatch`；交互菜单里选「Run all」可一次提交多个示例作业。

**若在容器内误执行 `make`**（当前目录无 `Makefile`）：

```bash
[root@slurmctld data]# make run-examples
make: *** No rule to make target 'run-examples'.  Stop.
```

### 5.2 集群视图与分区

```bash
[root@slurmctld data]# sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
cpu*         up   infinite      2   idle c[1-2]
gpu          up   infinite      0    n/a 
```

- `**cpu***`：默认分区，两台动态注册的 CPU 计算节点。
- `**gpu**`：`scontrol show partitions` 中 `Nodes=(null)`、`TotalNodes=0`；未启用 GPU worker 时，提交到 `gpu` 的作业会一直 `**PD**`，原因类似 `**(PartitionConfig)**`。

示例里 `**gpu_test.sh**` 会占一个永远排不到的 GPU 作业；练习结束后可用 `**scancel <JOBID>**` 清掉。

`**squeue`（GPU 作业永远排队）**：

```bash
[root@slurmctld data]# squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
                 6       gpu gpu-test     root PD       0:00      1 (PartitionConfig)
```

`**scontrol show partitions`（节选：cpu 有两节点，gpu 无节点）**：

```bash
[root@slurmctld data]# scontrol show partitions
PartitionName=cpu
   ...
   Nodes=c[1-2]
   ...
   TotalCPUs=24 TotalNodes=2 ...
PartitionName=gpu
   ...
   NodeSets=gpu_nodes
   Nodes=(null)
   ...
   TotalCPUs=0 TotalNodes=0 ...
```

### 5.3 切勿在 shell 里误跑 `slurmctld`

`slurmctld` 是**已在容器里由入口脚本启动好的守护进程**，不是查看 `slurm-*.out` 的命令。

若把输出文件名当成参数执行（如误输入 `slurmctld slurm-1.out`，由 Tab 补全混淆），可能干扰主进程，shell 以 `**make: *** [shell] Error 137`** 结束（常见为 `**SIGKILL**`，例如 OOM 或进程被强杀）。随后 `**slurmctld` 容器为 Exited**，`make shell` 报 *container is not running*。

**恢复**：在仓库目录执行 `**make up`**（或 `docker compose up -d`），待 `slurmctld` 再次 **healthy** 后再 `make shell`。

查看作业输出应使用 `**cat` / `less` / `tail`**，例如：`cat slurm-<JobID>.out`。

宿主机上若出现 `**make: *** [shell] Error 137**`，多为上述异常导致容器退出；在 Mac 上 `**docker compose ps -a**` 可见 `slurmctld` 为 `**Exited**`。

### 5.4 `squeue` 空、`sacct` 有记录

短作业结束极快，`squeue` 经常为空属正常；历史与状态用 `**sacct**`（及输出文件）查看。

```bash
[root@slurmctld data]# squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)

[root@slurmctld data]# sbatch --wrap="hostname"
Submitted batch job 2

[root@slurmctld data]# sacct -j 2 --format=JobID,JobName,Partition,State,ExitCode,AllocCPUS
JobID           JobName  Partition      State ExitCode  AllocCPUS 
------------ ---------- ---------- ---------- -------- ---------- 
2                  wrap        cpu  COMPLETED      0:0          1 
2.batch           batch             COMPLETED      0:0          1 
```

（`--format` 可按需增减列；不写则使用默认列集。）

### 5.5 `scontrol` 用法要点

`scontrol` 是**子命令驱动**的交互/批处理工具，进入交互后需输入合法关键字（如 `show config`、`show partitions`、`show nodes`），**不要**把 `root` 当成子命令：

```text
scontrol: root
invalid keyword: root
```

常用只读示例：

```bash
scontrol show config
scontrol show partitions
scontrol show nodes
scontrol show nodes c1
```

完整终端输出（含 `**Slurmctld(primary)…**` 收尾行）见下文 **5.11**。

`**show nodes` 节选（动态节点 `IDLE+DYNAMIC_NORM`）**：

```bash
[root@slurmctld data]# scontrol show nodes c1
NodeName=c1 Arch=aarch64 CoresPerSocket=12 
   CPUAlloc=0 CPUEfctv=12 CPUTot=12 CPULoad=0.39
   AvailableFeatures=cpu
   ...
   State=IDLE+DYNAMIC_NORM ...
   Partitions=cpu 
```

### 5.6 `#SBATCH` 与「批内 `srun` 不能超过作业分配」

- 以 `**#SBATCH**` 开头的行由 `**sbatch` 解析**，不是普通注释；用于申请整作业的资源与时间。
- 在 `**sbatch` 脚本**里，`srun` 只能使用**该作业已分配到的节点/CPU**；若脚本未请求 2 节点却写 `srun -N 2`，会报错：  
`**Only allocated 1 nodes asked for 2`**。  
解决办法：在脚本顶部增加例如 `**#SBATCH -N 2**`（以及需要的 `**#SBATCH --partition=cpu**`、`-o` 等）。
- 在容器 shell 里**直接**执行 `srun -N 2 hostname` 时，`srun` 会**向调度器新申请**资源，不受「当前无作业」限制，因此可以立刻拿到 `c1`、`c2`。

**未在脚本里申请 2 节点时，批内 `srun -N 2` 失败输出**：

```bash
[root@slurmctld data]# cat slurm-16.out
=== step1: 2 nodes, hostname ===
srun: error: Only allocated 1 nodes asked for 2
=== step2: 1 node, pwd ===
/data
```

**补上 `#SBATCH -N 2` 等之后，同一逻辑成功**：

```bash
[root@slurmctld data]# cat slurm-19.out
=== step1: 2 nodes, hostname ===
c1
c2
=== step2: 1 node, pwd ===
/data
```

### 5.7 示例脚本位置与提交方式

镜像内示例路径：`**/root/examples/jobs/**`（属主常为 `slurm`，`root` 一般仍可读）。

- **正确**：`sbatch array_job.sh`（需在可访问输出路径的分区/目录下提交，或脚本内已写绝对路径如 `/data/...`）。
- **错误**：`bash array_job.sh` —— 不会经过调度器，`SLURM_ARRAY_TASK_ID` 等变量为空，行为与数组作业无关。

**用 `bash` 直接跑脚本（无 Slurm 环境）**：

```bash
[root@slurmctld jobs]# bash array_job.sh
Array job task  of array job 
Running on: slurmctld
Task started at: Tue Apr 21 04:51:56 UTC 2026
Task  completed at: Tue Apr 21 04:51:56 UTC 2026
```

**用 `sbatch` 提交后（节选某 array task 输出文件）**：

```bash
[root@slurmctld data]# cat array_test_4_4.out
Array job task 4 of array job 4
Running on: c2
Task started at: Tue Apr 21 04:57:15 UTC 2026
Task 4 completed at: Tue Apr 21 04:57:19 UTC 2026
```

顺序跑多个示例：在宿主机用 `**make run-examples**`；或在容器内对多个脚本逐个 `**sbatch**`。需要「上一个跑完再跑下一个」时，可对 Slurm 25.x 尝试 `**sbatch --wait 脚本.sh**`，或用 `**--dependency=afterok:前一个JobID**` 链式提交。

### 5.8 本仓库练习中的若干命令片段

**多步脚本（先 2 节点 hostname，再 1 节点 pwd）** — 脚本内需 `#SBATCH -N 2` 等：

```bash
#!/bin/bash
#SBATCH --partition=cpu
#SBATCH -N 2
#SBATCH -n 2
#SBATCH -o slurm-%j.out

echo "=== step1: 2 nodes, hostname ==="
srun -N 2 -n 2 hostname

echo "=== step2: 1 node, pwd ==="
srun -N 1 -n 1 pwd
```

**同作业内多 `srun`、带 task 标签**：

```bash
#!/bin/bash
#SBATCH -J steps
#SBATCH --partition=cpu
#SBATCH -N 2
#SBATCH -n 4
#SBATCH -o slurm-%j.out

srun -n 4 -l hostname
srun -N 2 -n 2 -l hostname
```

**对应输出（`-l` 为 task 编号前缀；两行 `srun` 为两个 step）**：

```bash
[root@slurmctld data]# cat slurm-20.out
0: c1
3: c2
2: c1
1: c1
1: c2
0: c1
```

**资源与 Slurm 环境变量**：

```bash
srun -n 2 --cpus-per-task=2 bash -c 'echo OMP_NUM_THREADS=$OMP_NUM_THREADS; nproc'
srun -N 1 --mem=512 hostname
srun -N 1 env | grep SLURM
```

`**--cpus-per-task` 与 `env | grep SLURM` 节选**：

```bash
[root@slurmctld data]# srun -n 2 --cpus-per-task=2 bash -c 'echo OMP_NUM_THREADS=$OMP_NUM_THREADS; nproc'
OMP_NUM_THREADS=
OMP_NUM_THREADS=
2
2

[root@slurmctld data]# srun -n 4 hostname
c1
c1
c1
c1

[root@slurmctld data]# srun -N 1 env | grep SLURM
SLURM_CLUSTER_NAME=linux
SLURM_JOB_ID=25
SLURM_NNODES=1
SLURM_NODELIST=c1
SLURM_JOB_PARTITION=cpu
SLURM_CPUS_ON_NODE=1
...
```

注意：单独 `srun -n 4 hostname` 若只分到 **1 个节点**，四个 task 可能**都在同一节点**（如四个 `c1`）；与 `**sbatch` 里已申请 `-N 2`** 时的分布不同。

**后台两个 `srun` 再 `wait`**：标准输出可能等两个 step 都结束才完整写入 `slurm-*.out`，属正常现象。

```bash
[root@slurmctld data]# cat slurm-21.out
srun: Step created for StepId=21.1
both step groups finished
```

### 5.9 宿主机 zsh 与 `docker exec`

在 Mac **zsh** 下执行：

```bash
docker exec slurmctld ls /data/*.out
```

若本机当前目录无匹配，zsh 可能在**本地**展开 glob 失败：

```text
zsh: no matches found: /data/*.out
```

可改为（由 **容器内** shell 展开路径）：

```bash
docker exec slurmctld sh -c 'ls /data/*.out'
```

### 5.10 `squeue` 过滤用户

`squeue -u` 需要**用户名参数**。在容器里若为 `root`，可用：

```bash
squeue -u root
```

或直接使用 `**squeue**`。若写 `squeue -u $USER` 且 `**USER` 未设置**，会报 *option requires an argument*。

```bash
[root@slurmctld data]# squeue -u $USER
squeue: option requires an argument -- 'u'
Try "squeue --help" for more information
```

### 5.11 命令输出摘录：`make run-examples`（宿主机）与 `scontrol`（容器内）

以下输出来自 **giovtorres/slurm-docker-cluster** 练习环境（Slurm **25.11.4**）。其中 `**scontrol show partitions` / `show nodes` / `show config`** 三段与你在对话里粘贴的终端原文**逐行对齐**（含 `Configuration data as of 2026-04-21T06:19:30` 与节点 `CPULoad` / `FreeMem` 等当时值）。`make run-examples` 在 **Mac 仓库根目录**执行；`scontrol` 在 `**make shell`** 进容器后执行（或 `docker exec slurmctld …`）。再次执行同一命令时，时间戳与负载可能略有变化，属正常现象。

#### `make run-examples`（选择 `1` 跑全部示例，节选）

```text
megsun@MEGSUN-M-KHXH slurm-docker-cluster % make run-examples
./run_examples.sh
================================
Running Slurm Example Jobs
================================

[INFO] Copying example jobs to cluster...
Successfully copied 12.8kB to slurmctld:/data/examples/

Example jobs available:
array_job.sh
cpu_intensive.sh
gpu_test.sh
job_dependency.sh
memory_test.sh
multi_node.sh
multi_node_singularity.sh
simple_hostname.sh

Select mode:
  1) Run all examples
  2) Run specific example
  3) List examples only
Enter choice (1-3): 1

Submitting all example jobs...
[SUBMIT] array_job.sh
  ✓ Job ID: 4
[SUBMIT] cpu_intensive.sh
  ✓ Job ID: 5
[SUBMIT] gpu_test.sh
  ✓ Job ID: 6
[SUBMIT] job_dependency.sh
  ✓ Job ID: 9
[SUBMIT] memory_test.sh
  ✓ Job ID: 10
[SUBMIT] multi_node.sh
  ✓ Job ID: 11
[SUBMIT] multi_node_singularity.sh
  ✓ Job ID: 12
[SUBMIT] simple_hostname.sh
  ✓ Job ID: 13

Waiting for jobs to complete...
  Waiting... (      10 jobs still running)
  Waiting... (       8 jobs still running)
  ...

Job outputs:
--- array_test_4_4.out ---
Array job task 4 of array job 4
Running on: c2
...

================================
Done!

Tip: Use make jobs to view job queue
Tip: Use docker exec slurmctld squeue to check job status
Tip: docker exec slurmctld sh -c 'ls /data/*.out'   # 在 zsh 下避免本机展开 glob
```

> 终端若开启颜色，脚本可能输出 ANSI 转义序列；上表为便于阅读已去掉颜色。实际脚本会先拷贝示例到容器的 `/data/examples/` 再 `sbatch`。

#### `scontrol show config`（完整）

```bash
[root@slurmctld data]# scontrol show config
Configuration data as of 2026-04-21T06:19:30
AccountingStorageBackupHost = (null)
AccountingStorageEnforce = none
AccountingStorageHost   = slurmdbd
AccountingStorageExternalHost = (null)
AccountingStorageParameters = (null)
AccountingStoragePort   = 6819
AccountingStorageTRES   = cpu,mem,energy,node,billing,fs/disk,vmem,pages
AccountingStorageType   = accounting_storage/slurmdbd
AccountingStoreFlags    = (null)
AcctGatherEnergyType    = (null)
AcctGatherFilesystemType = (null)
AcctGatherInterconnectType = (null)
AcctGatherNodeFreq      = 0 sec
AcctGatherProfileType   = (null)
AllowSpecResourcesUsage = no
AuthAltTypes            = auth/jwt
AuthAltParameters       = jwt_key=/etc/slurm/jwt_hs256.key
AuthInfo                = (null)
AuthType                = auth/munge
BatchStartTimeout       = 10 sec
BcastExclude            = /lib,/usr/lib,/lib64,/usr/lib64
BcastParameters         = (null)
BOOT_TIME               = 2026-04-21T00:47:40
BurstBufferType         = (null)
CertgenParameters       = (null)
CertgenType             = (null)
CertmgrParameters       = (null)
CertmgrType             = (null)
CliFilterParameters     = (null)
CliFilterPlugins        = (null)
ClusterName             = linux
CommunicationParameters = (null)
CompleteWait            = 0 sec
CpuFreqDef              = Unknown
CpuFreqGovernors        = OnDemand,Performance,UserSpace
CredType                = cred/munge
DataParserParameters    = (null)
DebugFlags              = (null)
DefMemPerNode           = UNLIMITED
DependencyParameters    = (null)
DisableRootJobs         = no
EioTimeout              = 60
EnforcePartLimits       = NO
EpilogMsgTime           = 2000 usec
FairShareDampeningFactor = 1
FederationParameters    = (null)
FirstJobId              = 1
GresTypes               = gpu
GpuFreqDef              = (null)
GroupUpdateForce        = 1
GroupUpdateTime         = 600 sec
HASH_VAL                = Match
HashPlugin              = hash/k12
HealthCheckInterval     = 0 sec
HealthCheckNodeState    = ANY
HealthCheckProgram      = (null)
HttpParserType          = http_parser/libhttp_parser
InactiveLimit           = 0 sec
InteractiveStepOptions  = --interactive --preserve-env --pty $SHELL
JobAcctGatherFrequency  = 30
JobAcctGatherType       = (null)
JobAcctGatherParams     = (null)
JobCompHost             = localhost
JobCompLoc              = /var/log/slurm/jobcomp.log
JobCompParams           = (null)
JobCompPass             = (null)
JobCompPort             = 0
JobCompType             = jobcomp/filetxt
JobCompUser             = root
JobDefaults             = (null)
JobFileAppend           = 0
JobRequeue              = 1
JobSubmitPlugins        = lua
KillOnBadExit           = 0
KillWait                = 30 sec
LaunchParameters        = (null)
Licenses                = (null)
LogTimeFormat           = iso8601_ms
MailDomain              = (null)
MailProg                = /bin/mail
MaxArraySize            = 1001
MaxBatchRequeue         = 5
MaxDBDMsgs              = 20400
MaxJobCount             = 10000
MaxJobId                = 67043328
MaxMemPerNode           = UNLIMITED
MaxNodeCount            = 100
MaxStepCount            = 40000
MaxTasksPerNode         = 512
MCSPlugin               = (null)
MCSParameters           = (null)
MessageTimeout          = 10 sec
MetricsType             = (null)
MinJobAge               = 300 sec
MpiDefault              = (null)
MpiParams               = (null)
NamespaceType           = (null)
NEXT_JOB_ID             = 26
NodeFeaturesPlugins     = (null)
OverTimeLimit           = 0 min
PluginDir               = /usr/lib64/slurm
PlugStackConfig         = (null)
PreemptMode             = OFF
PreemptParameters       = (null)
PreemptType             = (null)
PreemptExemptTime       = 00:00:00
PrEpParameters          = (null)
PrEpPlugins             = prep/script
PriorityParameters      = (null)
PrioritySiteFactorParameters = (null)
PrioritySiteFactorPlugin = (null)
PriorityDecayHalfLife   = 7-00:00:00
PriorityCalcPeriod      = 00:05:00
PriorityFavorSmall      = no
PriorityFlags           = 
PriorityMaxAge          = 7-00:00:00
PriorityType            = priority/multifactor
PriorityUsageResetPeriod = NONE
PriorityWeightAge       = 0
PriorityWeightAssoc     = 0
PriorityWeightFairShare = 0
PriorityWeightJobSize   = 0
PriorityWeightPartition = 0
PriorityWeightQOS       = 0
PriorityWeightTRES      = (null)
PrivateData             = none
ProctrackType           = proctrack/linuxproc
PrologEpilogTimeout     = 65534
PrologFlags             = (null)
PropagatePrioProcess    = 0
PropagateResourceLimits = ALL
PropagateResourceLimitsExcept = (null)
RebootProgram           = (null)
ReconfigFlags           = (null)
RequeueExit             = (null)
RequeueExitHold         = (null)
ResumeFailProgram       = (null)
ResumeProgram           = (null)
ResumeRate              = 300 nodes/min
ResumeTimeout           = 60 sec
ResvEpilog              = (null)
ResvOverRun             = 0 min
ResvProlog              = (null)
ReturnToService         = 1
SchedulerParameters     = (null)
SchedulerTimeSlice      = 30 sec
SchedulerType           = sched/backfill
ScronParameters         = (null)
SelectType              = select/cons_tres
SelectTypeParameters    = CR_CORE_MEMORY
SlurmUser               = slurm(990)
SlurmctldAddr           = (null)
SlurmctldDebug          = debug2
SlurmctldHost[0]        = slurmctld
SlurmctldLogFile        = /var/log/slurm/slurmctld.log
SlurmctldPort           = 6817
SlurmctldSyslogDebug    = (null)
SlurmctldPrimaryOffProg = (null)
SlurmctldPrimaryOnProg  = (null)
SlurmctldTimeout        = 120 sec
SlurmctldParameters     = (null)
SlurmdDebug             = info
SlurmdLogFile           = /var/log/slurm/slurmd.log
SlurmdParameters        = config_overrides
SlurmdPidFile           = /var/run/slurm/slurmd.pid
SlurmdPort              = 6818
SlurmdSpoolDir          = /var/spool/slurm
SlurmdSyslogDebug       = (null)
SlurmdTimeout           = 300 sec
SlurmdUser              = root(0)
SlurmSchedLogFile       = (null)
SlurmSchedLogLevel      = 0
SlurmctldPidFile        = /var/run/slurm/slurmctld.pid
SLURM_CONF              = /etc/slurm/slurm.conf
SLURM_VERSION           = 25.11.4
SrunEpilog              = (null)
SrunPortRange           = 0-0
SrunProlog              = (null)
StateSaveLocation       = /var/lib/slurm
SuspendExcNodes         = (null)
SuspendExcParts         = (null)
SuspendExcStates        = (null)
SuspendProgram          = (null)
SuspendRate             = 60 nodes/min
SuspendTime             = INFINITE
SuspendTimeout          = 30 sec
SwitchParameters        = (null)
SwitchType              = (null)
TaskEpilog              = (null)
TaskPlugin              = task/affinity
TaskPluginParam         = (null type)
TaskProlog              = (null)
TCPTimeout              = 2 sec
TLSParameters           = (null)
TLSType                 = tls/none
TmpFS                   = /tmp
TopologyParam           = (null)
TopologyPlugin          = topology/flat
TrackWCKey              = no
TreeWidth               = 16
UsePam                  = no
UnkillableStepProgram   = (null)
UnkillableStepTimeout   = 60 sec
UrlParserType           = url_parser/libhttp_parser
VSizeFactor             = 0 percent
WaitTime                = 0 sec
X11Parameters           = (null)

Slurmctld(primary) at slurmctld is UP
```

#### `scontrol show partitions`（完整）

```bash
[root@slurmctld data]# scontrol show partitions
PartitionName=cpu
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=YES QoS=N/A
   DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO ExclusiveTopo=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=UNLIMITED MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED MaxCPUsPerSocket=UNLIMITED
   NodeSets=cpu_nodes
   Nodes=c[1-2]
   PriorityJobFactor=1 PriorityTier=1 RootOnly=NO ReqResv=NO OverSubscribe=NO
   OverTimeLimit=NONE PreemptMode=OFF
   State=UP TotalCPUs=24 TotalNodes=2 SelectTypeParameters=NONE
   JobDefaults=(null)
   DefMemPerNode=UNLIMITED MaxMemPerNode=UNLIMITED
   TRES=cpu=24,mem=15674M,node=2,billing=24
PartitionName=gpu
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=NO QoS=N/A
   DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO ExclusiveTopo=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=UNLIMITED MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED MaxCPUsPerSocket=UNLIMITED
   NodeSets=gpu_nodes
   Nodes=(null)
   PriorityJobFactor=1 PriorityTier=1 RootOnly=NO ReqResv=NO OverSubscribe=NO
   OverTimeLimit=NONE PreemptMode=OFF
   State=UP TotalCPUs=0 TotalNodes=0 SelectTypeParameters=NONE
   JobDefaults=(null)
   DefMemPerNode=UNLIMITED MaxMemPerNode=UNLIMITED
   TRES=(null)
```

#### `scontrol show nodes`（完整，当前为 c1、c2）

```bash
[root@slurmctld data]# scontrol show nodes
NodeName=c1 Arch=aarch64 CoresPerSocket=12 
   CPUAlloc=0 CPUEfctv=12 CPUTot=12 CPULoad=0.26
   AvailableFeatures=cpu
   ActiveFeatures=cpu
   Gres=(null)
   NodeAddr=172.20.0.7 NodeHostName=c1 Version=25.11.4
   OS=Linux 6.10.14-linuxkit #1 SMP Thu Aug 14 19:26:13 UTC 2025 
   RealMemory=7837 AllocMem=0 FreeMem=404 Sockets=1 Boards=1
   State=IDLE+DYNAMIC_NORM ThreadsPerCore=1 TmpDisk=0 Weight=1 Owner=N/A MCS_label=N/A
   Partitions=cpu 
   BootTime=2026-04-18T16:55:46 SlurmdStartTime=2026-04-20T11:38:57
   LastBusyTime=2026-04-21T06:06:22 ResumeAfterTime=None
   CfgTRES=cpu=12,mem=7837M,billing=12
   AllocTRES=
   CurrentWatts=0 AveWatts=0
NodeName=c2 Arch=aarch64 CoresPerSocket=12 
   CPUAlloc=0 CPUEfctv=12 CPUTot=12 CPULoad=0.26
   AvailableFeatures=cpu
   ActiveFeatures=cpu
   Gres=(null)
   NodeAddr=172.20.0.5 NodeHostName=c2 Version=25.11.4
   OS=Linux 6.10.14-linuxkit #1 SMP Thu Aug 14 19:26:13 UTC 2025 
   RealMemory=7837 AllocMem=0 FreeMem=404 Sockets=1 Boards=1
   State=IDLE+DYNAMIC_NORM ThreadsPerCore=1 TmpDisk=0 Weight=1 Owner=N/A MCS_label=N/A
   Partitions=cpu 
   BootTime=2026-04-18T16:55:46 SlurmdStartTime=2026-04-20T11:38:57
   LastBusyTime=2026-04-21T06:04:28 ResumeAfterTime=None
   CfgTRES=cpu=12,mem=7837M,billing=12
   AllocTRES=
   CurrentWatts=0 AveWatts=0
```

#### `scontrol show nodes c1`（完整）

```bash
[root@slurmctld data]# scontrol show nodes c1
NodeName=c1 Arch=aarch64 CoresPerSocket=12 
   CPUAlloc=0 CPUEfctv=12 CPUTot=12 CPULoad=0.26
   AvailableFeatures=cpu
   ActiveFeatures=cpu
   Gres=(null)
   NodeAddr=172.20.0.7 NodeHostName=c1 Version=25.11.4
   OS=Linux 6.10.14-linuxkit #1 SMP Thu Aug 14 19:26:13 UTC 2025 
   RealMemory=7837 AllocMem=0 FreeMem=404 Sockets=1 Boards=1
   State=IDLE+DYNAMIC_NORM ThreadsPerCore=1 TmpDisk=0 Weight=1 Owner=N/A MCS_label=N/A
   Partitions=cpu 
   BootTime=2026-04-18T16:55:46 SlurmdStartTime=2026-04-20T11:38:57
   LastBusyTime=2026-04-21T06:06:22 ResumeAfterTime=None
   CfgTRES=cpu=12,mem=7837M,billing=12
   AllocTRES=
   CurrentWatts=0 AveWatts=0
```

### 5.12 与官方手册笔记的对应关系

本节（**5.1–5.10**）与 **5.11** 中的 `**sinfo` / `squeue` / `sbatch` / `srun` / `sacct` / `scontrol`** 与上文「常用命令」一致；实践里额外强化了：**Docker 运维在宿主机、Slurm 用户在容器内、批作业资源与步内 `srun` 的关系、分区与 GPU 占位作业**，**5.11** 则保留可直接对照的完整终端输出，便于以后对齐真实集群行为。