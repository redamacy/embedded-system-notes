# FreeRTOS 信号量内核深度分析：osSemaphoreNew(1, 0, ...)

> 基于工程：FG23 BLE MicroStation — `FreeRTOSEntry.c:442`
>
> 源码：Gecko SDK 4.1.2 / FreeRTOS Kernel / CMSIS-RTOS2 Wrapper

---

## 1. 模块作用

```c
// FreeRTOSEntry.c:442
FG14ReceiveSemaphoreHandle = osSemaphoreNew(1, 0, &FG14ReceiveSemaphore_attributes);
```

**功能：** 创建一个二值信号量，初始"无令牌"。

**运行时行为：**

```
FG14_Receive_Task                        RAIL 回调 (app_process.c:402)
┌──────────────────┐                    ┌──────────────────────────┐
│ osSemaphoreAcquire│ ──── 阻塞 ────→   │ osSemaphoreRelease       │
│ (portMAX_DELAY)   │                   │ (RF 数据到达时)           │
│                  │ ←─── 唤醒 ────    │                          │
│ 处理 RF 数据     │                    │                          │
└──────────────────┘                    └──────────────────────────┘
```

**设计目的：** 任务在没有 RF 数据时完全不消耗 CPU（Blocked 态），只在 RAIL ISR 回调释放信号量后才被唤醒。这是 FreeRTOS 里最经典的 **中断→任务同步** 模式。

---

## 2. 调用链

```
osSemaphoreNew(1, 0, &attr)                         // CMSIS-RTOS2 wrapper
  ├── max_count==1 → 走二值信号量路径
  ├── xSemaphoreCreateBinary()                       // FreeRTOS API
  │     └── xQueueGenericCreate(
  │           uxQueueLength  = 1,                     // "队列长度" = 1
  │           uxItemSize     = queueSEMAPHORE_QUEUE_ITEM_LENGTH = 0,  // 不存数据!
  │           ucQueueType    = queueQUEUE_TYPE_BINARY_SEMAPHORE
  │         )
  │           │
  │           ├── pvPortMalloc(sizeof(Queue_t) + uxQueueLength * uxItemSize)
  │           │   = pvPortMalloc(sizeof(Queue_t) + 0)  // 只分配控制块!
  │           │
  │           └── prvInitialiseNewQueue(1, 0, storage, type, pxNewQueue)
  │                 ├── pcHead = (int8_t *)pxNewQueue  // 指向自身(无数据区)
  │                 ├── uxLength = 1
  │                 ├── uxItemSize = 0                 // 关键：item size = 0
  │                 ├── xQueueGenericReset()
  │                 │     ├── pcWriteTo = pcHead
  │                 │     ├── u.xQueue.pcTail = pcHead + (uxLength * uxItemSize) = pcHead
  │                 │     ├── u.xQueue.pcReadFrom = pcTail - uxItemSize = pcHead
  │                 │     ├── uxMessagesWaiting = 0    // ★ 初始计数 = 0
  │                 │     ├── xTasksWaitingToSend → 初始化空链表
  │                 │     └── xTasksWaitingToReceive → 初始化空链表
  │                 └── ucQueueType = queueQUEUE_TYPE_BINARY_SEMAPHORE
  │
  ├── initial_count == 0 → 跳过 xSemaphoreGive()
  └── 返回 (osSemaphoreId_t)hSemaphore
```

**关键结论：** Semaphore 就是一个 `uxItemSize=0` 的 Queue。`uxMessagesWaiting` 就是"信号量计数"。

---

## 3. 内核发生了什么：Queue_t 结构体实例化

`osSemaphoreNew(1,0)` 在堆上创建的结构体 `Queue_t` 如下：

```
┌──────────────────────────────────────────────────────┐
│                    Queue_t (88+ bytes)                │
├──────────────────────────────────────────────────────┤
│ pcHead ─────────→ 指向自身 (因为 uxItemSize=0)        │
│ pcWriteTo ──────→ = pcHead                           │
│                                                      │
│ u.xQueue:                               ← ★ 初始化时访问 xQueue 侧   │
│   pcTail             = pcHead                          │
│   pcReadFrom         = pcHead                          │
│                                                       │
│   同一块内存的 xSemaphore 视角 (仅在 mutex 操作时使用):   │
│     xMutexHolder       = NULL                          │
│     uxRecursiveCallCount = 0                            │
│                                                        │
│ ⚠️ 即使 ucQueueType == queueQUEUE_TYPE_BINARY_SEMAPHORE, │
│ prvInitialiseNewQueue → xQueueGenericReset 统一走       │
│ u.xQueue 路径初始化 pcTail/pcReadFrom。                  │
│ Semaphore 不"另起炉灶"，它复用 Queue 的全部操作原语。     │
│                                                      │
│ xTasksWaitingToSend:                                 │
│   uxNumberOfItems   = 0                              │
│   pxIndex ─────────→ &xListEnd                       │
│   xListEnd:                                          │
│     pxNext ────────→ &xListEnd   ← 空链表，自己指自己  │
│     pxPrevious ────→ &xListEnd                       │
│                                                      │
│ xTasksWaitingToReceive:                              │
│   uxNumberOfItems   = 0                              │
│   pxIndex ─────────→ &xListEnd                       │
│   xListEnd: (同上，空链表)                             │
│                                                      │
│ uxMessagesWaiting   = 0       ★ 初始计数 = 0          │
│ uxLength            = 1       ★ 最大容量 = 1          │
│ uxItemSize          = 0       ★ 不存储任何 data       │
│ cRxLock             = queueUNLOCKED (-1)             │
│ cTxLock             = queueUNLOCKED (-1)             │
│ ucStaticallyAllocated = pdFALSE                      │
└──────────────────────────────────────────────────────┘
```

**为什么 Semaphore 本质是 Queue？**

| Queue 特性 | Semaphore 用法 |
|------------|---------------|
| `uxMessagesWaiting` | 信号量当前计数 |
| `uxLength` | 最大计数 (max_count) |
| `uxItemSize` | **0** — 信号量不搬运数据，只传递"事件" |
| `xTasksWaitingToReceive` | 等待 take 信号量的阻塞任务链表 |
| `xTasksWaitingToSend` | 等待 give 信号量的阻塞任务链表（计数满时） |
| Give/Send | `prvCopyDataToQueue()` → `uxMessagesWaiting++` |
| Take/Receive | `uxMessagesWaiting--` |

**不分配数据存储区** — 信号量不需要 `pcHead → ... → pcTail` 的环形缓冲。这就是为什么 `pvPortMalloc(sizeof(Queue_t) + 0)` 只分配控制块。

---

### 3.1 深入：xTasksWaitingToSend 与 xTasksWaitingToReceive

这两个链表是 Queue/Semaphore 的 **阻塞调度核心**。理解它们，就理解了 FreeRTOS 任务间同步的底层。

#### 3.1.1 List_t 数据结构

