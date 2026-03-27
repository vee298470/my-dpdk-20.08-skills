---
name: dpdk-20.08
description: DPDK 20.08 资深开发专家 - 深入掌握 EAL 初始化、mempool/mbuf 内存架构、PMD 描述符环、primary/secondary 多进程、Service Core 卸载、QoS 调度、L3/L4 应用开发，提供性能分析与问题排查支持

---


# DPDK 20.08 开发专家技能

## 角色定位

你是一名 DPDK 20.08 开发专家，专注于：
1. **高性能数据包处理** - 零拷贝、批量处理、无锁编程
2. **内存管理深度分析** - mbuf/mempool 内部结构、大页内存优化
3. **驱动/PMD 层面调试** - 硬件队列、描述符环、DMA 映射
4. **问题排查** - 大页溢出、内存泄漏、性能瓶颈定位


## 核心知识领域

### 1. 内存管理架构

#### 1.1 大页内存 (Hugepages)

```
物理内存布局:
┌──────────────────────────────────────────────────────────────┐
│  Kernel Space                                                │
├──────────────────────────────────────────────────────────────┤
│  Normal Pages (4KB)                                          │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                             │
│  │Page1│ │Page2│ │Page3│ │Page4│ ...                         │
│  └─────┘ └─────┘ └─────┘ └─────┘                             │
├──────────────────────────────────────────────────────────────┤
│  Hugepages (2MB/1GB) - DPDK 使用                             │
│  ┌──────────────────────┐ ┌──────────────────────┐           │
│  │   2MB Hugepage #0    │ │   2MB Hugepage #1    │           │
│  │  ┌────────┬────────┐ │ │  ┌────────┬────────┐ │           │
│  │  │Mempool │ Mbufs  │ │ │  │Mempool │ Mbufs  │ │           │
│  │  │ Pool   │        │ │ │  │ Pool   │        │ │           │
│  │  └────────┴────────┘ │ │  └────────┴────────┘ │           │
│  └──────────────────────┘ └──────────────────────┘           │
└──────────────────────────────────────────────────────────────┘
```

**关键配置:**
```bash
# 查看大页状态
cat /proc/meminfo | grep Huge
ls -la /dev/hugepages/

# 配置大页 (永久)
echo "vm.nr_hugepages = 2048" >> /etc/sysctl.conf

# 配置大页 (临时)
echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# mount 大页
mount -t hugetlbfs nodev /mnt/huge
```

**DPDK EAL 参数:**
```bash
# 指定 socket 内存
--socket-mem 2048,2048

# 指定大页文件目录
--huge-dir=/mnt/huge

# 限制最大内存
-m 4096
```

#### 1.2 mempool 内部结构

```c
struct rte_mempool {
    // ========== 只读数据 (热数据，单独缓存行) ==========
    MARKER cacheline0;

    char name[RTE_MEMPOOL_NAMESIZE];  // 内存池名称
    int flags;                         // 内存池标志
    uint32_t size;                     // 环的大小
    uint32_t cache_size;               // 每核缓存大小
    uint32_t cache_flushthresh;        // 缓存刷新阈值

    // ========== 内存管理数据 ==========
    struct rte_mempool_memhdr_list mem_list;  // 内存块链表
    uint32_t nb_mem_chunks;            // 内存块数量

    // ========== 环 (存储空闲对象) ==========
    struct rte_ring *ring;

    // ========== 每核缓存 ==========
    struct rte_mempool_cache *local_cache;

    // ========== 统计信息 (调试用) ==========
    #ifdef RTE_LIBRTE_MEMPOOL_DEBUG
    struct rte_mempool_debug_stats stats[RTE_MAX_LCORE];
    #endif
};
```

**mempool 内存布局 (宏观视角 - 管理体系):**
```
┌────────────────────────────────────────────────────────────────┐
│                         mempool                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  rte_mempool (控制结构)                                   │  │
│  │  - name, size, elt_size, ring 指针等                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  rte_ring (空闲列表 - 存储对象指针)                         │  │
│  │  ┌────────┬────────┬────────┬────────┐                  │  │
│  │  │0x1000  │0x1100  │0x1200  │ ...    │ 指向内存块中的    │  │
│  │  └────────┴────────┴────────┴────────┘ 对象              │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  内存块 (2MB 大页 - 实际存储对象的地方)                     │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ obj header | obj data (elt_size) | trailer         │  │  │
│  │  │ (16B→64B)  | (用户定义，如 mbuf=2304B) | (0/16B)    │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │ obj header | obj data (elt_size) | trailer         │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │  ... 重复 (每个对象大小 = header+elt_size+trailer)        │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

注意：obj data 的内容取决于 mempool 用途
- 通用 mempool: obj data = 用户定义的任意结构
- mbuf mempool: obj data = mbuf 结构 (128B) + headroom(128B) + data(2048B)
```

**mbuf 在 mempool 中的布局 (微观视角 - 对象内部):**
```
通用 mempool 对象结构:
┌──────────────┬─────────────────────────────────┬──────────────┐
│ obj header   │        obj data                 │ obj trailer  │
│ (16B 对齐 64B) │  (用户传入的 elt_size)           │  (0 或 16B)   │
└──────────────┴─────────────────────────────────┴──────────────┘

展开为 mbuf mempool (elt_size = 2304 字节):
┌──────────────┬──────────────────────────────────────────────────┐
│ obj header   │            obj data (2304B)                      │
│ (64B)        │ ┌──────────┬──────────┬─────────────────────┐   │
│              │ │ mbuf 结构 │ headroom │ data                │   │
│              │ │ (128B)   │ (128B)   │ (2048B)             │   │
│              │ └──────────┴──────────┴─────────────────────┘   │
└──────────────┴──────────────────────────────────────────────────┘

完整内存块布局 (mbuf pool):
┌────────────────────────────────────────────────────────────────┐
│ 内存块 (2MB 连续物理内存)                                        │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ mbuf 对象 #0                                                 │ │
│ │ [obj hdr:64B][mbuf:128B][headroom:128B][data:2048B]        │ │
│ │                         ↑                                   │ │
│ │                         └── buf_addr 指向这里               │ │
│ └────────────────────────────────────────────────────────────┘ │
│ ┌────────────────────────────────────────────────────────────┐ │
│ │ mbuf 对象 #1 (相同结构，间隔 = header+elt_size+trailer)      │ │
│ └────────────────────────────────────────────────────────────┘ │
│ ... 重复                                                       │
└────────────────────────────────────────────────────────────────┘

计算示例 (2MB 大页):
- obj header:    64 字节 (对齐后)
- elt_size:      2304 字节 (128 mbuf + 128 headroom + 2048 data)
- obj trailer:   0 字节 (非 debug 模式)
- 分散对齐填充： 约 64 字节 (MEMPOOL_F_NO_SPREAD 未设置时)
- 总对象大小：   约 2368 字节

- 2MB 大页 = 2,097,152 字节
- 每页 mbuf 数量 = 2,097,152 / 2368 ≈ 885 个
```

#### 1.3 mbuf 结构深度分析

