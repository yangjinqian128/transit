# virtio-net start_xmit reboot 死循环问题分析与修复

> 问题版本：Linux 6.6 | 修复来源：Linux master (`e13b6da7045f9`)

---

## 一、问题现象

Guest 执行 `reboot` 时，virtio-net 的 `start_xmit` 可能陷入**无限循环**，导致系统无法完成重启，表现为 CPU 100% 软锁定或 watchdog 超时告警。

---

## 二、根因分析

### 2.1 死循环发生的调用链

```
CPU0 (reboot path)                              CPU1 (网络栈仍在运行)
───────────                                     ────────────
kernel_restart()
  └─ kernel_restart_prepare()
      └─ device_shutdown()                       start_xmit() 正在执行...
          └─ dev->bus->shutdown(dev)                    │
              == virtio_dev_shutdown()             do-while 循环中:
                  │                                   │
                  ├─ virtio_break_device()        ┌─ virtqueue_disable_cb()
                  │    vq->broken = true ────────►│  (不检查 broken，正常执行)
                  │                               │
                  │                               ├─ free_old_xmit_skbs()
                  │                               │   → virtqueue_get_buf()
                  ├─ virtio_synchronize_cbs()     │   → 检查 vq->broken=true
                  │                               │   → return NULL
                  │                               │   → last_used_idx 不更新！
                  │                               │
                  └─ dev->config->reset(dev)      ├─ virtqueue_enable_cb_delayed()
                      → QEMU: virtio_reset()      │   → 不检查 vq->broken!!!
                      → used->idx 归零             │   → 读 used->idx = 0
                                                  │   → vq->last_used_idx = 旧值(如100)
                                                  │   → (u16)(0 - 100) = 0xFF9C
                                                  │   → 0xFF9C > bufs(0) → true
                                                  │   → return false
                                                  │       │
                                                  └─ 循环回去 ←── 永不退出
```

### 2.2 6.6 中 start_xmit 的问题代码

```c
// drivers/net/virtio_net.c:2405 (6.6)
static netdev_tx_t start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    bool kick = !netdev_xmit_more();
    bool use_napi = sq->napi.weight;

    /* 这个 do-while 循环是死循环的根源 */
    do {
        if (use_napi)
            virtqueue_disable_cb(sq->vq);      // ① 不检查 vq->broken
        free_old_xmit_skbs(sq, false);          // ② 检查 broken → return NULL
    } while (use_napi && kick &&                // ③ 不检查 vq->broken！！！
             unlikely(!virtqueue_enable_cb_delayed(sq->vq)));

    // ... 后面的代码永远执行不到
}
```

### 2.3 三个关键函数对 vq->broken 的检查不一致

| 函数 | 位置 | 检查 vq->broken？ | 行为 |
|------|------|:---:|------|
| **virtqueue_disable_cb_split** | `virtio_ring.c:1087` | **NO** | 写入 avail->flags，不检查 broken |
| **virtqueue_get_buf_ctx_split** | `virtio_ring.c:974` | **YES** | 检查到 broken → return NULL，不更新 `last_used_idx` |
| **virtqueue_enable_cb_delayed_split** | `virtio_ring.c:1133` | **NO** | 只比较 `used->idx - last_used_idx` 的值 |

这三个函数通过 do-while 串在一起，形成了以下死循环条件：

```
disable_cb (不检查) → get_buf (return NULL, 不更新 idx)
    → enable_cb_delayed (不检查, used->idx 归零但 last_used_idx 是旧值)
        → u16 下溢导致永远返回 false → 死循环
```

### 2.4 为什么 `used->idx` 会归零

`virtio_dev_shutdown` 调用 `dev->config->reset(dev)`，最终触发 QEMU 的 `virtio_reset()`：

```c
// QEMU hw/virtio/virtio.c:3238
void virtio_reset(VirtIODevice *vdev) {
    virtio_set_status(vdev, 0);  // status = 0 → 清空所有 vring 状态
    // used->idx 被重置为 0
}
```

Guest 侧的 `vq->last_used_idx` 仍保持之前的值（因为 `virtqueue_get_buf` 在 broken 后返回 NULL，不执行 `vq->last_used_idx++`）。

---

## 三、设备关闭路径分析

### 3.1 reboot 时走的是 bus->shutdown，不是 driver->freeze