```c
// FreeRTOS list.h — 内核最小通用数据结构

// 链表头 (嵌入在 Queue_t 中)
typedef struct xLIST {
    volatile UBaseType_t uxNumberOfItems;   // 链表中有几个节点
    ListItem_t          *pxIndex;           // 遍历游标 (指向当前遍历位置)
    MiniListItem_t       xListEnd;          // ★ 哨兵节点 — 不是真正的任务
} List_t;

// 完整节点 (存在 TCB 里: xEventListItem / xStateListItem)
struct xLIST_ITEM {
    TickType_t            xItemValue;       // ★ 排序键值 — 按优先级或 tick 排序
    struct xLIST_ITEM    *pxNext;           // 双向链表 → 后继
    struct xLIST_ITEM    *pxPrevious;       // 双向链表 → 前驱
    void                 *pvOwner;          // ★ 指向拥有者 TCB
    struct xLIST         *pvContainer;      // 反向指向所属的 List_t
};

// 迷你节点 (只用于 xListEnd — 不存 pvOwner/pvContainer)
struct xMINI_LIST_ITEM {
    TickType_t            xItemValue;
    struct xLIST_ITEM    *pxNext;
    struct xLIST_ITEM    *pxPrevious;
};
```

**关键设计：`xListEnd` 是一个不能移走的"锚"。** 它永远在链表中，双向循环链表的空/非空判断靠它：

```
空链表:                              非空链表 (1个节点):
┌──────────────┐                     ┌──────────────┐
│  xListEnd    │                     │  xListEnd    │
│  pxNext ───────┐                   │  pxNext ──────────┐
│  pxPrev ───┐   │                   │  pxPrev ──────┐   │
└────────────│───┘                   └───────────────│───┘
             │   ┌─────────────────┐                  │   ┌──────────────────────┐
             └──→│ xListEnd (自己) │                  │   │ xEventListItem (TCB) │
             ←───│                 │                  │   │ xItemValue = 15(pri) │
                 └─────────────────┘                  │   │ pxNext ──────────┐   │
                                                      │   │ pxPrev ───┐      │   │
                                                      │   │ pvOwner=TCB      │   │
                                                      │   └──────────│───────┘   │
                                                      └──────────────┘           │
                                                          ↑                      │
                                                          └──────────────────────┘
                                                               (双向闭环)
```

#### 3.1.2 xTasksWaitingToReceive — 等"Take"的任务链表

**谁挂在这里？** 调用 `osSemaphoreAcquire()` / `xQueueReceive()` 时，`uxMessagesWaiting == 0`（队列/信号量为空），且 `xTicksToWait > 0` 的任务。

**代码路径：** 见第 5 节 `vTaskPlaceOnEventList(&pxQueue->xTasksWaitingToReceive, ...)`。

**排序规则：按任务优先级降序**（`xItemValue = uxPriority`）→ 优先级最高的任务排在最前面，最先被唤醒。

**工程场景：FG14_Receive_Task** 在 `uxMessagesWaiting == 0` 时调用 `osSemaphoreAcquire(portMAX_DELAY)`：

```
初始 (FG14_Receive_Task 阻塞前):
  xTasksWaitingToReceive:
    uxNumberOfItems = 0
    pxIndex → &xListEnd
    xListEnd.pxNext → &xListEnd    ← 空，自己指自己

阻塞后:
  xTasksWaitingToReceive:
    uxNumberOfItems = 1
    pxIndex → &xListEnd
    xListEnd.pxNext → FG14_Receive_Task.xEventListItem
      .xItemValue  = 15  (优先级)
      .pvOwner     → &FG14_Receive_TCB
      .pvContainer → &xTasksWaitingToReceive
    xListEnd.pxPrev ← 同上
```

**唤醒时机：** `osSemaphoreRelease()` → `xQueueGenericSend()` → 检测到 `!listLIST_IS_EMPTY(&xTasksWaitingToReceive)` → `xTaskRemoveFromEventList()` → 从链表中取出最高优先级的 TCB → 从 DelayedList 摘除 → 插入 ReadyList。

**多任务等待时：** 假设同时有 prio 15 和 prio 13 两个任务在等：

```
xTasksWaitingToReceive (按优先级降序排列):
  xListEnd → Task(15).xEventListItem → Task(13).xEventListItem → xListEnd (闭环)
              ↑ 头部 — 最先被唤醒        ↑ 尾部
```

#### 3.1.3 xTasksWaitingToSend — 等"Give"的任务链表

**谁挂在这里？** 调用 `osSemaphoreRelease()` / `xQueueSend()` 时，队列已满（`uxMessagesWaiting == uxLength`），且 `xTicksToWait > 0` 的任务。

**对于你的二值信号量 `(max_count=1)`：** 当 `uxMessagesWaiting` 已经从 0 变成 1（已被 Give 过一次但还没被 Take），此时再有人 Give 且愿意等待，就会挂入此链表。

**但在你的工程中实际不触发：** 因为所有 `osSemaphoreRelease` 调用都在 ISR 中（`xTicksToWait=0`），不会阻塞等待。所以 `xTasksWaitingToSend` 始终保持**空链表**。

```
你的工程 4 个信号量的 xTasksWaitingToSend — 始终为空:

  xTasksWaitingToSend:
    uxNumberOfItems = 0
    xListEnd.pxNext → &xListEnd     ← 无人等待 Give
```

**何时会用到：** 如果某个 **任务**（非 ISR）调用 `osSemaphoreRelease()` 但信号量已满，且指定了等待超时 `xTicksToWait > 0`，该任务就会挂到这里等待被 Take 清出空间。

#### 3.1.4 两个链表的关系

```
                    Queue_t (信号量)
                   ┌─────────────────────────────┐
                   │ uxMessagesWaiting = 0        │  ← 当前计数
                   │ uxLength = 1                 │  ← 容量上限
                   │                              │
GIVE 方向 ←──      │ xTasksWaitingToSend           │      ──→ TAKE 方向
(放令牌)           │   (空 — 无人等待 Give)         │           (取令牌)
                   │                              │
                   │ xTasksWaitingToReceive        │
                   │   Task(15).xEventListItem     │  ← 在等令牌的任务们
                   │   Task(13).xEventListItem     │
                   └─────────────────────────────┘

规则:
  • uxMessagesWaiting == 0  → Take 方阻塞 → 挂入 xTasksWaitingToReceive
  • uxMessagesWaiting >= uxLength → Give 方阻塞 → 挂入 xTasksWaitingToSend
  • Give 成功后 → 优先唤醒 xTasksWaitingToReceive (如果有)
  • Take 成功后 → 优先唤醒 xTasksWaitingToSend (如果有)
```

**设计精妙：** FreeRTOS 用同一个 `List_t` 结构 + 同一套 `vListInsert`/`uxListRemove` 操作原语，统一了消息队列的"满阻塞"和信号量的"空阻塞"两种互逆的阻塞场景。代码复用率接近 100%。两个链表配合 `uxMessagesWaiting` 计数器，天然形成了一个 **生产者-消费者阻塞队列**。

---

### 3.2 创建过程逐行追踪：每一步，每一个值

从 `FG14ReceiveSemaphoreHandle = osSemaphoreNew(1, 0, ...)` 开始，追踪到两个链表初始化的最后一条汇编。

#### 3.2.1 调用链概览