```c
// DPDK 20.08 mbuf 核心结构 (64 字节缓存行对齐)
struct rte_mbuf {
    // ========== Cacheline 0 (RX 路径热数据) ==========
    MARKER cacheline0;

    // 地址和长度 (16 字节)
    RTE_STD_C11
    union {
        rte_iova_t buf_iova;      // IO 虚拟地址 (DMA 用)
        phys_addr_t buf_physaddr; // 物理地址 (已废弃)
    };
    void *buf_addr;               // 虚拟地址
    uint16_t data_len;            // 数据长度
    uint16_t pkt_len;             // 包总长度 (多段时)

    // 状态和引用 (8 字节)
    uint16_t refcnt;              // 引用计数
    uint16_t nb_segs;             // 分段数量

    // RX offload 标记 (8 字节)
    union {
        uint64_t ol_flags;        // offload 标志
        struct {
            uint32_t packet_type; // 包类型 (RSS 哈希)
            // ... RX 特定标志
        };
    };

    // 数据偏移 (2 字节)
    uint16_t data_off;            // 数据起始偏移

    // ========== Cacheline 1 (TX 路径数据) ==========
    MARKER cacheline1;

    // 私有数据大小
    uint16_t priv_size;

    // 发送相关
    uint16_t timesync;            // PTP 时间戳
    uint32_t tx_offload;          // TX offload 参数

    // 队列信息
    uint16_t port;                // 端口 ID
    union {
        uint16_t vlan_tci;        // VLAN 标签
        uint16_t vlan_tci_outer;  // 外层 VLAN
    };

    // 哈希/队列
    union {
        uint32_t rss;             // RSS 哈希值
        uint32_t fdir.hi;         // Flow Director
    };

    // 序列号和 ACL
    uint16_t seqn;                // 序列号
    uint16_t mbuf_dyn1;           // 动态字段

    // 指向数据包数据的指针
    void *userdata;               // 用户数据指针

    // ========== 数据包数据 (紧随结构之后) ==========
    // char data[] __rte_cache_aligned;
};

// rte_mbuf 结构体内存布局 (128 字节):
// ┌─────────────────────────────────────┐
// │ rte_mbuf (128 字节，2 个缓存行)        │
// │  cacheline0: 64 字节 (RX 热数据)        │
// │   - buf_iova, buf_addr              │
// │   - data_len, pkt_len               │
// │   - refcnt, nb_segs                 │
// │   - ol_flags, packet_type           │
// │   - data_off                        │
// │  cacheline1: 64 字节 (TX 数据)          │
// │   - priv_size, tx_offload           │
// │   - port, vlan_tci                  │
// │   - rss, seqn                       │
// └─────────────────────────────────────┘
//
// 注意：rte_mbuf 后面紧跟数据缓冲区
// 数据缓冲区由 headroom 和 dataroom 组成
```

**mbuf 数据缓冲区布局:**
```
┌──────────────────────────────────────────────────────────────────┐
│ 数据缓冲区 (buf_addr 指向的区域，共 2176 字节)                      │
├────────────────────────────┬─────────────────────────────────────┤
│  headroom (128B)           │  dataroom (2048B)                   │
│                            │                                     │
│  用于 prepend 操作            │  实际数据包存储区域                  │
│  (添加 VLAN/IPsec/隧道头)   │  data_off 初始指向这里              │
│                            │                                     │
│  data_off 初始值=128 ──────>│  ↓                                  │
└────────────────────────────┴─────────────────────────────────────┘

完整 mbuf 对象在 mempool 中的布局:
┌───────────┬────────────┬──────────────┬─────────────┬──────────┐
│obj header │mbuf 结构    │  headroom    │  dataroom   │ trailer  │
│ (64B)     │  (128B)    │   (128B)     │   (2048B)   │ (0/16B)  │
└───────────┴────────────┴──────────────┴─────────────┴──────────┘
      ↑                           ↑
   buf_addr                 实际数据起始位置
   指向这里                   (data_off 偏移处)

计算示例 (2MB 大页):
- obj header:      64 字节 (16 字节对齐到 64 字节)
- elt_size:        2304 字节 (128 mbuf + 128 headroom + 2048 dataroom)
- obj trailer:     0 字节 (非 debug 模式) 或 16 字节 (debug 模式)
- 分散对齐填充：   约 64 字节 (确保对象跨内存通道分散)
- 单对象总计：     约 2368 字节

- 2MB 大页 = 2,097,152 字节
- 每页 mbuf 数量 = 2,097,152 / 2368 ≈ 885 个
```

### 2. 数据包处理流程

#### 2.0 EAL 初始化流程

```
应用程序启动
    │
    ▼
rte_eal_init()
    │
    ├──► 1. 解析命令行参数 (--file-prefix, -l, -n, --socket-mem)
    │
    ├──► 2. 探测 CPU 和 NUMA 拓扑
    │    └─── rte_eal_cpu_init()
    │
    ├──► 3. 初始化大页内存
    │    ├─── rte_eal_hugepage_init()
    │    └─── rte_eal_memory_init()
    │
    ├──► 4. 设置 PCI/总线访问
    │    └─── rte_eal_pci_init()
    │
    ├──► 5. 启动 secondary 进程通信 (if needed)
    │    └─── rte_eal_mp_init()
    │
    ├──► 6. 初始化 timer 子系统
    │    └─── rte_eal_timer_init()
    │
    ├──► 7. 初始化 intr 中断子系统
    │    └─── rte_eal_intr_init()
    │
    ├──► 8. 探测 PCI 设备
    │    └─── rte_eal_pci_probe()
    │
    ├──► 9. 初始化 VDEV 虚拟设备
    │    └─── rte_eal_vdev_init()
    │
    ├──► 10. 初始化 ring/pdump/mempool 库
    │    └─── rte_mempool_init(), rte_ring_init()
    │
    ├──► 11. 启动 lcore 线程
    │    └─── rte_eal_thread_init()
    │
    ▼
返回 0 (成功) → 调用 main() 函数
```

**初始化代码示例:**
```c
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mempool.h>

int main(int argc, char *argv[]) {
    struct rte_mempool *mbuf_pool;
    int ret;

    // Step 1: EAL 初始化
    ret = rte_eal_init(argc, argv);
    if (ret < 0)
        rte_exit(EXIT_FAILURE, "EAL init failed\n");

    argc -= ret;
    argv += ret;

    // Step 2: 创建 mbuf 池
    mbuf_pool = rte_pktmbuf_pool_create("mbuf_pool",
        8192,           // nb_mbufs
        256,            // cache_size
        0,              // priv_size
        RTE_MBUF_DEFAULT_BUF_SIZE,
        rte_socket_id() // socket_id
    );
    if (mbuf_pool == NULL)
        rte_exit(EXIT_FAILURE, "Cannot create mbuf pool\n");

    // Step 3: 初始化端口
    uint16_t port_id = 0;
    struct rte_eth_conf port_conf = {0};
    ret = rte_eth_dev_configure(port_id, 1, 1, &port_conf);
    if (ret < 0)
        rte_exit(EXIT_FAILURE, "Cannot configure port\n");

    // Step 4: 启动 lcore
    rte_eal_mp_remote_launch(main_loop, NULL, SKIP_MASTER);

    return 0;
}
```

#### 2.1 RX 接收流程

```
NIC 硬件                              驱动程序                        应用层
  │                                     │                               │
  │ 1. DMA 写入数据包到 mbuf              │                               │
  │────────────────────────────────────>│                               │
  │                                     │                               │
  │ 2. 更新 RX 描述符 DD 位                │                               │
  │                                     │                               │
  │ 3. 触发中断/轮询                      │                               │
  │                                     │                               │
  │                          4. rte_eth_rx_burst()                      │
  │                                     │                               │
  │                          5. 检查 DD 位，确认数据包就绪                 │
  │                          6. 更新 mbuf 元数据 (pkt_len, ol_flags)      │
  │                          7. 将 mbuf 指针加入 tx_burst 数组            │
  │                                     │                               │
  │                                     │ 8. 返回 nb_pkts               │
  │                                     │<──────────────────────────────│
  │                                     │                               │
```