```
kernel_restart()
  └─ kernel_restart_prepare()
      └─ device_shutdown()                 // drivers/base/core.c:4871
          └─ dev->bus->shutdown(dev)       // 优先 bus 级回调
              == virtio_bus.shutdown
              == virtio_dev_shutdown()     // drivers/virtio/virtio.c:404
```

`device_shutdown` 中 `bus->shutdown` **优先于** `driver->shutdown`，而 virtio-net **没有定义** `driver->shutdown` 回调。所以走的是 virtio_bus 的通用关闭路径。

### 3.2 virtio_dev_shutdown 的完整流程

```c
// drivers/virtio/virtio.c:404
static void virtio_dev_shutdown(struct device *_d)
{
    // 1. 如果驱动有自定义 shutdown，走驱动路径（virtio-net 没有，跳过）
    if (drv->shutdown) {
        drv->shutdown(dev);
        return;
    }

    // 2. 标记所有 virtqueue 为 broken
    virtio_break_device(dev);
    //   → 遍历所有 vq: WRITE_ONCE(vq->broken, true)

    // 3. 等待所有正在执行的中断回调完成（但不阻止 start_xmit 新调用）
    virtio_synchronize_cbs(dev);

    // 4. 硬件复位 → QEMU 清空 vring 状态（used->idx = 0）
    dev->config->reset(dev);
}
```

### 3.3 与 freeze/remove 路径的区别

| | reboot → virtio_dev_shutdown | hibernation → virtnet_freeze | 驱动卸载 → virtnet_remove |
|---|---|---|---|
| 入口 | `virtio_bus.shutdown` | `virtio_driver.freeze` | `virtio_driver.remove` |
| 先关网络栈？ | **否**（`netif_device_detach` 不在此路径） | 是（`virtnet_freeze_down` 先 close 再 detach） | 是（`unregister_netdev` 先于 remove） |
| vring buffer 释放 | **不释放** | 完整释放 | 完整释放 |
| virqueue 删除 | **不删除**（仅标记 broken） | 销毁 | 销毁 |
| 并发 start_xmit 风险 | **有**（网络栈仍活跃） | 无（已 close） | 无（已 unregister） |

**关键差异**：reboot 路径在 `vq->broken = true` + device reset 之后，网络栈仍然可能调用 `start_xmit`。而 freeze/remove 路径在 broken 之前已经通过 `virtnet_close` / `unregister_netdev` 阻止了新的发送请求。

---

## 四、修复方案

### 4.1 master 的修复思路

master commit `e13b6da7045f9` 从**结构上消除 do-while 循环**，而不是在 virtio_ring 层打补丁：

```
6.6 start_xmit:
  do { disable_cb; free; } while (enable_cb_delayed == false);
  //    ↑ 循环内检查 broken 不一致 → 死循环

master start_xmit:
  if (NAPI)  disable_cb only;      // 单次调用，不循环
  else       free_old_xmit_skbs;   // 单次调用，不循环
  // 回收工作：NAPI 模式下完全从 start_xmit 移除，交给 NAPI poll
```

### 4.2 六个具体改动

**改动 1：提取 `tx_may_stop`**

```c
// 从 check_sq_full_and_disable 中提取 stop 判断逻辑为独立函数
static bool tx_may_stop(struct virtnet_info *vi,
                        struct net_device *dev,
                        struct send_queue *sq)
{
    if (sq->vq->num_free < 2 + MAX_SKB_FRAGS) {
        netif_stop_subqueue(dev, sq - vi->sq);
        return true;
    }
    return false;
}
```

**改动 2：分流 NAPI 和非 NAPI 的流控**

```c
// 6.6:
check_sq_full_and_disable(vi, dev, sq);  // 统一调用

// master:
if (use_napi)
    tx_may_stop(vi, dev, sq);            // NAPI: 仅 stop，不回收
else
    check_sq_full_and_disable(vi, dev, sq);  // 非NAPI: stop + 同步回收
```

**改动 3：移除 do-while，分流回收路径**

```c
// 6.6:
do {
    if (use_napi) virtqueue_disable_cb(sq->vq);
    free_old_xmit_skbs(sq, false);
} while (use_napi && kick && unlikely(!virtqueue_enable_cb_delayed(sq->vq)));

// master: 无循环
if (!use_napi)
    free_old_xmit_skbs(sq, false);   // 非NAPI: 同步回收
else
    virtqueue_disable_cb(sq->vq);    // NAPI: 仅关回调，回收在 NAPI poll 中完成
```