```
osSemaphoreNew(1, 0, &attr)                    // CMSIS-RTOS2 wrapper
  └── xSemaphoreCreateBinary()                  // semphr.h 内联宏
        └── xQueueGenericCreate(               // queue.c
              uxQueueLength = 1,
              uxItemSize    = 0,                // ← queueSEMAPHORE_QUEUE_ITEM_LENGTH
              ucQueueType   = queueQUEUE_TYPE_BINARY_SEMAPHORE
            )
              │
              ├── pvPortMalloc(sizeof(Queue_t) + 1*0)  // 只分配控制块，无数据区
              │     ↓ 返回一块 88+ 字节的堆内存，假设地址为 0x2000_1000
              │
              └── prvInitialiseNewQueue(1, 0, 0x2000_1000, BINARY_SEMAPHORE, pxNewQueue)
                    │
                    ├── pxNewQueue->pcHead     = (int8_t *)pxNewQueue    // = 0x2000_1000
                    ├── pxNewQueue->uxLength   = 1
                    ├── pxNewQueue->uxItemSize = 0
                    └── xQueueGenericReset(pxNewQueue, pdTRUE)
                          │
                          ├── vListInitialise(&pxQueue->xTasksWaitingToSend)
                          ├── vListInitialise(&pxQueue->xTasksWaitingToReceive)
                          └── 初始化 pcWriteTo, u.xQueue.pcTail, u.xQueue.pcReadFrom,
                              uxMessagesWaiting=0, cRxLock=-1, cTxLock=-1
```

#### 3.2.2 核心：vListInitialise 逐条分析

`xQueueGenericReset` 中对两个链表的初始化是**完全对称**的。以 `xTasksWaitingToSend` 为例：

```c
// 调用: vListInitialise(&pxQueue->xTasksWaitingToSend);
//
// 参数: pxList = &pxQueue->xTasksWaitingToSend
//       即指向 Queue_t 内 xTasksWaitingToSend 字段的指针
```

```c
// ★ 语句 1
pxList->pxIndex = (ListItem_t *) &(pxList->xListEnd);
```

**效果：** `pxIndex` 指针指向 `xListEnd` 节点自身。
**为什么指向 xListEnd？** `pxIndex` 是遍历游标，`vListInsert` 用它确定插入位置。空链表时，唯一可指向的就是哨兵。实际插入操作不看 `pxIndex`，而是用 `xItemValue` 排序——但 `pxIndex` 必须非 NULL。

```
此时:
  pxIndex ───→ &xListEnd   (即 &pxList->xListEnd)
```

```c
// ★ 语句 2
pxList->uxNumberOfItems = (UBaseType_t) 0U;
```

**效果：** 计数器清零——链表里没有任务节点的原因是还没有人阻塞在这个信号量上。

```c
// ★ 语句 3
pxList->xListEnd.xItemValue = (TickType_t) portMAX_DELAY;
```

**效果：** 哨兵节点的排序值设为最大值 `0xFFFF_FFFF`。
**为什么？** 任务按优先级降序排列（数值大排前面）。`xListEnd` 作为哨兵，值设为最大保证新节点按优先级插入时，哨兵永远在列表"末尾"（也是"开头"，因为是双向闭环）。这是一个巧妙的边界处理：任何任务的优先级 ≤ `configMAX_PRIORITIES`（通常 ≤ 32），远小于 `portMAX_DELAY`，所以任何新节点都会排在 `xListEnd` 的"前面"。

```c
// ★ 语句 4 — 关键！
pxList->xListEnd.pxNext = (ListItem_t *) &(pxList->xListEnd);
```

**效果：** 哨兵的 `pxNext` 指向哨兵自己。

```c
// ★ 语句 5 — 关键！
pxList->xListEnd.pxPrevious = (ListItem_t *) &(pxList->xListEnd);
```

**效果：** 哨兵的 `pxPrevious` 也指向哨兵自己。

#### 3.2.3 初始化完成后：两个链表的完整状态

`vListInitialise` 执行两次后，`xTasksWaitingToSend` 和 `xTasksWaitingToReceive` 的**每一个字段**如下：

```
     ╔══════════════════════════════════════════════════════════╗
     ║          xTasksWaitingToSend (sizeof = 20 bytes)          ║
     ╠══════════════════════════════════════════════════════════╣
     ║                                                            ║
     ║  uxNumberOfItems = 0                    ← 空，无等待者     ║
     ║                                                            ║
     ║  pxIndex ──────────────────────────┐                      ║
     ║                                     │  指向哨兵自身        ║
     ║  xListEnd:                          ↓                      ║
     ║    .xItemValue  = 0xFFFF_FFFF       (portMAX_DELAY)        ║
     ║    .pxNext      = &xListEnd  ←──┐   (指向自身)            ║
     ║    .pxPrevious  = &xListEnd  ←──┼── (指向自身)            ║
     ║                                   │                        ║
     ║    可视化:                         │                        ║
     ║         ┌──────────────────────┐   │                        ║
     ║         │     xListEnd         │   │                        ║
     ║         │  xItemValue=0xFFFF.. │   │                        ║
     ║         │  pxNext ─────────────┼───┘                        ║
     ║         │  pxPrev ─────────────┼───┐                        ║
     ║         └──────────────────────┘   │                        ║
     ║              ↑                     │                        ║
     ║              └─────────────────────┘                        ║
     ║               (自己指自己，闭环)                              ║
     ╚══════════════════════════════════════════════════════════╝

     ╔══════════════════════════════════════════════════════════╗
     ║        xTasksWaitingToReceive (sizeof = 20 bytes)         ║
     ╠══════════════════════════════════════════════════════════╣
     ║                                                            ║
     ║  uxNumberOfItems = 0                    ← 空，无等待者     ║
     ║                                                            ║
     ║  pxIndex ──────────────────────────┐                      ║
     ║                                     │                      ║
     ║  xListEnd:                          ↓                      ║
     ║    .xItemValue  = 0xFFFF_FFFF       (portMAX_DELAY)        ║
     ║    .pxNext      = &xListEnd  ←──┐   (指向自身)            ║
     ║    .pxPrevious  = &xListEnd  ←──┼── (指向自身)            ║
     ║                                   │                        ║
     ║    可视化: (与上面完全一样，空链表)    │                        ║
     ║         ┌──────────────────────┐   │                        ║
     ║         │     xListEnd         │   │                        ║
     ║         │  xItemValue=0xFFFF.. │   │                        ║
     ║         │  pxNext ─────────────┼───┘                        ║
     ║         │  pxPrev ─────────────┼───┐                        ║
     ║         └──────────────────────┘   │                        ║
     ║              ↑                     │                        ║
     ║              └─────────────────────┘                        ║
     ╚══════════════════════════════════════════════════════════╝
```

**此时整个 `Queue_t` 在内存中的完整布局（两个链表 + 队列控制字段）：**

```
地址          内容                       所属字段
───────────  ────────────────────────   ─────────────────────────
0x2000_1000  → 0x2000_1000              pcHead (指向Queue_t自身)
0x2000_1004  → 0x2000_1000              pcWriteTo
0x2000_1008  → ...                      u 联合体起始
               (xQueue.pcTail / xSemaphore.xMutexHolder — 复用同一内存)
0x2000_100C  → ...                      (xQueue.pcReadFrom / xSemaphore.uxRecursiveCallCount)
              
              ── xTasksWaitingToSend ──
0x2000_1010  0x0000_0000                uxNumberOfItems = 0
0x2000_1014  → &xListEnd (即 0x2000_1018)  pxIndex
0x2000_1018  0xFFFF_FFFF                xListEnd.xItemValue = portMAX_DELAY
0x2000_101C  → 0x2000_1018              xListEnd.pxNext → 自己
0x2000_1020  → 0x2000_1018              xListEnd.pxPrevious → 自己
              
              ── xTasksWaitingToReceive ──
0x2000_1024  0x0000_0000                uxNumberOfItems = 0
0x2000_1028  → &xListEnd (即 0x2000_102C)  pxIndex
0x2000_102C  0xFFFF_FFFF                xListEnd.xItemValue = portMAX_DELAY
0x2000_1030  → 0x2000_102C              xListEnd.pxNext → 自己
0x2000_1034  → 0x2000_102C              xListEnd.pxPrevious → 自己
              
              ── 队列控制 ──
0x2000_1038  0x0000_0000                uxMessagesWaiting = 0
0x2000_103C  0x0000_0001                uxLength = 1
0x2000_1040  0x0000_0000                uxItemSize = 0
0x2000_1044  0xFFFF_FFFF (-1)           cRxLock = queueUNLOCKED
0x2000_1048  0xFFFF_FFFF (-1)           cTxLock = queueUNLOCKED
...          ...                        (后续字段省略)
```