#### 2.2 中断处理流程 (UIO/VFIO)

```
用户空间 DPDK 应用                    内核驱动 (uio_pci_generic/vfio-pci)
       │                                         │
       │ 1. 打开 /dev/uioX 或 /dev/vfio/X        │
       │<────────────────────────────────────────│
       │                                         │
       │ 2. mmap 中断事件fd                       │
       │<────────────────────────────────────────│
       │                                         │
       │ 3. rte_intr_enable()                     │
       │    - epoll_create()                     │
       │    - 注册中断 fd 到轮询                  │
       │                                         │
       │                                         │ ◄─── 4. 硬件中断
       │                                         │
       │                                         │ 5. 内核中断处理程序
       │                                         │    - clear_interrupt_status()
       │                                         │
       │ 6. epoll_wait() 返回                     │
       │    (事件 fd 可读)                        │
       │                                         │
       │ 7. rte_eth_rx_burst() 处理数据包         │
       │                                         │
       │ 8. rte_intr_enable() 重新使能中断        │
       │                                         │
```

**中断模式代码示例:**
```c
// 中断模式 RX 处理
static volatile int force_quit;

static void
signal_handler(int signum)
{
    if (signum == SIGINT || signum == SIGTERM) {
        printf("\n\nSignal %d received, preparing to exit...\n", signum);
        force_quit = 1;
    }
}

static int
lcore_interrupt_loop(__attribute__((unused)) void *arg)
{
    uint16_t portid, queueid;
    struct rte_mbuf *bufs[BURST_SIZE];
    struct rte_eth_dev *dev;
    int ret;

    queueid = rte_lcore_index(rte_lcore_id());

    RTE_ETH_FOREACH_DEV(portid) {
        dev = &rte_eth_devices[portid];

        // 注册中断
        rte_intr_enable(dev->intr_handle);

        while (!force_quit) {
            // 等待中断 (阻塞在 epoll_wait)
            ret = rte_intr_wait(dev->intr_handle, NULL);
            if (ret < 0) {
                if (ret == -EINTR)  // 被信号中断
                    continue;
                break;
            }

            // 中断触发，接收数据包
            uint16_t nb_rx = rte_eth_rx_burst(portid, queueid, bufs, BURST_SIZE);
            if (nb_rx > 0) {
                // 处理数据包...
                for (uint16_t i = 0; i < nb_rx; i++) {
                    process_packet(bufs[i]);
                }
                rte_pktmbuf_free_bulk(bufs, nb_rx);
            }

            // 重新使能中断
            rte_intr_enable(dev->intr_handle);
        }
    }

    return 0;
}
```

**中断 vs 轮询模式对比:**

| 特性 | 轮询模式 | 中断模式 |
|------|----------|----------|
| CPU 占用 | 高 (持续轮询) | 低 (事件驱动) |
| 延迟 | 极低 | 中等 |
| 吞吐量 | 高 | 中等 |
| 适用场景 | 高流量 (>10Gbps) | 低流量/突发流量 |
| 功耗 | 高 | 低 |

#### 2.3 mbuf 生命周期

```
创建                         使用                            销毁
  │                           │                               │
  │ rte_pktmbuf_alloc()       │                               │
  │<─────────┐                │                               │
  │          │                │                               │
  │          ▼                │                               │
  │  ┌───────────────┐        │                               │
  │  │ 从 mempool     │        │                               │
  │  │ 获取 mbuf      │        │                               │
  │  │ refcnt=1      │        │                               │
  │  └───────────────┘        │                               │
  │          │                │                               │
  │          ▼                │                               │
  │  ┌───────────────┐        │                               │
  │  │ 填充数据      │        │                               │
  │  │ data_len=X    │        │                               │
  │  └───────────────┘        │                               │
  │          │                │                               │
  │          ├────────────────│──────────────────────────────>│
  │          │                │ 发送                           │
  │          │                │ rte_eth_tx_burst()             │
  │          │                │                               │
  │          │                │ 拷贝/克隆                      │
  │          ├────────────────│──> rte_pktmbuf_clone()        │
  │          │                │     (增加 refcnt)              │
  │          │                │                               │
  │          ▼                │                               │
  │  ┌───────────────┐        │                               │
  │  │ rte_free()    │        │                               │
  │  │ refcnt--      │        │                               │
  │  └───────────────┘        │                               │
  │          │                │                               │
  │          ▼                │                               │
  │     [refcnt==0?]          │                               │
  │       │        │          │                               │
  │       是       否         │                               │
  │       │        │          │                               │
  │       ▼        └──────────┼──────────────────────────────>│
  │  ┌───────────────┐        │                               │
  │  │ 归还到        │        │                               │
  │  │ mempool       │        │                               │
  │  │ (加入 ring)   │        │                               │
  │  └───────────────┘        │                               │
  │                           │                               │
```

### 3. 问题排查手册

#### 3.1 大页溢出排查

**症状:**
```
EAL: Cannot allocate memory
EAL: Failed to allocate socket memory
```

**排查步骤:**

```bash
# Step 1: 检查当前大页使用情况
cat /proc/meminfo | grep Huge
# HugePages_Total:   2048
# HugePages_Free:    512    <- 剩余量
# HugePages_Rsvd:    1024   <- 已预留
# HugePages_Surp:    0      <- 超额预留

# Step 2: 检查 DPDK 进程使用
cat /sys/kernel/mm/hugepages/hugepages-2048kB/free_hugepages
cat /sys/kernel/mm/hugepages/hugepages-2048kB/resv_hugepages

# Step 3: 查看 DPDK 内存映射
cat /proc/<pid>/maps | grep huge
# 可以看到类似:
# 7f0000000000-7f0080000000 rw-p 00000000 00:19 1  /mnt/huge/rte_hugefile_...

# Step 4: 计算需求内存
# 每个端口每队列需要: nb_desc * mbuf_size
# 例如：4 端口 x 4 队列 x 2048 描述符 x 2304 字节 = 72MB

# Step 5: 增加大页
echo 4096 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

**DPDK 代码级排查:**
```c
// 检查 mempool 创建是否成功
struct rte_mempool *mp = rte_pktmbuf_pool_create(...);
if (mp == NULL) {
    // 获取详细错误
    printf("rte_errno=%d: %s\n", rte_errno, rte_strerror(rte_errno));

    // 常见错误码:
    // -ENOMEM: 内存不足
    // -EEXIST: 名称已存在
    // -EINVAL: 参数无效
}

// 检查大页是否足够
unsigned long socket_mem = rte_lcore_to_socket_id(lcore_id);
printf("Socket %d memory info:\n", socket_mem);

// 打印 mempool 统计
struct rte_mempool_info info;
rte_mempool_info(mp, &info);
printf("Pool size: %u, Free: %u, In-use: %u\n",
       info.size, info.free_count, info.size - info.free_count);
```

#### 3.2 内存泄漏排查

**排查工具:**

```bash
# 使用 dpdk-procinfo
./dpdk-procinfo --file-prefix=primary --show-mem

# 使用 testpmd 监控
testpmd> show mempool
testpmd> show mempool stats

# 定期采样
watch -n1 'cat /proc/<pid>/status | grep VmRSS'
```

**代码级排查:**

```c
// 1. 启用 mempool 调试
// 编译时添加: -DRTE_LIBRTE_MEMPOOL_DEBUG

// 2. 定期检查 mempool 使用量
void check_mempool_leak(struct rte_mempool *mp, const char *name) {
    static uint32_t last_free = 0;
    uint32_t free_count = rte_mempool_avail_count(mp);

    if (last_free > 0 && free_count < last_free - 100) {
        printf("WARNING: %s mempool may have leak! "
               "Free objects: %u (was %u)\n",
               name, free_count, last_free);
    }
    last_free = free_count;
}

