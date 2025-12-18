# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

ixy 是一个用户态网络驱动框架，仅用约 1000 行 C 代码实现了完整的网络驱动程序。它直接控制网卡硬件，不依赖内核模块（除非使用 VFIO）。项目主要用于教育目的，帮助理解网卡驱动的底层工作原理。

支持的网卡：
- Intel 82599ES (X520) 系列 - ixgbe 驱动
- Intel X540, X550 - ixgbe 驱动
- VirtIO 虚拟网卡 - virtio 驱动

## 常用命令

### 编译项目
```bash
cmake .
make
```

### 配置 Hugepages（必需）
```bash
sudo ./setup-hugetlbfs.sh
```

### 运行示例程序
所有程序都需要完整的 PCIe 总线地址（使用 `lspci` 查询）：

```bash
# 包生成器 - 在单个端口上发送数据包
sudo ./ixy-pktgen 0000:03:00.0

# 包转发器 - 在两个端口间转发数据包
sudo ./ixy-fwd 0000:03:00.0 0000:03:00.1

# 包捕获器 - 捕获数据包到 pcap 文件
sudo ./ixy-pcap 0000:03:00.0
```

### 使用 VFIO/IOMMU（可选，允许无 root 权限运行）
```bash
# 1. 启用 IOMMU（BIOS 中启用 VT-d，grub 中添加 intel_iommu=on）
# 2. 解绑 ixgbe 驱动
echo 0000:05:00.0 > /sys/bus/pci/devices/0000:05:00.0/driver/unbind

# 3. 加载 vfio-pci 驱动
modprobe vfio-pci

# 4. 绑定设备到 vfio-pci（使用 lspci -nn 查询 vendor_id 和 device_id）
echo 8086 1528 > /sys/bus/pci/drivers/vfio-pci/new_id

# 5. 修改设备权限
chown $USER:$GROUP /dev/vfio/*
```

## 代码架构

### 核心抽象层次

ixy 使用三层抽象结构：

1. **应用层** (`src/app/`) - 示例应用程序
2. **通用设备接口** (`src/driver/device.h`) - 驱动无关的抽象
3. **驱动实现** (`src/driver/ixgbe.c`, `src/driver/virtio.c`) - 硬件特定实现

### 核心组件

#### 设备抽象 (`src/driver/device.h`)
- `struct ixy_device` - 所有网卡驱动的基础结构
- 使用函数指针实现多态：`rx_batch`, `tx_batch`, `read_stats`, `set_promisc` 等
- `container_of` 宏用于从通用设备结构转换到驱动特定结构（如 `struct ixgbe_device`）
- 内联函数提供寄存器访问抽象：`set_reg32()`, `get_reg32()`, `wait_clear_reg32()` 等

#### 内存管理 (`src/memory.h`, `src/memory.c`)
- **Hugepage 分配**：使用 2MB 大页面提升性能，减少 TLB miss
- **Mempool**：简单的栈式内存池，管理数据包缓冲区
  - 当前实现是单线程的（非线程安全）
  - `struct pkt_buf` 包含物理地址 (`buf_addr_phy`)，用于 DMA 传输
- **DMA 内存**：为网卡队列描述符分配物理连续内存
- 数据包缓冲区结构：64 字节对齐，包含 40 字节头部空间

#### 驱动初始化流程
1. **PCI 设备发现** (`src/pci.c`) - 扫描 PCIe 总线，读取配置空间
2. **内存映射** - 映射 BAR（Base Address Register）到用户空间
3. **设备重置和初始化** - 驱动特定的初始化序列
4. **队列设置** - 配置 RX/TX 描述符环形队列
5. **中断配置**（仅 VFIO 模式）- 设置 MSI-X 中断

#### RX/TX 路径

**接收路径**：
- 描述符环形队列（默认 512 条目）
- `ixgbe_rx_batch()` - 批量接收数据包，减少 PCIe 事务开销
- 从描述符读取 DMA 完成状态，将物理地址映射回 `pkt_buf`

**发送路径**：
- 描述符环形队列（默认 512 条目）
- `ixgbe_tx_batch()` - 批量发送数据包
- 异步发送：数据包在 NIC 完成传输前不能释放
- `clean_index` 追踪已完成的传输，定期清理释放的缓冲区

#### VFIO/IOMMU 支持 (`src/libixy-vfio.c`)
- VFIO 提供用户态 DMA 访问和中断支持
- 通过 IOMMU 进行地址转换，支持非 root 用户运行
- 中断处理通过 eventfd 机制实现

### 关键设计决策

1. **批量处理**：所有 RX/TX 操作都是批量的（通常 32-64 个包），提升性能
2. **零拷贝**：数据包直接在 DMA 内存中分配，避免内存复制
3. **轮询模式**：默认使用忙等待而非中断，降低延迟
4. **简单内存管理**：栈式 mempool，牺牲线程安全换取简单性
5. **直接寄存器访问**：使用内存屏障而非复杂的同步机制

### 寄存器访问和内存屏障

代码使用编译器屏障 (`__asm__ volatile ("" : : : "memory")`) 而非完整的内存屏障：
- x86 架构的内存模型较强，MMIO 访问自然有序
- DPDK 采用相同方法
- `volatile` 关键字防止编译器优化掉寄存器访问

### 示例应用程序逻辑

**ixy-fwd**：双向包转发
- 从两个端口接收数据包
- 触碰数据包内容（`buf->data[1]++`）以模拟真实负载
- 在端口间交叉转发
- 未发送的包直接丢弃（避免延迟累积）

**ixy-pktgen**：包生成器
- 预分配数据包缓冲区并填充模板数据
- 发送 UDP 包（10.0.0.1 -> 10.0.0.2）
- 每个包包含递增的序列号
- 使用 `ixy_tx_batch_busy_wait()` 忙等待直到所有包排队

## 开发注意事项

### 安全警告
- **NIC 具有完整的 DMA 内存访问权限**：错误配置可能导致内存损坏或系统崩溃
- **设备解绑**：运行时会自动解绑 PCIe 设备的现有驱动，设备将从系统消失
- 不要在重要系统上修改驱动代码

### 数据手册参考
代码注释频繁引用数据手册章节：
- Intel 82599 数据手册（Revision 3.3, March 2016）
- VirtIO 规范 v1.0

阅读代码时应同时查阅相关数据手册。

### 性能考虑
- 单核心可转发 > 2500 万包/秒（3.0 GHz CPU）
- 避免频繁的时间查询（使用计数器间隔采样：`counter++ & 0xFFF`）
- 批量操作至关重要
- 某些系统上 CPU 频率过高可能导致性能下降（尝试降频或应用双向流量）

### 代码风格
- C11 标准
- 编译选项：`-O2 -march=native -std=c11`
- 简洁明了优先于抽象
- 内联函数用于性能关键路径
- `ixgbe_type.h` 是从 Intel 驱动复制的寄存器定义，可视为机器可读的数据手册

### NUMA 支持
- 当前需要通过外部 `numactl` 工具处理 NUMA 亲和性
- PCIe 设备绑定到特定 CPU，DMA 内存应固定到对应 NUMA 节点
- 处理包接收的线程也应绑定到同一 NUMA 节点