**关键观察：** 两个链表在 Queue_t 中**紧挨着存放**。每个 `List_t` 占 20 字节（`uxNumberOfItems` 4 + `pxIndex` 4 + `MiniListItem_t` 12）。两个链表在内存中的物理位置相邻，但逻辑上完全独立——分别管理"等 Take"和"等 Give"两个方向的任务。

---

### 3.3 Release (Give) 时的链表变化：完整路径

以工程实际场景：**RAIL 中断中 `osSemaphoreRelease(FG14ReceiveSemaphoreHandle)`**。

#### 3.3.1 场景：还没有任务在等 Take

这是最常见的情况——信号量刚创建，还没有任务调过 `osSemaphoreAcquire`。

```
调用前状态:
  uxMessagesWaiting = 0
  xTasksWaitingToSend    = 空
  xTasksWaitingToReceive = 空
```

**调用链：**

```
osSemaphoreRelease(sem)                          // CMSIS-RTOS2 (ISR 上下文中调用 xSemaphoreGiveFromISR)
  └── xSemaphoreGiveFromISR(sem, &xHigherPriorityTaskWoken)  // FreeRTOS ISR 版
        └── xQueueGenericSendFromISR(pxQueue, NULL, &xHigherPriorityTaskWoken, queueSEND_TO_BACK)
              // pxQueue->uxMessagesWaiting (0) < pxQueue->uxLength (1) → 可以 Give
              │
              ├── prvCopyDataToQueue()           // uxItemSize=0 → 不 memcpy
              │     └── uxMessagesWaiting = 1    // ★ 0 → 1
              │
              └── 检查 xTasksWaitingToReceive
                    └── uxNumberOfItems = 0 → 空，无人等待
                        → 不做任何唤醒
                        → 返回
```

**结果：**

```
调用后状态:
  uxMessagesWaiting = 1           ← ★ 多了一个"令牌"
  xTasksWaitingToSend    = 空     ← 未变
  xTasksWaitingToReceive = 空     ← 未变

                       仅计数器 +1
                   两个链表毫无变化
```

用图表示：

```
  Release 前:                            Release 后:

  ┌─ FG14ReceiveSemaphore ─┐           ┌─ FG14ReceiveSemaphore ─┐
  │ uxMessagesWaiting = 0   │           │ uxMessagesWaiting = 1   │  ← 变了
  │ uxLength = 1            │           │ uxLength = 1            │
  │                          │           │                          │
  │ Send 链表: 空            │           │ Send 链表: 空            │  ← 没变
  │   [xListEnd ←→ 自己]     │   ──→    │   [xListEnd ←→ 自己]     │
  │                          │           │                          │
  │ Receive 链表: 空          │           │ Receive 链表: 空          │  ← 没变
  │   [xListEnd ←→ 自己]     │           │   [xListEnd ←→ 自己]     │
  └──────────────────────────┘           └──────────────────────────┘
```

**如果此时再有第二次 Release 且 `xTicksToWait=0`：** `uxMessagesWaiting (1) >= uxLength (1)` → 满！返回失败，`uxMessagesWaiting` 保持 1。两个链表依然不变。

#### 3.3.2 场景：有任务在等 Take（Release 唤醒等待者）

这是系统运行中的正常路径：FG14_Receive_Task 已经调了 `osSemaphoreAcquire` 并阻塞，RAIL 中断到达。

```
调用前状态:
  uxMessagesWaiting = 0                              ← 无令牌
  xTasksWaitingToSend    = 空
  xTasksWaitingToReceive = [ FG14_Receive_Task(15) ]  ← 有一个等待者

  xTasksWaitingToReceive 链表展开:
    ┌──────────────┐
    │  xListEnd    │
    │  xItemValue  │ = 0xFFFF_FFFF (哨兵)
    │  pxNext ─────┼───→ ┌─────────────────────────┐
    │  pxPrev ─────┼──←──│ TCB.xEventListItem      │
    └──────────────┘     │  .xItemValue = 15        │  ← 按优先级排序
                          │  .pxNext ─────→ &xListEnd│
                          │  .pxPrev ─────→ &xListEnd│
                          │  .pvOwner  ───→ &TCB    │  ← 指向 FG14_Receive_Task
                          │  .pvContainer → &ReceiveList│
                          └─────────────────────────┘
```

**调用链（ISR 版本）：**

```
xQueueGenericSendFromISR(pxQueue, NULL, &xHigherPriorityTaskWoken, queueSEND_TO_BACK)
  │
  ├── uxMessagesWaiting (0) < uxLength (1) → 可以 Give
  │
  ├── prvCopyDataToQueue()
  │     └── uxMessagesWaiting = 1       // 0 → 1
  │
  └── if (!listLIST_IS_EMPTY(&pxQueue->xTasksWaitingToReceive))
      │   // uxNumberOfItems = 1, NOT empty!
      │
      └── xTaskRemoveFromEventList(&pxQueue->xTasksWaitingToReceive)
            │
            ├── ① 取出链表头部: pxUnblockedTCB = (TCB_t *)(xListEnd.pxNext->pvOwner)
            │      = FG14_Receive_Task 的 TCB
            │
            ├── ② uxListRemove(&pxUnblockedTCB->xEventListItem)
            │      ★ 从 xTasksWaitingToReceive 摘除 xEventListItem
            │      ★ uxNumberOfItems: 1 → 0
            │      ★ xListEnd.pxNext → &xListEnd, xListEnd.pxPrev → &xListEnd  (恢复空链表)
            │
            ├── ③ 因为 uxSchedulerSuspended == pdFALSE (调度器未挂起)
            │      uxListRemove(&pxUnblockedTCB->xStateListItem)
            │      ★ 从 DelayedList 摘除 xStateListItem
            │
            │      prvAddTaskToReadyList(pxUnblockedTCB)
            │      ★ 插入 ReadyList[15] (任务回到就绪态)
            │
            └── ④ pxUnblockedTCB->uxPriority(15) > pxCurrentTCB->uxPriority(?)
                  → 如果是 ISR 上下文，通常返回 pdTRUE，设置 *pxHigherPriorityTaskWoken
                  → ISR 返回时触发 PendSV 上下文切换
```

**链表变化过程：**