// 3. 检查 mbuf 引用计数
void check_mbuf_refcnt(struct rte_mbuf *m) {
    if (rte_mbuf_refcnt_read(m) != 1) {
        RTE_LOG(WARNING, MBUF, "mbuf refcnt=%d, expected 1\n",
                rte_mbuf_refcnt_read(m));
    }
}

// 4. 使用 valgrind (仅用于开发环境)
valgrind --leak-check=full ./app --file-prefix=test
```

#### 3.3 性能瓶颈分析

**性能监控指标:**

```c
// RX/TX 效率
struct rte_eth_stats stats;
rte_eth_stats_get(port_id, &stats);

printf("RX: packets=%lu, bytes=%lu, errors=%lu, dropped=%lu\n",
       stats.ipackets, stats.ibytes, stats.ierrors, stats.idropped);
printf("TX: packets=%lu, bytes=%lu, errors=%lu\n",
       stats.opackets, stats.obytes, stats.oerrors);

// 计算丢包率
float drop_rate = (float)stats.idropped / (stats.ipackets + stats.idropped) * 100;
printf("Drop rate: %.2f%%\n", drop_rate);

// 计算吞吐量
float throughput_mpps = (stats.ipackets - last_ipackets) / 1000000.0;
printf("Throughput: %.2f Mpps\n", throughput_mpps);
```

**常见瓶颈点:**

| 瓶颈点 | 症状 | 解决方案 |
|--------|------|----------|
| RX 描述符耗尽 | `rte_eth_rx_burst` 返回 0 | 增加 `nb_rx_desc`，提高轮询频率 |
| TX 队列满 | `rte_eth_tx_burst` 返回 < nb_pkts | 增加 `nb_tx_desc`，实现 backpressure |
| mempool 耗尽 | 分配失败，丢包 | 增加 pool size，优化释放逻辑 |
| 缓存未命中 | CPU cycles 高，PPS 低 | 优化数据结构对齐，使用预取 |
| NUMA 跨访问 | 性能不均 | 确保 lcore/mempool/设备同 socket |

### 4. PMD 驱动层面

#### 4.1 RX 描述符环

```c
// RX 描述符结构 (以 Intel IXGBE 为例)
union ixgbe_adv_rx_desc {
    struct {
        __le64 pkt_addr;     // DMA 地址 (数据包位置)
        __le64 hdr_addr;     // 头部地址 (分离头部用)
    } read;
    struct {
        __le64 lo;
        __le64 hi;
    } wb; // write-back
};

// 驱动接收流程:
// 1. 硬件 DMA 写入数据包到 pkt_addr 指向的 mbuf
// 2. 硬件设置 DD (Done) 位
// 3. 驱动轮询 DD 位
// 4. 驱动读取描述符，更新 mbuf 元数据
// 5. 驱动分配新 mbuf，更新描述符

// RX 环状态检查:
void dump_rx_ring(uint16_t port_id, uint16_t queue_id) {
    struct rte_eth_dev *dev = &rte_eth_devices[port_id];
    struct igb_rx_queue *rxq = dev->data->rx_queues[queue_id];

    printf("RX Ring Info:\n");
    printf("  nb_desc: %u\n", rxq->nb_rx_desc);
    printf("  rx_tail: %u\n", rxq->rx_tail);
    printf("  nb_rx_hold: %u\n", rxq->nb_rx_hold);

    // 检查描述符
    for (int i = 0; i < 4; i++) {
        union ixgbe_adv_rx_desc *desc = IXGBE_PCI_OFFSET_ADDR(rxq->rx_ring) + i;
        printf("  Desc[%d]: DD=%d, pkt_addr=0x%lx\n",
               i, !!(desc->wb.upper.status_error & IXGBE_RXD_STAT_DD),
               desc->read.pkt_addr);
    }
}
```

#### 4.2 TX 描述符环

```c
// TX 描述符结构
union ixgbe_adv_tx_desc {
    struct {
        __le64 buffer_addr;  // 数据包地址
        __le32 cmd_type_len; // 命令/类型/长度
        __le32 olinfo_status; // offload 信息
    } read;
    struct {
        __le64 rsvd;
        __le32 nxtseqstat;
        __le32 status;
    } wb;
};

// TX 清理流程:
// 1. 检查描述符的 DD 位 (硬件已完成发送)
// 2. 释放已发送的 mbuf
// 3. 更新 tx_tail

void tx_ring_cleanup(struct igb_tx_queue *txq) {
    uint16_t i = txq->tx_head;
    while (i != txq->tx_tail) {
        union ixgbe_adv_tx_desc *desc = txq->tx_ring + i;

        // 检查 DD 位
        if (!(desc->wb.status & IXGBE_TXD_STAT_DD))
            break;  // 硬件未完成

        // 释放 mbuf
        struct rte_mbuf *m = txq->sw_ring[i].mbuf;
        rte_pktmbuf_free(m);
        txq->sw_ring[i].mbuf = NULL;

        i++;
        if (i == txq->nb_tx_desc)
            i = 0;
    }
    txq->tx_head = i;
}
```

### 5. 调试 API 和工具

#### 5.1 运行时调试

```c
// 打印 mbuf 信息
void dump_mbuf(struct rte_mbuf *m, const char *tag) {
    printf("=== Mbuf %s ===\n", tag);
    printf("  buf_iova:   0x%lx\n", (ulong)m->buf_iova);
    printf("  buf_addr:   %p\n", m->buf_addr);
    printf("  data_off:   %u\n", m->data_off);
    printf("  data_len:   %u\n", m->data_len);
    printf("  pkt_len:    %u\n", m->pkt_len);
    printf("  nb_segs:    %u\n", m->nb_segs);
    printf("  refcnt:     %u\n", rte_mbuf_refcnt_read(m));
    printf("  ol_flags:   0x%lx\n", m->ol_flags);
    printf("  packet_type: 0x%x\n", m->packet_type);

    // 打印 offload 标志
    char buf[256];
    rte_get_rx_ol_flag_list(m->ol_flags, buf, sizeof(buf));
    printf("  RX flags:   %s\n", buf);
}

// 打印 mempool 统计
void dump_mempool_stats(struct rte_mempool *mp) {
    printf("=== Mempool: %s ===\n", mp->name);
    printf("  size:        %u\n", mp->size);
    printf("  cache_size:  %u\n", mp->cache_size);
    printf("  avail_count: %u\n", rte_mempool_avail_count(mp));
    printf("  in_use_count:%u\n", rte_mempool_in_use_count(mp));

    // 打印每核缓存
    for (int i = 0; i < RTE_MAX_LCORE; i++) {
        struct rte_mempool_cache *cache = mp->local_cache + i;
        if (cache->len > 0) {
            printf("  Lcore %d cache: %u objects\n", i, cache->len);
        }
    }
}

// 打印端口统计
void dump_port_stats(uint16_t port_id) {
    struct rte_eth_stats stats;
    rte_eth_stats_get(port_id, &stats);

    printf("=== Port %u Statistics ===\n", port_id);
    printf("  RX packets:    %lu\n", stats.ipackets);
    printf("  RX bytes:      %lu\n", stats.ibytes);
    printf("  RX errors:     %lu\n", stats.ierrors);
    printf("  RX dropped:    %lu\n", stats.idropped);
    printf("  RX no_mbuf:    %lu\n", stats.rx_nombuf);
    printf("  TX packets:    %lu\n", stats.opackets);
    printf("  TX bytes:      %lu\n", stats.obytes);
    printf("  TX errors:     %lu\n", stats.oerrors);
}
```

#### 5.2 GDB 调试技巧

```bash
# 附加到 DPDK 进程
gdb -p <pid>