**改动 4：BQL 动态 kick 决策**

```c
// 6.6:
bool kick = !netdev_xmit_more();
// ...
if (kick || netif_xmit_stopped(txq)) { ... }

// master:
bool xmit_more = netdev_xmit_more();
bool kick;
// ...
kick = use_napi ? __netdev_tx_sent_queue(txq, skb->len, xmit_more) :
                  !xmit_more || netif_xmit_stopped(txq);
if (kick) { ... }
```

NAPI 模式下使用 BQL（Byte Queue Limits）的 `__netdev_tx_sent_queue` 代替简单的 `!xmit_more` 判断，在吞吐和延迟之间实现数据驱动的均衡。

**改动 5：post-kick NAPI 竞赛处理**

```c
// master 新增（6.6 没有）
if (use_napi && kick && unlikely(!virtqueue_enable_cb_delayed(sq->vq)))
    virtqueue_napi_schedule(&sq->napi, sq->vq);
```

kick 之后 Host 可能极快地完成消费并写入 used ring，此时 `enable_cb_delayed` 可以立即检测到并调度 NAPI 回收——单次调用，不循环，不会死锁。

**改动 6：`xmit_more` 变量语义反转**

```c
// 6.6:
bool kick = !netdev_xmit_more();

// master:
bool xmit_more = netdev_xmit_more();  // 保留原始语义
bool kick;                            // 延迟决策
```

---

## 五、补丁文件

回合补丁已生成并通过 `git apply --check` 验证：

**文件：** `0001-virtio-net-fix-start_xmit-deadloop-on-reboot-6.6.patch`

### 适配说明

| master 原 patch 内容 | 6.6 适配 |
|---------------------|---------|
| `free_old_xmit(sq, txq, false)` | 6.6 函数名为 `free_old_xmit_skbs(sq, false)`，第二个参数是 `in_napi` 不是 `txq` |
| `sq->stats.stop/wake` 统计 | 6.6 的 `virtnet_sq_stats` 没有这些字段，`tx_may_stop` 中跳过 |
| `xmit_skb(sq, skb, !use_napi)` (3 参数) | 6.6 签名为 `xmit_skb(sq, skb)` (2 参数) |
| `virtnet_xmit_ptr_pack` 类型编码 | 6.6 未实现，不涉及 |

### 不需适配的部分

以下 API 在 6.6 中已存在且签名一致：

- `__netdev_tx_sent_queue(txq, bytes, xmit_more)` — 6.6 `include/linux/netdevice.h:3570`
- `netdev_xmit_more()` — 6.6 `include/linux/netdevice.h:4992`
- `virtqueue_napi_schedule()` — 6.6 `drivers/net/virtio_net.c:435`
- `virtqueue_enable_cb_delayed()` — 6.6 已存在
- `virtqueue_disable_cb()` — 6.6 已存在

---

## 六、调用链总结

```
Guest reboot (kernel_restart)
│
├─ kernel_restart_prepare()
│   └─ device_shutdown()
│       └─ virtio_dev_shutdown()              ← bus->shutdown 优先
│           ├─ [driver->shutdown == NULL]       virtio-net 未定义
│           ├─ virtio_break_device()            所有 vq->broken = true
│           ├─ virtio_synchronize_cbs()         等中断回调完成
│           └─ dev->config->reset()             QEMU virtio_reset → used->idx=0
│
│   并发风险:
│   start_xmit() 可能仍在执行
│   ├─ 6.6: do-while → enable_cb_delayed 不检查 broken → 死循环
│   └─ master: 无循环 → 单次调用 → 安全退出
│
├─ do_kernel_restart_prepare()
├─ migrate_to_reboot_cpu()
├─ syscore_shutdown()
└─ machine_restart(cmd)                       ← 最终重启硬件
```

---

## 七、修复验证方法

```bash
# 1. 应用补丁
cd /path/to/linux-6.6
git apply 0001-virtio-net-fix-start_xmit-deadloop-on-reboot-6.6.patch

# 2. 编译验证
make M=drivers/net/

# 3. 功能验证
#    - 在 VM 中反复执行 reboot，确认无 CPU 软锁定
#    - 用 pktgen 高压发送时 reboot，确认无 hang
#    - 验证 TX 性能无回归（netperf / iperf3）
```

---

*分析时间：2026-07-03，基于 Linux 6.6 (`/home/code/kernel`) vs Linux master (`/home/code/atomgit/linux`)*