```
  ★ 步骤②之前 (uxListRemove 调用前):

  xTasksWaitingToReceive:                  TCB.xEventListItem:
  ┌────────────────────┐                  ┌──────────────────────┐
  │ uxNumberOfItems = 1│                  │ .xItemValue = 15     │
  │ pxIndex → ...      │                  │ .pxNext → &xListEnd  │
  │                    │   ←──────────→   │ .pxPrev → &xListEnd  │
  │ xListEnd:          │                  │ .pvOwner → &TCB      │
  │  .pxNext ──────────┼──→              │ .pvContainer → List  │
  │  .pxPrev ──────────┼──←              └──────────────────────┘
  └────────────────────┘

  ★ 步骤②之后 (uxListRemove 完成):

  xTasksWaitingToReceive:                  TCB.xEventListItem:
  ┌────────────────────┐                  ┌──────────────────────┐
  │ uxNumberOfItems = 0│                  │ .pxNext = ???        │ ← 被uxListRemove清
  │ pxIndex → &xListEnd│                  │ .pxPrev = ???        │
  │                    │    (断开连接)      │ .pvOwner → &TCB     │ ← pvOwner 不变
  │ xListEnd:          │                  │ .pvContainer = NULL  │ ← 被uxListRemove清
  │  .pxNext → &xListEnd│ ← 恢复自指     └──────────────────────┘
  │  .pxPrev → &xListEnd│
  └────────────────────┘

  同时: TCB.xStateListItem 从 DelayedList 移到 ReadyList[15]
```

**最终状态：**

```
Release 完成后:
  uxMessagesWaiting = 1           ← 计数器+1 (虽然立即被唤醒的任务消费)
  xTasksWaitingToSend    = 空     ← 未变
  xTasksWaitingToReceive = 空     ← ★ 从 1 项变回空 — 等待者被移走了

  FG14_Receive_Task: 从 Blocked 态 → Ready 态 (在 ReadyList[15] 中)
```

---

### 3.4 Acquire (Take) 时的链表变化：完整路径

以工程实际场景：**FG14_Receive_Task 调用 `osSemaphoreAcquire(FG14ReceiveSemaphoreHandle, portMAX_DELAY)`**。

#### 3.4.1 场景：信号量有令牌（uxMessagesWaiting > 0）

```
调用前状态:
  uxMessagesWaiting = 1            ← 有令牌（比如刚被 ISR Give 过）
  xTasksWaitingToSend    = 空
  xTasksWaitingToReceive = 空
```

**调用链：**

```
osSemaphoreAcquire(sem, portMAX_DELAY)
  └── xSemaphoreTake(sem, portMAX_DELAY)
        └── xQueueSemaphoreTake(pxQueue, portMAX_DELAY)
              │
              ├── taskENTER_CRITICAL()
              ├── uxSemaphoreCount = pxQueue->uxMessagesWaiting = 1
              ├── if (uxSemaphoreCount > 0)  → TRUE!
              │     │
              │     ├── pxQueue->uxMessagesWaiting = 1 - 1 = 0   // ★ 1 → 0
              │     │
              │     └── if (!listLIST_IS_EMPTY(&pxQueue->xTasksWaitingToSend))
              │           // uxNumberOfItems = 0 → 空，跳过
              │           // (没有人阻塞在 Give 上等待空间)
              │
              └── taskEXIT_CRITICAL()
              → 返回 pdPASS —— 立即成功，连阻塞都没进！
```

**结果：**

```
调用前:                                  调用后:
  uxMessagesWaiting = 1                    uxMessagesWaiting = 0    ← 令牌被消费
  Send 链表: 空                            Send 链表: 空             ← 未变
  Receive 链表: 空                          Receive 链表: 空          ← 未变

              没有任何链表被操作，全程只改了 uxMessagesWaiting
```

这正是信号量机制的"热路径"——大多数时候没有竞争，就是一个简单的计数器减一操作。两个链表在整个过程中连碰都没被碰。

#### 3.4.2 场景：信号量无令牌 + 任务阻塞（关键场景）

这是系统启动后的首次 Acquire——信号量刚创建，`uxMessagesWaiting = 0`，还没有人 Give。

```
调用前状态:
  uxMessagesWaiting = 0            ← 无令牌
  xTasksWaitingToSend    = 空
  xTasksWaitingToReceive = 空      ← 还没有任务在等
```

**调用链（逐函数展开，标出每一步对链表的影响）：**

```
xQueueSemaphoreTake(pxQueue, portMAX_DELAY)
  │
  ├── ① taskENTER_CRITICAL()           // 关中断
  │     uxSemaphoreCount = pxQueue->uxMessagesWaiting = 0
  │
  ├── ② if (uxSemaphoreCount > 0) → FALSE (0 > 0 is false)
  │     进入 else 分支 — 没有令牌，准备阻塞
  │
  ├── ③ if (xTicksToWait == 0) → FALSE (portMAX_DELAY != 0)
  │     if (xEntryTimeSet == pdFALSE) → 记录进入时间
  │
  ├── ④ taskEXIT_CRITICAL()            // 开中断！
  │     ★ 此时中断仍能 fire。如果在 vTaskSuspendAll 之前
  │     ISR 调了 xSemaphoreGiveFromISR，令牌会先被放入
  │     （cRxLock/cTxLock 机制处理这个竞态）
  │
  ├── ⑤ vTaskSuspendAll()              // 挂起调度器，但不断中断
  │
  ├── ⑥ prvLockQueue(pxQueue)          // cRxLock/cTxLock 设为 queueLOCKED_UNMODIFIED
  │
  ├── ⑦ if (prvIsQueueEmpty(pxQueue))  // uxMessagesWaiting 还是 0 吗？
  │     → TRUE（ISR 没在此期间 Give）
  │     │
  │     ├── ★ vTaskPlaceOnEventList(
  │     │         &pxQueue->xTasksWaitingToReceive,   // 目标：Receive 链表
  │     │         portMAX_DELAY                       // 永不超时
  │     │       )
  │     │     │
  │     │     ├── ⑦a. vListInsert(&xTasksWaitingToReceive, &pxCurrentTCB->xEventListItem)
  │     │     │
  │     │     │     ★ vListInsert 按 xItemValue 降序插入
  │     │     │     ★ xEventListItem.xItemValue = pxCurrentTCB->uxPriority = 15
  │     │     │     ★ 空链表 → 直接插入在 xListEnd 的"前面"（即 pxNext 方向）
  │     │     │
  │     │     │     插入后链表:
  │     │     │     ┌──────────────┐
  │     │     │     │  xListEnd    │
  │     │     │     │  xItemValue  │ = 0xFFFF_FFFF (哨兵)
  │     │     │     │  pxNext ─────┼───→ ┌─────────────────────────┐
  │     │     │     │  pxPrev ─────┼──←──│ TCB.xEventListItem      │
  │     │     │     └──────────────┘     │  .xItemValue = 15        │
  │     │     │                           │  .pxNext ─────→ &xListEnd│
  │     │     │                           │  .pxPrev ─────→ &xListEnd│
  │     │     │                           │  .pvOwner  ───→ &TCB    │
  │     │     │                           │  .pvContainer → &ReceiveList│
  │     │     │                           └─────────────────────────┘
  │     │     │
  │     │     │     uxNumberOfItems: 0 → 1
  │     │     │
  │     │     └── ⑦b. prvAddCurrentTaskToDelayedList(portMAX_DELAY, pdTRUE)
  │     │           ★ TCB.xStateListItem 从 ReadyList[15] 移到 DelayedList
  │     │           ★ 任务状态: Running → Blocked
  │     │
  │     ├── prvUnlockQueue(pxQueue)
  │     │
  │     └── if (xTaskResumeAll() == pdFALSE)  // 恢复调度器
  │           ★ 当前任务已不在 ReadyList → 调度器选新任务
  │           → portYIELD_WITHIN_API()
  │           → PendSV → 上下文切换到下一个最高优先级就绪任务
  │
  └── ★ 注意: 代码不会立刻返回到 FG14_Receive_Task
        因为 PendSV 已经把 CPU 切走了。下次被唤醒时会从
        portYIELD_WITHIN_API() 之后继续执行 for(;;) 循环，
        重新检查 uxMessagesWaiting，此时 > 0，走"有令牌"路径。
```