# 查看 mbuf 内容
(gdb) p *m
(gdb) p/x m->ol_flags
(gdb) p rte_get_rx_ol_flag_name(m->ol_flags)

# 查看 mempool
(gdb) p *mp
(gdb) p mp->ring->count

# 查看描述符环
(gdb) p rxq->rx_ring[0]@4
(gdb) p txq->tx_ring[0]@4

# 查看大页映射
(gdb) cat /proc/<pid>/maps | grep huge
```

### 6. 多进程架构 (Primary/Secondary)

#### 6.1 进程间通信 (RPC) 机制

```
┌───────────────────────────────────────────────────────────────────┐
│                          Primary 进程                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐   │
│  │   设备管理      │  │   内存管理      │  │   共享内存区        │   │
│  │  (创建端口)    │  │  (创建 pool)   │  │  - rte_config      │   │
│  └────────────────┘  └────────────────┘  │  - mempool/ring 指针│   │
│         │                     │          │  - eth_dev 结构    │   │
│         └──────────┬──────────┘          └────────────────────┘   │
│                    │                                              │
│                    ▼                                              │
│  ┌───────────────────────────────────────────────────────────────┐│
│  │           MP Channel (rte_mp_channel)                         ││
│  │   - /var/run/.rte/<prefix>/rte/mp_socket                      ││
│  └───────────────────────────────────────────────────────────────┘│
└───────────────────────────────────────────────────────────────────┘
                             ▲
                             │ unix socket 通信
                             ▼
┌───────────────────────────────────────────────────────────────────┐
│                         Secondary 进程                             │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐   │
│  │   应用逻辑      │  │   请求资源      │  │   映射共享内存      │   │
│  │  (包处理)      │  │  (通过 RPC)    │  │  (直接访问 pool/   │   │
│  └────────────────┘  └────────────────┘  │   ring)            │   │
│                                          └────────────────────┘   │
└───────────────────────────────────────────────────────────────────┘
```

**Primary 进程代码示例:**
```c
#include <rte_eal.h>
#include <rte_mp.h>

// 处理 secondary 进程的请求
static int
handle_resource_request(const struct rte_mp_msg *msg, const void *peer)
{
    printf("Received request from secondary: %s\n", msg->name);

    // 回复共享资源信息
    struct rte_mp_msg reply;
    memset(&reply, 0, sizeof(reply));
    strcpy(reply.name, "resource_reply");
    reply.num_fds = 0;

    // 发送 mempool/ring 的 fd 给 secondary
    // reply.fds[0] = rte_mempool_get_fd(mp);

    return rte_mp_reply(&reply, peer);
}

int main(int argc, char *argv[]) {
    // Primary 进程初始化
    rte_eal_init(argc, argv);

    // 注册回调处理 secondary 请求
    rte_mp_action_register("resource_req", handle_resource_request);

    // 创建共享资源
    struct rte_mempool *mp = rte_pktmbuf_pool_create("shared_pool", ...);
    struct rte_ring *ring = rte_ring_create("shared_ring", ...);

    // 主处理循环
    while (1) {
        // 处理数据包...
    }

    return 0;
}
```

**Secondary 进程代码示例:**
```c
#include <rte_eal.h>
#include <rte_mp.h>

static struct rte_mempool *shared_mp;
static struct rte_ring *shared_ring;

// 向 primary 请求资源
static int
request_resources_from_primary(void)
{
    struct rte_mp_msg msg, reply;
    memset(&msg, 0, sizeof(msg));
    strcpy(msg.name, "resource_req");

    // 发送请求
    int ret = rte_mp_request(&msg, &reply, 1);
    if (ret < 0) {
        printf("Failed to request resources\n");
        return -1;
    }

    // 从回复中获取 fd，映射共享内存
    // shared_mp = rte_mempool_lookup("shared_pool");
    // shared_ring = rte_ring_lookup("shared_ring");

    return 0;
}

int main(int argc, char *argv[]) {
    // Secondary 进程必须以 secondary 模式启动
    rte_eal_init(argc, argv);

    // 请求共享资源
    request_resources_from_primary();

    // 现在可以访问共享的 mempool/ring
    // 但不能创建新的端口或分配大页内存

    while (1) {
        // 从共享 ring 接收数据包并处理
        struct rte_mbuf *m;
        if (rte_ring_dequeue(shared_ring, (void **)&m) == 0) {
            process_packet(m);
            rte_pktmbuf_free(m);
        }
    }

    return 0;
}
```

**启动命令:**
```bash
# 启动 primary 进程
sudo ./app --file-prefix=app -l 0-7 -n 4 --socket-mem 2048,0 -p 0x3

# 启动 secondary 进程 (可以启动多个)
sudo ./app --file-prefix=app --secondary -l 8-15 -n 4
```

**使用场景:**
- **控制/数据平面分离**: primary 负责管理配置，secondary 负责包处理
- **热升级**: 启动新的 secondary 进程，逐步迁移流量，然后停止旧的
- **多应用共享 NIC**: 多个独立应用共享同一物理 NIC

### 7. Service Core 框架

DPDK 20.08 的 Service Core 框架允许将后台服务（如 lcore 仲裁、定时器、事件调度）卸载到专用核上。

```
┌───────────────────────────────────────────────────────────────────┐
│                       Data Cores (处理包)                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  Core 1  │  │  Core 2  │  │  Core 3  │  │  Core 4  │         │
│  │  RX/TX   │  │  RX/TX   │  │  RX/TX   │  │  RX/TX   │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
└───────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│                      Service Core (后台服务)                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Eventdev    │  │   Timer      │  │   Crypto      │           │
│  │  Scheduler   │  │   Library    │  │   Device      │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└───────────────────────────────────────────────────────────────────┘
```

**Service Core 配置示例:**
```c
#include <rte_service.h>
#include <rte_eventdev.h>

int main(int argc, char *argv[]) {
    rte_eal_init(argc, argv);

    // 1. 初始化 service 子系统
    rte_service_init();

    // 2. 创建 eventdev (作为 service 运行)
    struct rte_event_dev_config event_conf = {
        .nb_event_ports = 4,
        .nb_event_queues = 8,
        .nb_event_queue_flows = 1024,
    };
    uint8_t eventdev_id = rte_event_dev_attach("eventdev0", &event_conf);

    // 3. 获取 eventdev 需要的 service 数量
    uint32_t service_count;
    rte_event_dev_service_id_get(eventdev_id, &service_count);

    // 4. 映射 service 到专用 lcore
    uint32_t sid = service_ids[0];
    rte_service_map_lcore_set(sid, service_lcore_id, 1);

    // 5. 以 service 角色启动 lcore
    rte_service_lcore_add(service_lcore_id);
    rte_service_lcore_start(service_lcore_id);

    // 6. Data cores 启动处理循环
    rte_eal_mp_remote_launch(data_plane_loop, NULL, SKIP_MASTER);

    return 0;
}

// Data plane 使用 eventdev 调度
static int
data_plane_loop(void *arg) {
    uint8_t dev_id = rte_event_dev_get_dev_id("eventdev0");

    while (1) {
        struct rte_event ev;
        // 从 eventdev 获取事件 (由 service core 调度)
        if (rte_event_dequeue_burst(dev_id, port_id, &ev, 1, 0) > 0) {
            // 处理事件
            process_packet(ev.mbuf);
            // 返回事件到队列
            rte_event_enqueue_burst(dev_id, port_id, &ev, 1);
        }
    }
}
```

### 8. QoS 和 Traffic Manager

#### 8.1 Metering (三色标记器)

```c
#include <rte_meter.h>

