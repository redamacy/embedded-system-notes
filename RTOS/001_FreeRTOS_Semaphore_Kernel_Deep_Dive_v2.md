# FreeRTOS 信号量内核深度分析

> **核心论点：Semaphore 就是 `uxItemSize=0` 的 Queue。两个事件链表 + 一个计数器，串联 ISR → Scheduler → Task 的系统流动。**
>
> 源码：Gecko SDK 4.1.2 / FreeRTOS Kernel / CMSIS-RTOS2 Wrapper
>
> 工程上下文：FG23 BLE MicroStation，4 个二值信号量 `osSemaphoreNew(1, 0, ...)` 驱动全部数据流

---

## 1. 核心模型

```c
// FreeRTOSEntry.c:442 — 工程实际创建
FG14ReceiveSemaphoreHandle = osSemaphoreNew(1, 0, &FG14ReceiveSemaphore_attributes);
```

**Semaphore 不另起炉灶，它复用了 Queue 的全部操作原语。**

| Queue 字段 | Semaphore 语义 |
|------------|---------------|
| `uxMessagesWaiting` | 信号量当前计数 |
| `uxLength` | 最大计数 (max_count) |
| `uxItemSize` | **0** — 信号量不搬运数据，只传递"事件" |
| `xTasksWaitingToReceive` | 等待 Take 的阻塞任务链表（按优先级降序） |
| `xTasksWaitingToSend` | 等待 Give 的阻塞任务链表（计数满时，工程中不触发） |

**不分配数据存储区：** `pvPortMalloc(sizeof(Queue_t) + 0)` 只分配控制块，没有 `pcHead → pcTail` 的环形缓冲。

### 关键行为规则

```
uxMessagesWaiting == 0  → Take 方阻塞 → 挂入 xTasksWaitingToReceive
uxMessagesWaiting >= uxLength → Give 方阻塞 → 挂入 xTasksWaitingToSend
Give 成功后 → 优先唤醒 xTasksWaitingToReceive (如果有)
Take 成功后 → 优先唤醒 xTasksWaitingToSend (如果有)
```

两个链表配合 `uxMessagesWaiting` 计数器，天然形成一个 **生产者-消费者阻塞队列**。

---

## 2. 创建：osSemaphoreNew(1, 0, NULL)

### 2.1 调用链

```
osSemaphoreNew(1, 0, &attr)                         // CMSIS-RTOS2 wrapper
  ├── max_count==1 → 走二值信号量路径
  ├── xSemaphoreCreateBinary()                       // semphr.h 内联宏
  │     └── xQueueGenericCreate(
  │           uxQueueLength  = 1,
  │           uxItemSize     = queueSEMAPHORE_QUEUE_ITEM_LENGTH = 0,
  │           ucQueueType    = queueQUEUE_TYPE_BINARY_SEMAPHORE
  │         )
  │           │
  │           ├── pvPortMalloc(sizeof(Queue_t) + 0)   // 只分配控制块
  │           │
  │           └── prvInitialiseNewQueue(1, 0, storage, type, pxNewQueue)
  │                 ├── pcHead = (int8_t *)pxNewQueue  // 指向自身(无数据区)
  │                 ├── uxLength = 1
  │                 ├── uxItemSize = 0
  │                 └── xQueueGenericReset(pxNewQueue, pdTRUE)
  │                       ├── vListInitialise(&xTasksWaitingToSend)    // 见 Appendix A
  │                       ├── vListInitialise(&xTasksWaitingToReceive)
  │                       ├── pcWriteTo = pcHead, u.xQueue.pcTail = pcHead
  │                       ├── u.xQueue.pcReadFrom = pcHead
  │                       └── uxMessagesWaiting = 0   // ← 初始计数
  │
  ├── initial_count == 0 → 跳过 xSemaphoreGive()
  └── 返回 (osSemaphoreId_t)hSemaphore
```

### 2.2 初始化后的关键状态

