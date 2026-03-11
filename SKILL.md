---
name: asterinas-driver-dev
description: 给 Student 使用的 Asterinas 总驱动技能，用来总结跨题可复用的驱动写法、判断顺序、约束和反模式。何时使用：当你在实现或修改 Asterinas 驱动代码，需要知道先检查什么、什么时候这样写、为什么这样写、哪些地方最容易出错时。触发短语：Asterinas 驱动开发, virtio 驱动, 驱动写法, 队列与中断, DMA 约束.
roles: [student]
---

# Asterinas VirtIO 驱动开发指南

## 一、开始前的检查清单

1. 确认设备类型：查看 `transport.device_type()` 返回的 `VirtioDeviceType`
2. 确认队列数量和索引：不同设备有不同的队列布局
3. 确认需要的 DMA 缓冲区数量和大小

## 二、驱动初始化流程

必须按以下顺序执行：
1. 创建 VirtQueue（调用 VirtQueue::new）
2. 分配 DMA 缓冲区（DmaStream::alloc）
3. 构建设备结构体（用 Arc 包装）
4. 预填充接收队列（add_dma_buf + notify）
5. 注册中断回调（register_queue_callback）
6. 注册配置变更回调（register_cfg_callback）
7. 完成初始化（transport.finish_init）
8. 注册到子系统

**关键约束**：必须先创建队列、分配缓冲区，再注册回调，最后调用 finish_init。

## 三、参考已有实现

在实现新驱动前，**务必先阅读** `kernel/comps/virtio/src/device/` 下的已有实现：
- Console 驱动：`console/device.rs`
- Input 驱动：`input/device.rs`
- Network 驱动：`network/device.rs`
- Block 驱动：`block/device.rs`

这些是 Asterinas 的标准写法，应该作为主要参考。

## 四、VirtQueue API

### 添加缓冲区
- 设备读取的数据放在 inputs 数组
- 设备写入的数据放在 outputs 数组
- 调用 add_dma_buf 后检查 should_notify，决定是否通知设备

### 取回缓冲区
- 用 can_pop 检查是否有已处理缓冲区
- 用 pop_used 取回缓冲区和长度
- 处理完后要把缓冲区重新放回队列

## 五、DMA 缓冲区约束

### 发送数据（Driver → Device）
1. 用 writer() 写入数据
2. 必须调用 sync_to_device 同步
3. 然后添加到队列

### 接收数据（Device → Driver）
1. 必须调用 sync_from_device 同步
2. 用 reader() 读取数据，需要用 limit() 设置长度

**最易错点**：忘记同步缓冲区会导致数据不一致。

## 六、中断处理

- 回调中获取队列锁时要使用 disable_irq().lock()，避免嵌套中断
- 处理完事件后要把缓冲区重新放回队列

## 七、特性协商

- 使用 from_bits_truncate 处理特性位，而非 from_bits
- 根据设备类型移除不支持的特性

## 八、常见反模式

1. 队列索引混淆：确认每个队列的索引号
2. DMA 同步遗漏：必须在数据传输前后同步
3. 中断嵌套：在回调中直接获取锁而不禁用中断
4. 缓冲区泄漏：从队列取出后忘记放回

## 九、编码步骤

1. **先看同类驱动**：参考已有实现了解标准写法
2. **理解队列布局**：确认每个队列的用途和索引
3. **先跑通编译**：确保类型正确再处理逻辑
4. **逐个功能验证**：先验证基础功能，再添加高级特性