// srTCM (单速率三色标记器 - RFC 2697)
struct rte_meter_srtrcm {
    uint32_t cbs;  // 承诺桶大小 (bytes)
    uint32_t ebs;  // 超额桶大小 (bytes)
    uint32_t c_tb; // 承诺桶当前值
    uint32_t e_tb; // 超额桶当前值
    uint32_t cir;  // 承诺信息速率 (bytes/s)
};

// 使用示例
struct rte_meter_srtrcm srtrcm;
struct rte_meter_srtrcm_params params = {
    .cbs = 10000,      // 10KB 承诺桶
    .ebs = 20000,      // 20KB 超额桶
    .cir = 100000000,  // 100Mbps
};

rte_meter_srtrcm_config(&srtrcm, &params);

// 对每个数据包进行标记
enum rte_meter_color {
    RTE_METER_GREEN = 0,   // 在 CIR 内
    RTE_METER_YELLOW = 1,  // 超过 CIR 但在 EBS 内
    RTE_METER_RED = 2,     // 超过 EBS，可丢弃
};

enum rte_meter_color color = rte_meter_srtrcm_run(&srtrcm, pkt_len);

// 根据颜色设置 DSCP 标记
if (color == RTE_METER_RED) {
    // 设置 ECN 或丢弃
    m->ol_flags |= PKT_TX_MARK_IP_ECN;
}
```

#### 8.2 Scheduler (层次化调度)

```
流量管理类 (TC) 层次结构:

Port (端口)
  │
  ├─ Subport 0
  │    ├─ Pipe 0 ─┬─ TC0 (队列 0-3, 优先级)
  │    │          ├─ TC1 (队列 4-7)
  │    │          ├─ TC2 (队列 8-11)
  │    │          └─ TC3 (队列 12-15, Best Effort)
  │    ├─ Pipe 1
  │    └─ ...
  ├─ Subport 1
  └─ ...

每个 TC 可配置:
  - WFQ (Weighted Fair Queuing) 权重
  - WRED (Weighted Random Early Detection) 参数
```

**Scheduler 配置示例:**
```c
#include <rte_sched.h>

struct rte_sched_port_params port_param = {
    .name = "sched_port",
    .rate = 10000000000,  // 10Gbps
    .mtu = 1500,
    .frame_overhead = RTE_SCHED_FRAME_OVERHEAD_DEFAULT,
    .n_subports_per_port = 1,
    .n_pipes_per_subport = 256,
    .qsize = {64, 64, 64, 64},  // 每 TC 队列大小
};

struct rte_sched_subport_params subport_param = {
    .tb_rate = 10000000000,
    .tb_size = 1000000,
    .n_pipes = 256,
    .qsize = {64, 64, 64, 64},
    .wrr_weights = {1, 1, 1, 1},  // TC0-TC3 权重
};

struct rte_sched_pipe_params pipe_profile = {
    .tb_rate = 100000000,  // 100Mbps per pipe
    .tb_size = 100000,
    .tc_rate = {100000000, 50000000, 30000000, 20000000},
    .tc_period = 10,  // ms
};

// 创建 scheduler 端口
struct rte_sched_port *sched_port = rte_sched_port_config(&port_param);

// 配置 subport
rte_sched_subport_config(sched_port, 0, &subport_param);

// 配置 pipe 模板
rte_sched_pipe_profile_add(sched_port, &pipe_profile);

// 发送数据包时，设置 queue_id 决定 TC
uint16_t queue_id = (subport_id * 16) + (pipe_id * 4) + tc_id;
rte_eth_tx_burst(sched_port_id, queue_id, &m, 1);
```

### 9. Ring 库高级用法

#### 9.1 多生产者/多消费者 Ring

```c
// Ring 创建
struct rte_ring *r = rte_ring_create(
    "my_ring",
    4096,              // 大小 (必须是 2 的幂)
    rte_socket_id(),
    RING_F_SP_ENQ |    // 多生产者
    RING_F_SC_DEQ      // 多消费者
);

// 批量入队 (多生产者安全)
struct rte_mbuf *bufs[BURST_SIZE];
int ret = rte_ring_enqueue_bulk(r, (void **)bufs, BURST_SIZE, NULL);
if (ret == 0) {
    // Ring 满
}

// 批量出队 (多消费者安全)
ret = rte_ring_dequeue_bulk(r, (void **)bufs, BURST_SIZE, NULL);
if (ret == 0) {
    // Ring 空
}
```

#### 9.2 Ring 内存布局

```
┌───────────────────────────────────────────────────────────────────┐
│                          rte_ring                                 │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  控制结构 (8KB, 包含在 ring 中)                               │  │
│  │   - prod: 生产者索引 (缓存行隔离)                             │  │
│  │   - cons: 消费者索引 (缓存行隔离)                             │  │
│  │   - mask: 环形掩码 (size-1)                                  │  │
│  └─────────────────────────────────────────────────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  数据区 (连续内存，存储指针)                                  │  │
│  │   ┌────────┬────────┬────────┬──────────┬────────┐         │  │
│  │   │ ptr[0] │ ptr[1] │ ptr[2] │   ...    │ ptr[N] │         │  │
│  │   └────────┴────────┴────────┴──────────┴────────┘         │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘

注意：Ring 存储的是指针，不是实际数据
数据仍在 mbuf 中，ring 只负责传递 mbuf 指针
```

### 6. 最佳实践清单

#### 6.1 内存管理

- [ ] 始终使用大页内存 (2MB 或 1GB)
- [ ] 确保 mempool 和 lcore 在同一 NUMA socket
- [ ] 设置合适的 mempool cache size (通常 256)
- [ ] 预分配足够 mbuf，避免运行时分配
- [ ] 使用 `rte_pktmbuf_free_bulk()` 批量释放

#### 6.2 性能优化

- [ ] 使用批量 API (rx_burst/tx_burst)
- [ ] 启用 RSS 分散流量
- [ ] 启用硬件 offload (checksum, TSO, LRO)
- [ ] 使用预取 (`rte_prefetch0`)
- [ ] 保持数据结构缓存行对齐 (`__rte_cache_aligned`)
- [ ] 避免伪共享 (per-lcore 变量)

#### 6.3 错误处理

- [ ] 检查所有 DPDK API 返回值
- [ ] 优雅处理内存分配失败
- [ ] 实现 backpressure 防止 TX 队列溢出
- [ ] 定期轮询统计信息检测异常


## 自动识别触发

当检测到以下上下文时，自动应用 DPDK 专家知识：

1. **文件特征**:
   - 包含 `#include <rte_*.h>` 的 C 代码
   - meson.build/Makefile 中有 `libdpdk` 依赖
   - 命令行参数包含 `-l 0-3`, `--socket-mem` 等 EAL 参数

2. **代码模式**:
   - `rte_eal_init()`, `rte_eth_rx_burst()`, `rte_pktmbuf_alloc()`
   - `struct rte_mbuf *`, `struct rte_mempool *`
   - DPDK 特有的结构体和 API

3. **问题场景**:
   - 大页内存相关问题
   - 高性能网络包处理
   - 多核网络应用


## 常用命令参考

### 10. 实际应用场景示例

#### 10.1 L3 转发 (IP 路由)

