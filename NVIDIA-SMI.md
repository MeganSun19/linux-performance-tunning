***

* NVDIA SMI system management interface 

NVIDIA 系统管理接口 (nvidia-smi) 是一个基于 NVIDIA 管理库 (NVML) 的命令行实用程序，旨在帮助管理和监控 NVIDIA GPU 设备。
如果想用python写代码监控GPU,可以搜索pynvml 库，和nvida-smi获取的数据是一样的。
SM: Streaming multiprocessor: 

query:

!nvidia-smi --query-gpu=timestamp,name,utilization.gpu,memory.used --format=csv,noheader
2026/04/02 01:52:51.628, Tesla T4, 0 %, 0 MiB