**链表变化总结：**

```

  Acquire 前（信号量为空，任务准备阻塞）:
  ═══════════════════════════════════════════════
  FG14_Receive_Task: Running 态 (pxCurrentTCB)
  ReadyList[15]:     [FG14_Receive_Task, ...]
  DelayedList:       空
  xTasksWaitingToReceive: 空 [xListEnd ←→ 自己]
  uxMessagesWaiting: 0


  Acquire 后（vTaskPlaceOnEventList 完成）:
  ═══════════════════════════════════════════════
  FG14_Receive_Task: Blocked 态 (在事件链表中)
  ReadyList[15]:     [其他任务]              ← FG14移除了
  DelayedList:       [FG14_Receive_Task]      ← STATE 链表
  xTasksWaitingToReceive: [FG14_Receive_Task] ← EVENT 链表
  uxMessagesWaiting: 0
  pxCurrentTCB:      → 下一个最高就绪任务      ← 已切换

  ★ 同一个 TCB 同时出现在两个链表中:
    • TCB.xStateListItem   → DelayedList (超时管理)
    • TCB.xEventListItem   → xTasksWaitingToReceive (事件等待)
```

#### 3.4.3 场景：Acquire 时 xTasksWaitingToSend 有人等

这个问题反过来了：假如信号量有令牌（`uxMessagesWaiting = 1`），Take 成功后计数器变成 0，此时检查 `xTasksWaitingToSend`。

```
调用前:
  uxMessagesWaiting = 1
  xTasksWaitingToSend = [某任务 TaskX 阻塞在 Give 上 — 信号量满时它想 Give]
                         (在你的工程中从不发生，但机制如此)
  xTasksWaitingToReceive = 空

xQueueSemaphoreTake:
  uxMessagesWaiting = 1 - 1 = 0        // 消费令牌
  检查 xTasksWaitingToSend → 不空！
  xTaskRemoveFromEventList(&xTasksWaitingToSend)
    → TaskX 从 xTasksWaitingToSend 摘除
    → TaskX.xStateListItem: DelayedList → ReadyList[priority]
    → TaskX 现在可以 Give 了（因为消费令牌后有了空间）

结果:
  uxMessagesWaiting = 0         (刚才 1→0, 然后被 TaskX Give → 1, 取决于调度顺序)
  但实际上 Take 代码返回前 uxMessagesWaiting 被 TaskX Give +1 → 回到 1
     → TaskX 的令牌"传递"给了下一个 Take 者
```

**你的工程中 `xTasksWaitingToSend` 永远为空——** 所有 Give/Release 来自 ISR，ISR 中不允许阻塞等待，所以等不到空间就直接返回失败，不会挂入 `xTasksWaitingToSend`。

---

## 4. initial_count=0 的含义

```c
// 在 cmsis_os2.c:1606
if ((hSemaphore != NULL) && (initial_count != 0U)) {
    if (xSemaphoreGive(hSemaphore) != pdPASS) {
        vSemaphoreDelete(hSemaphore);
        hSemaphore = NULL;
    }
}
```

**`initial_count=0` → 跳过 Give → `uxMessagesWaiting` 保持 0。**

这意味着：
- 创建后，任何 `osSemaphoreAcquire()` 调用都会因为 `uxMessagesWaiting==0` 而**立即进入阻塞**
- 必须先由**中断或其他任务** `osSemaphoreRelease()` 才能唤醒

工程中 4 个信号量全是 `(1, 0)`：

| 信号量 | 唤醒者 | 设计原因 |
|--------|--------|----------|
| FG14ReceiveSemaphore | RAIL RX 中断回调 | 无 RF 数据时任务不消耗 CPU |
| FG14SendSemaphore | TIMER1 / 缓冲区满 | 无待发送数据时阻塞 |
| BG22ReceiveSemaphore | GPIO SPI CS 中断 | 无 BLE 数据时阻塞 |
| SendMutexSemaphore | TIMER2 ACK 中断 | 无 ACK 时发送流程阻塞 |

**全部 `initial=0` 的设计动机：** 任务启动后应该立即阻塞，等待事件驱动。这不是"初始没有资源"，而是"事件尚未发生"。

---

## 5. Task Take 失败后如何进入阻塞态

```c
// FreeRTOSEntry.c:228
osSemaphoreAcquire(FG14ReceiveSemaphoreHandle, portMAX_DELAY);
```

### 5.1 完整内核路径

```
osSemaphoreAcquire(sem, portMAX_DELAY)                // CMSIS-RTOS2
  └── xSemaphoreTake(sem, portMAX_DELAY)              // FreeRTOS
        └── xQueueSemaphoreTake(pxQueue, portMAX_DELAY)
```

### 5.2 xQueueSemaphoreTake 内核逐行分析

```c
// queue.c:1474 → 你的场景：uxMessagesWaiting == 0, xTicksToWait == portMAX_DELAY

BaseType_t xQueueSemaphoreTake(QueueHandle_t xQueue, TickType_t xTicksToWait)
{
    for(;;)  // ← 可能会循环多次（被虚假唤醒后重新检查）
    {
        taskENTER_CRITICAL();  // 关中断，保护 uxMessagesWaiting
        {
            const UBaseType_t uxSemaphoreCount = pxQueue->uxMessagesWaiting;
            // ★ 第一次进入：uxSemaphoreCount == 0

            if(uxSemaphoreCount > 0)  // 有计数 → 立即成功，不会阻塞
            {
                pxQueue->uxMessagesWaiting = uxSemaphoreCount - 1;
                // ... 检查 xTasksWaitingToSend，有人等 give 就唤醒 ...
                taskEXIT_CRITICAL();
                return pdPASS;
            }
            else  // ★ 进入这个分支
            {
                if(xTicksToWait == 0)     // 不等 → 立即返回 errQUEUE_EMPTY
                    return errQUEUE_EMPTY;
                else if(xEntryTimeSet == pdFALSE)
                {
                    vTaskInternalSetTimeOutState(&xTimeOut);  // 记录进入时间
                    xEntryTimeSet = pdTRUE;
                }
            }
        }
        taskEXIT_CRITICAL();  // ★ 开中断（此时 RAIL 中断可能已经 fire 并 release 了信号量！）

        // ↓ 下面开始真正阻塞
        vTaskSuspendAll();         // 挂起调度器（但不断中断）
        prvLockQueue(pxQueue);     // 锁队列事件链表（ISR 不能直接操作，改通过 cRxLock/cTxLock 计数）

        if(xTaskCheckForTimeOut(&xTimeOut, &xTicksToWait) == pdFALSE)
        {
            if(prvIsQueueEmpty(pxQueue) != pdFALSE)  // ★ 再检查一次：count 还是 0?
            {
                // ★ 核心：把当前任务挂到信号量的"等待接收"事件链表
                vTaskPlaceOnEventList(
                    &(pxQueue->xTasksWaitingToReceive),  // 目标链表
                    xTicksToWait                          // portMAX_DELAY
                );
                prvUnlockQueue(pxQueue);

                if(xTaskResumeAll() == pdFALSE)  // 恢复调度器，触发任务切换
                {
                    portYIELD_WITHIN_API();       // ★ 调度器发现当前任务已不在就绪链表
                }                                 //    → 切换到最高优先级的就绪任务
            }
        }
        // ... 超时或循环重试 ...
    }
}
```