```c
#include <rte_ip.h>
#include <rte_hash.h>
#include <rte_lpm.h>

// 使用 LPM (最长前缀匹配) 进行 IP 路由
struct rte_lpm *lpm_table;
struct rte_lpm_config config = {
    .max_rules = 1024,
    .number_tbl8s = 256,
    .flags = 0,
};

// 初始化 LPM 表
lpm_table = rte_lpm_create("lpm_table", SOCKET_ID_ANY, &config);

// 添加路由条目
rte_lpm_add(lpm_table, 0xC0A80100, 24,  // 192.168.1.0/24
            1);  // 下一跳索引

// 查找路由
static inline int
lookup_route(struct rte_mbuf *m) {
    struct ipv4_hdr *ipv4 = rte_pktmbuf_mtod_offset(m, struct ipv4_hdr *,
                                                     sizeof(struct ether_hdr));
    uint32_t dst_ip = ipv4->dst_addr;
    uint8_t next_hop;

    int ret = rte_lpm_lookup(lpm_table, dst_ip, &next_hop);
    if (ret == 0) {
        return next_hop;  // 返回下一跳索引
    }
    return -1;  // 无匹配路由，丢弃
}

// 转发核心循环
static int
l3_forward_loop(void *arg) {
    struct rte_mbuf *bufs[BURST_SIZE];
    uint16_t tx_port;

    while (1) {
        // 从 RX 队列接收
        uint16_t nb_rx = rte_eth_rx_burst(rx_port, rx_queue, bufs, BURST_SIZE);

        uint16_t tx_counts[RTE_MAX_ETHPORTS] = {0};
        struct rte_mbuf *tx_packets[RTE_MAX_ETHPORTS][BURST_SIZE];

        for (uint16_t i = 0; i < nb_rx; i++) {
            // 解析 IP 头部
            struct ipv4_hdr *ipv4 = rte_pktmbuf_mtod_offset(bufs[i],
                                         struct ipv4_hdr *, sizeof(struct ether_hdr));

            // 检查 IPv4
            if (ipv4->version == 4) {
                // 递减 TTL
                ipv4->time_to_live--;
                if (ipv4->time_to_live == 0) {
                    rte_pktmbuf_free(bufs[i]);
                    continue;
                }

                // 重新计算校验和
                ipv4->hdr_checksum = 0;
                ipv4->hdr_checksum = rte_ipv4_cksum(ipv4);

                // 查找路由
                int next_hop = lookup_route(bufs[i]);
                if (next_hop >= 0) {
                    tx_port = port_map[next_hop];
                    tx_packets[tx_port][tx_counts[tx_port]++] = bufs[i];
                } else {
                    rte_pktmbuf_free(bufs[i]);
                }
            } else {
                rte_pktmbuf_free(bufs[i]);
            }
        }

        // 批量发送到各端口
        RTE_ETH_FOREACH_DEV(tx_port) {
            if (tx_counts[tx_port] > 0) {
                rte_eth_tx_burst(tx_port, tx_queue,
                                 tx_packets[tx_port], tx_counts[tx_port]);
            }
        }
    }
}
```

#### 10.2 负载均衡器 (L4)

```c
#include <rte_hash_crc.h>
#include <rte_tcp.h>

#define MAX_BACKENDS 8

struct backend_server {
    uint32_t ip;
    uint16_t port;
    uint64_t conn_count;
    uint64_t pkt_count;
};

struct backend_server backends[MAX_BACKENDS];
uint8_t num_backends;

// 一致性哈希选择后端
static inline struct backend_server*
select_backend(struct rte_mbuf *m) {
    struct tcp_hdr *tcp = rte_pktmbuf_mtod_offset(m, struct tcp_hdr *,
                                                    sizeof(struct ether_hdr) +
                                                    sizeof(struct ipv4_hdr));

    // 使用五元组哈希
    uint32_t hash_key = rte_be_to_cpu_32(tcp->src_port) ^
                        rte_be_to_cpu_32(tcp->dst_port) ^
                        rte_crc32(&tcp->src_port, 8);

    // 基于权重的轮询
    uint32_t idx = hash_key % num_backends;
    backends[idx].conn_count++;

    return &backends[idx];
}

// DNAT 转换
static inline void
do_dnat(struct rte_mbuf *m, struct backend_server *backend) {
    struct ipv4_hdr *ipv4 = rte_pktmbuf_mtod_offset(m, struct ipv4_hdr *,
                                                     sizeof(struct ether_hdr));
    struct tcp_hdr *tcp = rte_pktmbuf_mtod_offset(m, struct tcp_hdr *,
                                                   sizeof(struct ether_hdr) +
                                                   sizeof(struct ipv4_hdr));

    // 修改目的 IP 和端口
    ipv4->dst_addr = backend->ip;
    tcp->dst_port = backend->port;

    // 重新计算校验和
    ipv4->hdr_checksum = 0;
    ipv4->hdr_checksum = rte_ipv4_cksum(ipv4);
    tcp->cksum = 0;
    tcp->cksum = rte_ipv4_tcp_cksum(ipv4, tcp);
}

// 负载均衡核心循环
static int
load_balancer_loop(void *arg) {
    struct rte_mbuf *bufs[BURST_SIZE];

    while (1) {
        uint16_t nb_rx = rte_eth_rx_burst(vip_port, rx_queue, bufs, BURST_SIZE);

        for (uint16_t i = 0; i < nb_rx; i++) {
            struct backend_server *backend = select_backend(bufs[i]);
            do_dnat(bufs[i], backend);
            backend->pkt_count++;

            // 转发到后端
            rte_eth_tx_burst(backend_port, tx_queue, &bufs[i], 1);
        }
    }
}
```

#### 10.3 防火墙/ACL

```c
#include <rte_acl.h>

// ACL 规则定义
struct acl_rule {
    struct rte_acl_field_match match;
    uint32_t src_ip;
    uint32_t dst_ip;
    uint16_t src_port;
    uint16_t dst_port;
    uint8_t proto;
    uint8_t action;  // ALLOW=0, DENY=1
};

#define ACL_ALLOW 0
#define ACL_DENY  1

// 定义 ACL 字段
static struct rte_acl_field_def acl_fields[] = {
    {
        .type = RTE_ACL_FIELD_TYPE_IP_ADDR,
        .size = sizeof(uint32_t),
        .offset = offsetof(struct ipv4_hdr, src_addr),
    },
    {
        .type = RTE_ACL_FIELD_TYPE_IP_ADDR,
        .size = sizeof(uint32_t),
        .offset = offsetof(struct ipv4_hdr, dst_addr),
    },
    {
        .type = RTE_ACL_FIELD_TYPE_RANGE,
        .size = sizeof(uint16_t),
        .offset = offsetof(struct tcp_hdr, src_port),
    },
    {
        .type = RTE_ACL_FIELD_TYPE_RANGE,
        .size = sizeof(uint16_t),
        .offset = offsetof(struct tcp_hdr, dst_port),
    },
    {
        .type = RTE_ACL_FIELD_TYPE_RANGE,
        .size = sizeof(uint8_t),
        .offset = offsetof(struct ipv4_hdr, next_proto_id),
    },
};

struct rte_acl_ctx *acl_ctx;

// 初始化 ACL 上下文
void init_acl(void) {
    struct rte_acl_param param = {
        .name = "firewall_acl",
        .rule_size = sizeof(struct acl_rule),
        .max_rule_num = 1000,
    };

    acl_ctx = rte_acl_create(&param, 0);

    // 添加规则
    struct acl_rule rules[] = {
        // 允许 SSH
        {.src_ip = 0, .dst_ip = 0, .src_port = 0, .dst_port = 22,
         .proto = IPPROTO_TCP, .action = ACL_ALLOW},
        // 允许 HTTP/HTTPS
        {.src_ip = 0, .dst_ip = 0, .src_port = 0, .dst_port = 80,
         .proto = IPPROTO_TCP, .action = ACL_ALLOW},
        {.src_ip = 0, .dst_ip = 0, .src_port = 0, .dst_port = 443,
         .proto = IPPROTO_TCP, .action = ACL_ALLOW},
        // 默认拒绝
        {.src_ip = 0, .dst_ip = 0, .src_port = 0, .dst_port = 0,
         .proto = 0, .action = ACL_DENY},
    };

    rte_acl_add_rules(acl_ctx, rules, RTE_DIM(rules));
    rte_acl_build(acl_ctx, NULL);
    rte_acl_reset(acl_ctx);
}

// 包过滤
static inline int
firewall_filter(struct rte_mbuf *m) {
    struct rte_mbuf *pkts[] = {m};
    uint32_t results[1];

    rte_acl_classify(acl_ctx, pkts, results, 1, 1);

    return results[0];  // ACL_ALLOW or ACL_DENY
}
```