```
uxMessagesWaiting     = 0     ← 无令牌，任何 Take 都会阻塞
uxLength              = 1     ← 二值信号量
uxItemSize            = 0     ← 不存储数据
xTasksWaitingToSend   = 空    ← [xListEnd ←→ 自己]
xTasksWaitingToReceive = 空   ← [xListEnd ←→ 自己]
```

### 2.3 为什么 initial_count=0

```c
// cmsis_os2.c — initial_count==0 跳过 Give
if ((hSemaphore != NULL) && (initial_count != 0U)) {
    xSemaphoreGive(hSemaphore);  // 不执行
}
```

**设计意图：任务创建后立即阻塞，完全由事件驱动。** 工程中 4 个信号量全部 `(1, 0)`：

| 信号量 | 唤醒源 | 模式 |
|--------|--------|------|
| FG14ReceiveSemaphore | RAIL RX 回调 | RF 到达 → 处理 |
| FG14SendSemaphore | TIMER1 / 缓冲区满 | 有待发数据 → 发送 |
| BG22ReceiveSemaphore | GPIO SPI CS 中断 | BLE 数据到达 → 读取 |
| SendMutexSemaphore | TIMER2 ACK 中断 | ACK 到达 → 继续 |

这不是"初始没有资源"，而是"**事件尚未发生**"。

---

## 3. Give：中断释放信号量

以工程实际场景：**RAIL 中断回调中 `osSemaphoreRelease(FG14ReceiveSemaphoreHandle)`**。

### 3.1 热路径：无等待者

最常见的情况——还没有任务调过 `osSemaphoreAcquire`：

```
调用前:  uxMessagesWaiting = 0, Receive 链表 = 空

xQueueGenericSendFromISR()
  ├── uxMessagesWaiting(0) < uxLength(1) → 可以 Give
  ├── prvCopyDataToQueue()  // uxItemSize=0 → 不 memcpy
  │     └── uxMessagesWaiting = 1    // 0 → 1
  └── 检查 xTasksWaitingToReceive → 空 → 不做任何唤醒

调用后:  uxMessagesWaiting = 1, Receive 链表 = 空 (未变)
```

**全程只改了一个计数器，两个链表碰都没碰。** 如果此时再有第二次 Release（信号量已满），`uxMessagesWaiting(1) >= uxLength(1)` → 返回失败。

### 3.2 唤醒路径：有任务在等 Take

正常运行时 FG14_Receive_Task 已经阻塞在 `osSemaphoreAcquire`，ISR 到达时：

```
调用前:
  uxMessagesWaiting = 0
  xTasksWaitingToReceive = [ FG14_Receive_Task(prio=15) ]
```

```
xQueueGenericSendFromISR(pxQueue, NULL, &xHigherPriorityTaskWoken, queueSEND_TO_BACK)
  │
  ├── uxMessagesWaiting(0) < uxLength(1) → 可以 Give
  ├── prvCopyDataToQueue() → uxMessagesWaiting = 1
  │
  └── if (!listLIST_IS_EMPTY(&xTasksWaitingToReceive))
        │   // uxNumberOfItems = 1 → NOT empty
        │
        └── xTaskRemoveFromEventList(&xTasksWaitingToReceive)
              │
              ├── ① pxUnblockedTCB = listGET_OWNER_OF_HEAD_ENTRY(pxEventList)
              │      = FG14_Receive_Task 的 TCB (最高优先级在链表头)
              │
              ├── ② uxListRemove(&pxUnblockedTCB->xEventListItem)
              │      ★ 从 xTasksWaitingToReceive 摘除
              │      ★ uxNumberOfItems: 1 → 0，xListEnd 恢复自指
              │
              ├── ③ uxListRemove(&pxUnblockedTCB->xStateListItem)
              │      ★ 从 DelayedList 摘除
              │
              ├── ④ prvAddTaskToReadyList(pxUnblockedTCB)
              │      ★ 插入 ReadyList[15]
              │
              └── ⑤ pxUnblockedTCB->uxPriority(15) > pxCurrentTCB->uxPriority(?)
                    → 返回 pdTRUE → *pxHigherPriorityTaskWoken = pdTRUE
                    → ISR 返回时触发 PendSV → 切换到 FG14_Receive_Task
```

**链表变化：**