### 5.3 vTaskPlaceOnEventList 做了什么

```c
// tasks.c:3130
void vTaskPlaceOnEventList(List_t * const pxEventList, const TickType_t xTicksToWait)
{
    // 步骤 1：把当前 TCB 的 xEventListItem 插入信号量的 xTasksWaitingToReceive 链表
    //          按优先级排序 → 高优先级任务先被唤醒
    vListInsert(pxEventList, &(pxCurrentTCB->xEventListItem));

    // 步骤 2：把当前 TCB 的 xStateListItem 从 ReadyList 移到 DelayedList
    //         (portMAX_DELAY 不会真正超时，但还是要挂到延时链表等待被事件唤醒)
    prvAddCurrentTaskToDelayedList(xTicksToWait, pdTRUE);
}
```

---

## 6. TCB × List × Queue × Event List 联动分析

### 6.1 关键结构体关系

```
          TCB_t (FG14_Receive_Task)
         ┌─────────────────────────────┐
         │ pxTopOfStack                │  ← 任务栈指针（阻塞时保存上下文）
         │ uxPriority = 15             │  ← 最高任务优先级
         │                             │
         │ xStateListItem  ────────────┼──→ 在 ReadyList[15] 或 DelayedList
         │   .pvOwner ────────────────→│    自己 (TCB_t*)
         │   .xItemValue               │    优先级或唤醒 tick
         │   .pxNext / .pxPrevious     │    链表指针
         │                             │
         │ xEventListItem ─────────────┼──→ 在 xTasksWaitingToReceive
         │   .pvOwner ────────────────→│    自己 (TCB_t*)
         │   .xItemValue               │    优先级
         │   .pxNext / .pxPrevious     │    链表指针
         │                             │
         │ pxStack                     │  ← 栈底
         └─────────────────────────────┘

              Queue_t (FG14ReceiveSemaphore)
             ┌─────────────────────────────┐
             │ pcHead ───→ self            │
             │ uxMessagesWaiting = 0        │  ← 当前无令牌
             │ uxLength = 1                 │
             │ uxItemSize = 0               │
             │                             │
             │ xTasksWaitingToReceive       │  ← ★ Task 的 xEventListItem 挂在这里
             │   ┌─────────────────────┐   │
             │   │ xListEnd            │   │
             │   │ pxNext ─→ xEventListItem │
             │   │ pxPrev ─→ xEventListItem │
             │   │ xItemValue = 15 ←─┼──────│── 优先级 15
             │   │ pvOwner ──→ TCB ──┘      │
             │   └─────────────────────┘   │
             │                             │
             │ xTasksWaitingToSend          │  ← 空（无人等待 give）
             └─────────────────────────────┘

              Global Scheduler Data (tasks.c)
             ┌─────────────────────────────────────────┐
             │ pxReadyTasksLists[15]                   │
             │   ┌──────────┐                          │
             │   │ xListEnd │  ← FG14_Receive_Task     │
             │   └──────────┘    阻塞前在这里            │
             │                                         │
             │ xDelayedTaskList                        │
             │   ┌──────────┐                          │
             │   │ xListEnd │  ← FG14_Receive_Task     │
             │   └──────────┘    阻塞后移到这里           │
             │                                         │
             │ pxCurrentTCB ──→ ...                    │
             └─────────────────────────────────────────┘
```

### 6.2 阻塞前后的状态变化

```
阻塞前：
  pxReadyTasksLists[15] 包含 FG14_Receive_Task.xStateListItem
  xDelayedTaskList      空
  xTasksWaitingToReceive 空
  pxCurrentTCB          → FG14_Receive_Task  ← 正在执行

vTaskPlaceOnEventList() 执行后：
  xTasksWaitingToReceive 包含 FG14_Receive_Task.xEventListItem  ← 按优先级排序
  pxReadyTasksLists[15] 移除了 FG14_Receive_Task.xStateListItem
  xDelayedTaskList      包含 FG14_Receive_Task.xStateListItem   ← portMAX_DELAY
  pxCurrentTCB          → 切换为下一个最高优先级就绪任务
```

**设计精妙之处：TCB 有两个 ListItem**

| ListItem | 用途 | 所在链表 |
|----------|------|----------|
| `xStateListItem` | 表示任务状态 | ReadyList / DelayedList / SuspendedList |
| `xEventListItem` | 表示在等什么事件 | Queue的 xTasksWaitingToReceive / xTasksWaitingToSend |

这两个 ListItem 让一个 TCB 可以**同时**出现在 DelayedList（超时管理）和 EventList（事件等待）中。唤醒时从两个链表中摘除即可。

---

## 7. Scheduler 参与调度：xTaskResumeAll 和上下文切换

```c
// queue.c:1628 — 阻塞路径的最后一步
if(xTaskResumeAll() == pdFALSE)
{
    portYIELD_WITHIN_API();  // 触发 PendSV，切换到新任务
}
```

`xTaskResumeAll()` 解冻调度器后，检查是否有更高优先级任务就绪（这里当前任务已经移出 ReadyList，所以一定有）。返回 `pdFALSE` 意味着需要上下文切换。

**上下文切换流程：**

```
portYIELD_WITHIN_API()
  └── portYIELD()
        └── PendSV 挂起 → 硬件自动入栈 (R0-R3, R12, LR, PC, xPSR)
              → PendSV_Handler:
                  保存 R4-R11 到当前任务栈
                  pxCurrentTCB->pxTopOfStack = SP
                  选最高优先级就绪任务 → pxCurrentTCB = 新任务
                  SP = 新任务->pxTopOfStack
                  恢复 R4-R11
                  硬件自动出栈 → 跳转到新任务的 PC
```

**你的工程场景：** FG14_Receive_Task (prio 15) 阻塞后，如果 FG14_Send_Task (prio 14) 或 BG22_Receive_Task (prio 13) 在就绪态，CPU 会切换到它们。

---

## 8. Release 唤醒路径：中断→信号量 Give→任务唤醒

```c
// app_process.c:402 — RAIL 事件回调中
stat = osSemaphoreRelease(FG14ReceiveSemaphoreHandle);
```

### 8.1 调用链

```
osSemaphoreRelease(sem)                   // CMSIS-RTOS2
  └── xSemaphoreGive(sem)                // FreeRTOS
        └── xQueueGenericSend(pxQueue,
              pvItemToQueue = NULL,       // ★ 信号量不传数据
              xTicksToWait  = 0,          // 非阻塞
              xCopyPosition = queueSEND_TO_BACK)
```

### 8.2 xQueueGenericSend 内核路径 (信号量 Give)