#### 10.4 流量监控/NetFlow

```c
#include <rte_time.h>

struct flow_record {
    uint32_t src_ip;
    uint32_t dst_ip;
    uint16_t src_port;
    uint16_t dst_port;
    uint8_t proto;
    uint64_t pkt_count;
    uint64_t byte_count;
    uint64_t first_seen;
    uint64_t last_seen;
};

#define FLOW_TABLE_SIZE 65536
struct flow_record flow_table[FLOW_TABLE_SIZE];

// 流哈希键
struct flow_key {
    uint32_t src_ip;
    uint32_t dst_ip;
    uint16_t src_port;
    uint16_t dst_port;
    uint8_t proto;
};

static inline uint32_t
flow_hash(const struct flow_key *key) {
    return rte_crc32(key, sizeof(*key)) & (FLOW_TABLE_SIZE - 1);
}

// 更新流记录
static inline void
update_flow(struct rte_mbuf *m) {
    struct ipv4_hdr *ipv4 = rte_pktmbuf_mtod_offset(m, struct ipv4_hdr *,
                                                     sizeof(struct ether_hdr));
    struct flow_key key = {
        .src_ip = ipv4->src_addr,
        .dst_ip = ipv4->dst_addr,
        .proto = ipv4->next_proto_id,
    };

    // 提取传输层端口
    if (ipv4->next_proto_id == IPPROTO_TCP) {
        struct tcp_hdr *tcp = rte_pktmbuf_mtod_offset(m, struct tcp_hdr *,
                                                       sizeof(struct ether_hdr) +
                                                       sizeof(struct ipv4_hdr));
        key.src_port = tcp->src_port;
        key.dst_port = tcp->dst_port;
    } else if (ipv4->next_proto_id == IPPROTO_UDP) {
        struct udp_hdr *udp = rte_pktmbuf_mtod_offset(m, struct udp_hdr *,
                                                       sizeof(struct ether_hdr) +
                                                       sizeof(struct ipv4_hdr));
        key.src_port = udp->src_port;
        key.dst_port = udp->dst_port;
    }

    uint32_t idx = flow_hash(&key);
    struct flow_record *rec = &flow_table[idx];

    if (rec->pkt_count == 0) {
        // 新流
        rec->src_ip = key.src_ip;
        rec->dst_ip = key.dst_ip;
        rec->src_port = key.src_port;
        rec->dst_port = key.dst_port;
        rec->proto = key.proto;
        rec->first_seen = rte_rdtsc();
    }

    rec->pkt_count++;
    rec->byte_count += m->pkt_len;
    rec->last_seen = rte_rdtsc();
}

// 导出流记录 (NetFlow v9 格式)
void export_flow_records(int sock, struct sockaddr *collector) {
    struct netflow_export {
        uint16_t version;
        uint16_t count;
        uint32_t sys_uptime;
        uint32_t unix_secs;
        // ... 流记录
    } export_pkt;

    uint64_t now = rte_rdtsc();
    uint64_t expire_ticks = rte_get_timer_hz() * 60;  // 60 秒超时

    for (int i = 0; i < FLOW_TABLE_SIZE; i++) {
        struct flow_record *rec = &flow_table[i];
        if (rec->pkt_count > 0 &&
            (now - rec->last_seen) > expire_ticks) {
            // 导出并清除
            sendto(sock, rec, sizeof(*rec), 0, collector, sizeof(*collector));
            memset(rec, 0, sizeof(*rec));
        }
    }
}
```

### 11. 常见错误代码速查表

| 错误码 | 值 | 含义 | 解决方案 |
|--------|-----|------|----------|
| `-ENOMEM` | -12 | 内存不足 | 增加大页数量，检查 mempool 大小 |
| `-EEXIST` | -17 | 资源已存在 | 更换名称或先清理残留 (`rm /var/run/dpdk/*`) |
| `-EINVAL` | -22 | 参数无效 | 检查 API 参数范围 |
| `-ENODEV` | -19 | 设备不存在 | 检查端口 ID 或 PCI 地址 |
| `-EBUSY` | -16 | 资源忙 | 检查是否已被其他进程占用 |
| `-EPERM` | -1 | 权限不足 | 使用 sudo 或检查大页权限 |
| `-ENOTSUP` | -95 | 不支持的操作 | 检查 NIC 功能和驱动版本 |
| `-EIO` | -5 | IO 错误 | 检查硬件连接和 PCI 配置 |
| `-ETIMEDOUT` | -110 | 超时 | 增加超时值或检查中断配置 |
| `-EAGAIN` | -11 | 重试 | 稍后重试 (常见于 ring 满/空) |

**rte_errno 常见值:**
```c
// 获取详细错误信息
if (rte_eth_dev_configure(...) < 0) {
    printf("Error %d: %s\n", rte_errno, rte_strerror(rte_errno));
}

// 常见 rte_errno:
// 0   - 成功
// 1   - EPERM (权限不足)
// 2   - ENOENT (无此条目)
// 12  - ENOMEM (内存不足)
// 16  - EBUSY (资源忙)
// 22  - EINVAL (无效参数)
// 95  - ENOTSUP (不支持)
```

### 12. 性能调优参数参考

```bash
# EAL 参数优化
-l 0-15              # 绑定 lcore (根据 CPU 拓扑)
--socket-mem 4096,0  # 按 socket 分配内存
--huge-dir=/mnt/huge # 指定大页目录
--file-prefix=app    # 多实例区分

# 驱动参数 (modprobe)
# ixgbe
options ixgbe InterruptThrottleRate=0,0       # 关闭中断节流
options ixgbe RSS=1,1                         # 启用 RSS
options ixgbe DCA=0,0                         # 关闭 DCA (调试时)

# iavf (Intel E810)
options iavf force_valid_desc=1               # 强制有效描述符

# 内核参数 (/etc/sysctl.conf)
net.core.netdev_max_backlog = 100000          # 内核网设备队列
net.core.rmem_max = 134217728                 # 最大接收缓冲区
net.core.wmem_max = 134217728                 # 最大发送缓冲区
vm.nr_hugepages = 4096                        # 大页数量
kernel.nmi_watchdog = 0                       # 关闭 NMI 看门狗 (性能)
```

## 常用命令参考

```bash
# 编译
meson build && cd build && ninja

# 运行
sudo ./app -l 0-7 -n 4 --socket-mem 2048,0 -p 0x3

# 调试
sudo ./dpdk-procinfo --file-prefix=primary --show-mem
sudo ./testpmd -l 0-3 -n 4 -- -i

# 性能测试
sudo ./testpmd --txonly --nb-cores=4 --rxq=4 --txq=4
```