```
★ uxListRemove 前:                       ★ uxListRemove 后:

xTasksWaitingToReceive:                  xTasksWaitingToReceive:
┌────────────────────┐                  ┌────────────────────┐
│ uxNumberOfItems=1  │                  │ uxNumberOfItems=0  │
│ xListEnd:          │                  │ xListEnd:          │
│  pxNext ──→ TCB    │                  │  pxNext ──→ 自己   │  ← 恢复空链表
│  pxPrev ──→ TCB    │                  │  pxPrev ──→ 自己   │
└────────────────────┘                  └────────────────────┘

同时: TCB.xStateListItem: DelayedList → ReadyList[15]
```

**最终状态：** `uxMessagesWaiting = 1`，`xTasksWaitingToReceive = 空`，FG14_Receive_Task 从 Blocked → Ready。

### 3.3 ISR 版本差异

工程中的 Release 都在中断上下文——CMSIS-RTOS2 检测到 ISR 环境，走 `xSemaphoreGiveFromISR()`：

| | 任务上下文 | ISR 上下文 |
|---|---|---|
| 函数 | `xQueueGenericSend()` | `xQueueGenericSendFromISR()` |
| 唤醒方式 | `queueYIELD_IF_USING_PREEMPTION()` | `portYIELD_FROM_ISR(pxHigherPriorityTaskWoken)` |
| 事件链表操作 | 直接操作 | 调度器锁住时通过 `cRxLock/cTxLock` 延迟处理 |

**cRxLock/cTxLock 机制（简要）：** ISR 中 Give 时如果调度器正被 `vTaskSuspendAll()` 冻结，ISR 不直接操作事件链表，而是通过 `cRxLock` 计数器记录"ISR 期间到达的事件"。等调度器解锁时（`xTaskResumeAll`），统一处理这些积压事件。

---

## 4. Take：任务获取信号量

以工程实际场景：**FG14_Receive_Task 调用 `osSemaphoreAcquire(sem, portMAX_DELAY)`**。

### 4.1 热路径：有令牌

信号量刚被 ISR Give 过，`uxMessagesWaiting > 0`：

```c
// xQueueSemaphoreTake() — 热路径，进都不进阻塞分支
taskENTER_CRITICAL();
uxSemaphoreCount = pxQueue->uxMessagesWaiting;  // = 1
if (uxSemaphoreCount > 0) {                     // TRUE!
    pxQueue->uxMessagesWaiting = 0;             // 1 → 0
    // 检查 xTasksWaitingToSend → 空 → 跳过
}
taskEXIT_CRITICAL();
return pdPASS;  // 立即成功，两个链表碰都没碰
```

### 4.2 阻塞路径：无令牌

系统启动后首次 Acquire——`uxMessagesWaiting == 0`：

```
xQueueSemaphoreTake(pxQueue, portMAX_DELAY)
  │
  ├── ① taskENTER_CRITICAL()
  │      uxSemaphoreCount = 0 → 进入 else 分支
  │
  ├── ② taskEXIT_CRITICAL()           // 开中断！ISR 此刻仍可 Give
  │
  ├── ③ vTaskSuspendAll()             // 冻结调度器，不断中断
  │     prvLockQueue(pxQueue)          // 锁事件链表（ISR Give 走 cRxLock）
  │
  ├── ④ if (prvIsQueueEmpty(pxQueue))  // uxMessagesWaiting 还是 0？
  │     │  // TRUE — ISR 没在此期间 Give
  │     │
  │     ├── vTaskPlaceOnEventList(&xTasksWaitingToReceive, portMAX_DELAY)
  │     │     │
  │     │     ├── vListInsert(&xTasksWaitingToReceive, &pxCurrentTCB->xEventListItem)
  │     │     │     ★ 按 uxPriority=15 降序插入
  │     │     │     ★ xEventListItem 被记录为"在等 Receive 事件"
  │     │     │     ★ uxNumberOfItems: 0 → 1
  │     │     │
  │     │     └── prvAddCurrentTaskToDelayedList(portMAX_DELAY, pdTRUE)
  │     │           ★ xStateListItem: ReadyList[15] → DelayedList
  │     │           ★ 任务状态: Running → Blocked
  │     │
  │     ├── prvUnlockQueue(pxQueue)
  │     │
  │     └── if (xTaskResumeAll() == pdFALSE)
  │           ★ 当前任务已不在 ReadyList → 调度器选新任务
  │           → portYIELD_WITHIN_API() → PendSV → 上下文切换
  │
  └── ★ 代码不会立刻返回。下次被 ISR Give 唤醒后，
        从 for(;;) 循环重新检查 uxMessagesWaiting，此时 > 0，走热路径。
```