```c
// queue.c:784
for(;;)
{
    taskENTER_CRITICAL();
    {
        // 信号量：uxMessagesWaiting(0) < uxLength(1) → TRUE
        if(pxQueue->uxMessagesWaiting < pxQueue->uxLength)
        {
            // prvCopyDataToQueue 对信号量（uxItemSize==0）：
            //   不 memcpy 任何数据
            //   只做：pxQueue->uxMessagesWaiting = uxMessagesWaiting + 1
            //         → uxMessagesWaiting 从 0 变成 1
            xYieldRequired = prvCopyDataToQueue(pxQueue, NULL, queueSEND_TO_BACK);

            // ★ 检查是否有人在等 receive
            if(listLIST_IS_EMPTY(&(pxQueue->xTasksWaitingToReceive)) == pdFALSE)
            {
                // ★ 有！FG14_Receive_Task 的 xEventListItem 在这里
                if(xTaskRemoveFromEventList(&(pxQueue->xTasksWaitingToReceive)) != pdFALSE)
                {
                    // 被唤醒的任务优先级 15 > 当前上下文优先级
                    // → 需要调度
                    queueYIELD_IF_USING_PREEMPTION();
                }
            }
            taskEXIT_CRITICAL();
            return pdPASS;
        }
    }
}
```

### 8.3 xTaskRemoveFromEventList 做了什么

```c
// tasks.c:3209  — 在 critical section 中调用
BaseType_t xTaskRemoveFromEventList(const List_t * const pxEventList)
{
    // 1. 从事件链表中取出最高优先级任务的 TCB
    pxUnblockedTCB = listGET_OWNER_OF_HEAD_ENTRY(pxEventList);
    //    = FG14_Receive_Task 的 TCB

    // 2. 从 xTasksWaitingToReceive 摘除 xEventListItem
    uxListRemove(&(pxUnblockedTCB->xEventListItem));

    // 3. 如果调度器没锁
    if(uxSchedulerSuspended == pdFALSE)
    {
        // 4. 从 DelayedList 摘除 xStateListItem
        uxListRemove(&(pxUnblockedTCB->xStateListItem));

        // 5. 插入 ReadyList[15]
        prvAddTaskToReadyList(pxUnblockedTCB);
    }
    else
    {
        // 调度器被锁 → 先挂到 xPendingReadyList，等调度器解锁时再处理
        vListInsertEnd(&xPendingReadyList, &(pxUnblockedTCB->xEventListItem));
    }

    // 6. 被唤醒的任务优先级 > 当前运行的任务？ → 返回 pdTRUE，触发调度
    if(pxUnblockedTCB->uxPriority > pxCurrentTCB->uxPriority)
        return pdTRUE;  // → queueYIELD_IF_USING_PREEMPTION()
}
```

### 8.4 在 ISR 中调用的特殊路径

你工程里的 `app_process.c:402` 在 RAIL 回调中调用，属于**中断上下文**。CMSIS-RTOS2 会检测到 ISR 环境，走 `xSemaphoreGiveFromISR()`：

```c
// cmsis_os2.c 中 osSemaphoreRelease 检测 ISR → 调用 xSemaphoreGiveFromISR()
// 差异：不阻塞、使用 portYIELD_FROM_ISR() 触发 PendSV
```

**这意味着：** RAIL 中断中 Release → 信号量计数 +1 → FG14_Receive_Task 回到 ReadyList[15] → ISR 返回时，如果 FG14_Receive_Task 比被中断的代码优先级高（15 是最高任务优先级，所以一定），PendSV 立即切换。

---

## 9. 完整运行时序图

```
时间 →

FG14_Receive_Task (prio 15):  运行 ──→ Take失败 ──→ BLOCKED ──────────────────→ 被唤醒 ──→ RUNNING(FG14ReceiveProcess)
                                    │                                          │
                                    │ vTaskPlaceOnEventList()                  │ xTaskRemoveFromEventList()
                                    │ TCB.xEventListItem →                     │ TCB.xEventListItem 摘除
                                    │   xTasksWaitingToReceive                 │ TCB.xStateListItem:
                                    │ TCB.xStateListItem:                      │   DelayedList → ReadyList[15]
                                    │   ReadyList[15] → DelayedList            │
                                    │                                          │
FG14_Send_Task (prio 14):        READY ──→ RUNNING ──────────────────────→ READY
                                    │ (调度器切换到它)                          │
                                    │                                          │
RAIL ISR (prio 2 HW):          ───────────────────────────  RF 数据到达 ──→ osSemaphoreRelease()
                                                                               uxMessagesWaiting: 0→1
                                                                               唤醒 FG14_Receive_Task
                                                                               PendSV 挂起
                                                                               ISR 返回 → PendSV →
                                                                               FG14_Receive_Task 运行
```

---

## 10. 为什么这样设计

### 10.1 Semaphore = Queue (uxItemSize=0)

**不重新发明轮子。** 信号量的所有操作（阻塞、唤醒、优先级排序、超时）与队列 95% 一样。FreeRTOS 用同一套 Queue 机制统一了：
- 消息队列（有数据搬运）
- 二值信号量（计数 0/1）
- 计数信号量（计数 0~N）
- 互斥量（信号量 + 优先级继承）

这使内核代码量极大减少，bug 面也缩小。

### 10.2 双 ListItem 设计 (xStateListItem + xEventListItem)

TCB 中两个 ListItem 让一个任务可以同时处于两个链表中：
- **xStateListItem** 在 DelayedList → 响应超时唤醒
- **xEventListItem** 在 Queue 的事件链表 → 响应事件唤醒

不论哪种方式先到期，任务都能被正确唤醒并从两个链表中摘除。

### 10.3 Critical Section + Scheduler Suspend 双层保护

```c
taskENTER_CRITICAL();    // 关中断 — 保护极短的 queue 状态检查
// ...检查 uxMessagesWaiting...
taskEXIT_CRITICAL();     // 开中断 — 此时 ISR 可以 Give

vTaskSuspendAll();       // 调度器停 — 但中断不关！ISR 仍能 Give/唤醒任务
prvLockQueue(pxQueue);   // 锁事件链表 — ISR Give 时通过 cRxLock/cTxLock 记录
// ...操作事件链表...
prvUnlockQueue(pxQueue);
xTaskResumeAll();        // 恢复调度器 — 处理 pending ready list
```

**精妙之处：** 中断线程（vTaskSuspendAll）保护任务状态操作不被打断，但 ISR 仍能 fire。ISR 中的 Give 通过 `cRxLock/cTxLock` 计数器"延迟"处理事件链表的修改。这避免了长时间关中断。

### 10.4 initial_count=0 的设计哲学

对于事件驱动系统：**任务应该等待事件，而不是轮询。** `initial=0` 确保：
- 任务创建后就阻塞，不浪费 CPU 时间
- 只有真正有数据时才运行
- 配合 `portMAX_DELAY`，任务不超时，完全事件驱动

---

## 11. 工程价值

**这条 `osSemaphoreNew(1, 0, ...)` 背后是：**
- 一个 `sizeof(Queue_t)` ≈ 88 字节的内核对象
- 两个事件链表（等待 Give / 等待 Take），每个 20 字节
- TCB 中的两个 ListItem，每个 20 字节
- 一套从 ISR→信号量→任务 的完整唤醒链

**你的 FG23 工程用 4 个这样的信号量串起了整个数据流：**

```
GPIO ISR → BG22ReceiveSemaphore → BG22_Receive_Task → 填充缓冲区
TIMER1 ISR → FG14SendSemaphore → FG14_Send_Task → 构建/发送 RF 帧
RAIL RX ISR → FG14ReceiveSemaphore → FG14_Receive_Task → 处理下行数据
TIMER2 ISR → SendMutexSemaphore → 发送流程 ACK 确认
```

全部 4 个信号量都是 `(1, 0)` — 统统"事件驱动，初始阻塞"。这就是实时嵌入式系统的核心范式：**CPU 只在有事做的时候运行，其余时间全在 IDLE（可进入低功耗 sleep）。**
