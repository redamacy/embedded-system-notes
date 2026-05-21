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
│ u.xSemaphore:                                        │
│   xMutexHolder       = NULL    (非 mutex 不用)        │
│   uxRecursiveCallCount = 0                            │
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