**同一个 TCB 同时出现在两个链表中：**

```
FC14_Receive_Task 阻塞后:

  TCB.xEventListItem  → xTasksWaitingToReceive   (等什么事件)
  TCB.xStateListItem  → DelayedList              (超时管理)

  被唤醒时从两个链表同时摘除，回到 ReadyList[15]。
```

**链表变化总结：**

```
Acquire 前 (Running):                  Acquire 后 (Blocked):
  ReadyList[15] = [FG14_Receive]         ReadyList[15] = [其他任务]
  DelayedList = 空                       DelayedList = [FG14_Receive]
  xTasksWaitingToReceive = 空            xTasksWaitingToReceive = [FG14_Receive]
  pxCurrentTCB = FG14_Receive            pxCurrentTCB = 下一个最高优先级就绪任务
```

---

## 5. 系统流动：ISR → Scheduler → Task

### 5.1 完整时序图

```
时间 →

FG14_Receive_Task  运行 ──→ Take失败 ──→ BLOCKED ────────────────→ 被唤醒 ──→ RUNNING
(prio 15)               │                                          │
                         │ vTaskPlaceOnEventList()                  │ xTaskRemoveFromEventList()
                         │ xEventListItem →                         │ xEventListItem 摘除
                         │   xTasksWaitingToReceive                 │ xStateListItem:
                         │ xStateListItem:                          │   DelayedList → ReadyList[15]
                         │   ReadyList[15] → DelayedList            │
                         │                                          │
FG14_Send_Task            READY ──→ RUNNING ────────────────────→ READY
(prio 14)                 │ (调度器切换到它)
                         │
RAIL ISR            ─────────────────────────── RF 到达 ──→ osSemaphoreRelease()
(prio 2 HW)                                                   uxMessagesWaiting: 0→1
                                                               TCB 唤醒 → ReadyList[15]
                                                               PendSV 挂起
                                                               ISR 返回 → PendSV →
                                                               FG14_Receive_Task 运行
```

### 5.2 双层保护机制

```c
taskENTER_CRITICAL();    // 关中断 — 保护极短的 queue 状态检查
// ...检查 uxMessagesWaiting...
taskEXIT_CRITICAL();     // 开中断 — 此时 ISR 可以 Give

vTaskSuspendAll();       // 冻结调度器 — 防止其他任务打断链表操作
prvLockQueue(pxQueue);   // 锁事件链表 — ISR Give 通过 cRxLock 记录
// ...操作 xTasksWaitingToReceive...
prvUnlockQueue(pxQueue);
xTaskResumeAll();        // 恢复调度器 — 处理 pending ready list
```

**精妙之处：** 中断只关最短的时间（`uxMessagesWaiting` 检查），链表操作期间中断仍然能 fire。ISR 中的 Give 不直接操作链表，而是通过 `cRxLock/cTxLock` 计数器延迟到调度器恢复时处理——既保证了链表完整性，又避免了长时间关中断。

### 5.3 工程完整数据流：4 个信号量串联

```
GPIO ISR → BG22ReceiveSemaphore → BG22_Receive_Task → 填充缓冲区
TIMER1 ISR → FG14SendSemaphore → FG14_Send_Task → 构建/发送 RF 帧
RAIL RX ISR → FG14ReceiveSemaphore → FG14_Receive_Task → 处理下行数据
TIMER2 ISR → SendMutexSemaphore → 发送流程 ACK 确认
```

全部 4 个信号量都是 `(1, 0)` — 统统"事件驱动，初始阻塞"。**CPU 只在有事做的时候运行，其余时间全在 IDLE（可进入低功耗 sleep）。**

---

## 6. 设计意图

### 6.1 Semaphore = Queue (uxItemSize=0)

信号量的所有操作（阻塞、唤醒、优先级排序、超时）与队列 95% 一样。FreeRTOS 用同一套 Queue 机制统一了消息队列、二值信号量、计数信号量、互斥量。内核代码量极大减少，bug 面也缩小。

### 6.2 TCB 双 ListItem (xStateListItem + xEventListItem)

| ListItem | 所在链表 | 作用 |
|----------|----------|------|
| `xStateListItem` | ReadyList / DelayedList / SuspendedList | 表示任务调度状态 |
| `xEventListItem` | Queue 的 xTasksWaitingToReceive / xTasksWaitingToSend | 表示在等什么事件 |

一个任务可以同时处于 DelayedList（超时管理）和 EventList（事件等待）。不论哪种方式先到期，任务都能被正确唤醒并从两个链表中摘除。

### 6.3 Critical Section + Scheduler Suspend 分层

不是"关中断做所有事"，而是"关中断做最少的事，调度锁做大块的事，ISR 通过 lock 计数器延迟处理"。这保证了实时性（ISR 延迟短）和正确性（链表不被并发破坏）。

### 6.4 initial_count=0：事件驱动 > 轮询

`initial=0` 确保任务创建后就阻塞，不浪费 CPU 时间片。配合 `portMAX_DELAY`，任务不超时，完全事件驱动。这是实时嵌入式系统的核心范式。

---

## Appendix A: List_t / ListItem_t 数据结构

```c
// FreeRTOS list.h

typedef struct xLIST {
    volatile UBaseType_t uxNumberOfItems;   // 链表中有几个节点
    ListItem_t          *pxIndex;           // 遍历游标
    MiniListItem_t       xListEnd;          // 哨兵节点 — 永远在链表中
} List_t;

struct xLIST_ITEM {
    TickType_t            xItemValue;       // 排序键值 — 按优先级或 tick 排序
    struct xLIST_ITEM    *pxNext;           // 双向链表 → 后继
    struct xLIST_ITEM    *pxPrevious;       // 双向链表 → 前驱
    void                 *pvOwner;          // 指向拥有者 TCB
    struct xLIST         *pvContainer;      // 反向指向所属的 List_t
};

struct xMINI_LIST_ITEM {
    TickType_t            xItemValue;
    struct xLIST_ITEM    *pxNext;
    struct xLIST_ITEM    *pxPrevious;
};
```

**关键设计：`xListEnd` 是哨兵节点，永远在链表中，是双向循环链表的"锚"。**

```
空链表:                              非空链表 (1个节点):
┌──────────────┐                     ┌──────────────┐
│  xListEnd    │                     │  xListEnd    │
│  pxNext ───────┐                   │  pxNext ──────────┐
│  pxPrev ───┐   │                   │  pxPrev ──────┐   │
└────────────│───┘                   └───────────────│───┘
             │   ┌──────────┐                        │   ┌──────────────────────┐
             └──→│ 自己     │                        │   │ xEventListItem (TCB) │
             ←───│          │                        │   │ xItemValue = 15(pri) │
                 └──────────┘                        │   │ pxNext ──────────┐   │
                                                     │   │ pxPrev ───┐      │   │
                                                     │   │ pvOwner=TCB      │   │
                                                     │   └──────────│───────┘   │
                                                     └──────────────┘           │
                                                         ↑                      │
                                                         └──────────────────────┘
```

**哨兵 `xItemValue = portMAX_DELAY` (0xFFFFFFFF)：** 任务按优先级降序排列（数值大排前面），任何任务优先级 `≤ configMAX_PRIORITIES`，远小于 `portMAX_DELAY`，所以哨兵永远在"末尾"。

### vListInitialise 5 条语句

```c
void vListInitialise(List_t * const pxList)
{
    pxList->pxIndex = (ListItem_t *) &(pxList->xListEnd);    // ① 游标指向哨兵
    pxList->uxNumberOfItems = (UBaseType_t) 0U;               // ② 计数器清零
    pxList->xListEnd.xItemValue = (TickType_t) portMAX_DELAY; // ③ 哨兵值=最大
    pxList->xListEnd.pxNext = (ListItem_t *) &(pxList->xListEnd);     // ④ 闭环
    pxList->xListEnd.pxPrevious = (ListItem_t *) &(pxList->xListEnd); // ⑤ 闭环
}
```

---

## Appendix B: Queue_t 初始化后完整内存布局

```
地址          内容                       所属字段
───────────  ────────────────────────   ─────────────────────────
0x2000_1000  → 0x2000_1000              pcHead (指向Queue_t自身)
0x2000_1004  → 0x2000_1000              pcWriteTo
0x2000_1008  → ...                      u 联合体
0x2000_100C  → ...

              ── xTasksWaitingToSend ──
0x2000_1010  0x0000_0000                uxNumberOfItems = 0
0x2000_1014  → &xListEnd (0x2000_1018)  pxIndex
0x2000_1018  0xFFFF_FFFF                xListEnd.xItemValue
0x2000_101C  → 0x2000_1018              xListEnd.pxNext → 自己
0x2000_1020  → 0x2000_1018              xListEnd.pxPrevious → 自己

              ── xTasksWaitingToReceive ──
0x2000_1024  0x0000_0000                uxNumberOfItems = 0
0x2000_1028  → &xListEnd (0x2000_102C)  pxIndex
0x2000_102C  0xFFFF_FFFF                xListEnd.xItemValue
0x2000_1030  → 0x2000_102C              xListEnd.pxNext → 自己
0x2000_1034  → 0x2000_102C              xListEnd.pxPrevious → 自己

              ── 队列控制 ──
0x2000_1038  0x0000_0000                uxMessagesWaiting = 0
0x2000_103C  0x0000_0001                uxLength = 1
0x2000_1040  0x0000_0000                uxItemSize = 0
0x2000_1044  0xFFFF_FFFF (-1)           cRxLock = queueUNLOCKED
0x2000_1048  0xFFFF_FFFF (-1)           cTxLock = queueUNLOCKED
```

每个 `List_t` 占 20 字节（`uxNumberOfItems` 4 + `pxIndex` 4 + `MiniListItem_t` 12），两个链表在内存中紧挨存放，逻辑上完全独立。

---

## Appendix C: xTasksWaitingToSend — 信号量满时阻塞 Give

**工程中不触发**（所有 Release 来自 ISR，`xTicksToWait=0`），但机制上值得了解。

**触发条件：** 某**任务**（非 ISR）调用 `osSemaphoreRelease()` 时 `uxMessagesWaiting >= uxLength`，且 `xTicksToWait > 0`。该任务挂入 `xTasksWaitingToSend`，直到有人 Take 腾出空间后被唤醒。

**应答关系：**

```
Take 成功 (uxMessagesWaiting--) →
  检查 xTasksWaitingToSend → 不空 →
  取出等待 Give 的任务 → 唤醒 → 把令牌"传递"给它
```

---

## Appendix D: PendSV 上下文切换

PendSV（Pendable Service Call）是 Cortex-M 中优先级最低的可挂起中断，专门用于上下文切换。

```
portYIELD_WITHIN_API()
  └── portYIELD()
        └── 置位 PendSV → 硬件自动入栈 (R0-R3, R12, LR, PC, xPSR)
              → PendSV_Handler:
                  保存 R4-R11 到当前任务栈
                  pxCurrentTCB->pxTopOfStack = SP
                  选最高优先级就绪任务 → pxCurrentTCB = 新任务
                  SP = 新任务->pxTopOfStack
                  恢复 R4-R11
                  硬件自动出栈 → 跳转到新任务的 PC
```

为什么用 PendSV 而不是在 ISR 中直接切换？因为 ISR 中切换会打断中断嵌套——PendSV 在所有硬件 ISR 返回后才执行，保证切换时机干净。

> 详细 Scheduler 分析见 Scheduler 专题文档（待编写）。
