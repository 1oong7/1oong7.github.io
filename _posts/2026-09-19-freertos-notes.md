---
layout: post
title: FreeRTOS 学习笔记
date: 2026-09-19 21:00:35 +0800
description: 整理 FreeRTOS 任务状态、调度、队列、信号量、任务通知、中断接口、事件组与内存管理等核心机制。
tags: [FreeRTOS, RTOS, 嵌入式, 任务调度]
categories: [嵌入式学习]
published: true
toc:
  sidebar: left
---

## 四种状态

FreeRTOS 中通常把任务分为四种基本状态：

1. 运行态 Running：运行态表示任务当前正在占用 CPU，其代码正在被处理器执行
2. 就绪态 Ready：就绪态表示任务已经具备运行条件，但当前没有获得 CPU
3. 阻塞态 Blocked：阻塞态表示任务正在等待某个条件，因此暂时不参与 CPU 竞争
4. 挂起态 Suspended：挂起态表示任务被明确暂停，不再参与调度，而且通常不会因为时间到期而自动恢复
   ![FreeRTOS 任务状态转换]({{ '/assets/img/blog/freertos/task-states.png' | relative_url }})

```
                         高优先级任务就绪
              ┌──────────────────────────────┐
              │                              ↓
就绪态 Ready ───────被调度──────→ 运行态 Running
     ↑                              │
     │                              ├─ 等待时间/事件 ─→ 阻塞态 Blocked
     │                              │                     │
     │                              │                     └─ 条件满足/超时
     │                              │
     │                              └─ 主动/被动挂起 ─→ 挂起态 Suspended
     │                                                    │
     └──────────────────────恢复任务───────────────────────┘
```

| 对比项               | 阻塞态 Blocked       | 挂起态 Suspended   |
| -------------------- | -------------------- | ------------------ |
| 产生原因             | 等待时间或事件       | 显式调用挂起操作   |
| 是否有超时机制       | 可以有               | 通常没有           |
| 时间到后能否自动恢复 | 可以                 | 不可以             |
| 事件到来能否自动恢复 | 可以                 | 通常不可以         |
| 恢复方式             | 条件满足或超时       | `vTaskResume()` 等 |
| 典型用途             | 等队列、信号量、通知 | 强制暂停某个任务   |

## 优先级制度和抢占式调度

调度器始终选择“处于就绪态的最高优先级任务”运行；更高优先级任务一旦就绪，可以抢占当前低优先级任务。

### 1. FreeRTOS 任务优先级

任务创建时需要指定优先级：

```c
xTaskCreate(
    SensorTask,
    "Sensor",
    256,
    NULL,
    3,              // 任务优先级
    &sensorHandle
);
```

优先级的有效范围由下面的配置决定：

```c
#define configMAX_PRIORITIES  8
```

此时可以使用的优先级为：`0～7`
FreeRTOS 任务优先级的规则是：`数值越大，任务优先级越高 `

### 2. 抢占式调度

抢占式调度通过下面的配置启用：

```c
#define configUSE_PREEMPTION  1
```

在抢占式调度中，当一个比当前任务优先级更高的任务进入就绪态时，调度器可以立即暂停当前任务，转而执行高优先级任务。

### 3. 抢占式和协作式调度的区别

#### 抢占式调度

```c
#define configUSE_PREEMPTION  1
```

特点：

- 高优先级任务就绪后可以抢占低优先级任务；
- 实时响应更及时；
- 是多数 FreeRTOS 项目的常见配置；
- 共享资源保护要求更严格。

#### 协作式调度

```c
#define configUSE_PREEMPTION  0
```

特点：

- 当前任务通常需要主动阻塞或让出 CPU；
- 高优先级任务就绪后不一定立即运行；
- 某个任务长时间不让出 CPU，会影响整个系统；
- 任务间切换行为相对简单，但实时响应通常较差。
  两者都需要 Tick，也都可以使用任务优先级；主要区别是高优先级任务就绪后能否自动抢占。

### 4. 优先级反转

### 最终归纳

1. FreeRTOS 任务优先级数值越大，优先级越高。
2. 调度器只比较处于就绪态的任务。
3. 高优先级任务就绪后，可以抢占正在运行的低优先级任务。
4. 同优先级任务可以时间片轮转，也可以通过阻塞主动让出 CPU。
5. 高优先级任务如果永不阻塞，会造成低优先级任务饥饿。

## 队列

FreeRTOS 队列（Queue）用于在任务之间，或者中断与任务之间安全地传递数据。
常见包括队列、二进制信号量、计数信号量和互斥锁。

```
发送端                              		接收端
Task / ISR  →  [数据 1][数据 2][数据 3]  →  Task
                 队尾          队头
```

队列主要有三个作用：

- 传递数据；
- 缓冲生产者和消费者之间的速度差异；
- 实现任务同步——队列为空时接收任务可以阻塞，队列满时发送任务可以阻塞。

队列的基本特征：

- FIFO先进先出
- 按值复制数据：FreeRTOS 队列默认复制数据，而不是保存发送变量本身，内核会把 `data` 的内容复制进队列。发送完成后，即使修改原变量，队列中的副本也不会改变：

```c
xQueueSend(sensorQueue, &data, 0);
data.temperature = 100;  // 不影响已经进入队列的数据
```

- 线程安全：FreeRTOS 队列内部已经处理了任务并发和中断访问问题。多个任务可以向同一个队列发送，也可以从同一个队列接收。一般不需要再额外使用 Mutex 保护队列本身。

队列的具体使用不在这里展开了，需要的时候可以 vibe coding，下面来讨论一些使用队列时的特殊情况。

### 使用静态内存池传递大型数据

因为队列采用复制机制，如果每个元素很大：

```c
typedef struct
{
    uint8_t data[1024];
} LargePacket_t;
```

每次发送和接收都复制 1024 字节，会带来：

- 较大的队列存储空间；
- 更长的数据复制时间；
- 更长的临界操作时间；
- 更高的任务和中断负担。
  这种情况下可以通过队列传递指针：

```c
LargePacket_t *packet;

xQueueSend(packetQueue, &packet, 0);
```

注意创建队列时元素大小是指针大小：

```c
packetQueue =
    xQueueCreate(8, sizeof(LargePacket_t *));
```

但传指针必须明确以下问题：

- 缓冲区是谁创建的？
- 发送后生产者还能否修改？
- 接收后由谁释放或归还？
- 指针指向的数据是否仍然有效？
- 队列满、发送失败时如何处理缓冲区？
- 多个任务是否会同时访问该缓冲区？

常见方法是使用**静态内存池**。这样比随意使用动态 `malloc/free` 更容易控制内存生命周期。

```
空闲缓冲区队列
       ↓
生产者取得缓冲区并填充
       ↓
数据队列
       ↓
消费者处理缓冲区
       ↓
归还空闲缓冲区队列
```

静态内存池的核心思想是：

> 程序启动时预先准备固定数量的消息对象，运行期间只进行“借出”和“归还”，不再频繁调用 `pvPortMalloc()` 和`vPortFree()`。

针对前面的 `LEDMessage *` 队列，可以使用“两队列内存池”：

```
FreeQueue（空闲指针）
        ↓ 生产者取出
填充 LEDMessage
        ↓
LEDQueue（待处理指针）
        ↓ 消费者处理
处理完成
        ↓
归还 FreeQueue
```

#### 1. 数据结构

假设最多允许同时存在 8 条 LED 消息：

```c
#define LED_MESSAGE_COUNT  8

typedef enum
{
    LEDState_Off = 0,
    LEDState_On  = 1
} LEDState;

typedef enum
{
    LEDColor_Red,
    LEDColor_Green,
    LEDColor_Blue
} LEDColor;

typedef struct
{
    LEDColor color;
    LEDState state;
} LEDMessage;
```

静态创建 8 个消息对象：

```c
static LEDMessage ledMessagePool[LED_MESSAGE_COUNT];
```

这里的内存在程序运行期间始终有效，不存在局部变量生命周期问题。

---

#### 2. 创建两个指针队列

需要两个队列：

```c
osMessageQueueId_t LEDQueueHandle;
osMessageQueueId_t LEDFreeQueueHandle;
```

创建队列：

```c
void LEDMessagePool_Init(void)
{
    LEDQueueHandle = osMessageQueueNew(
        LED_MESSAGE_COUNT,
        sizeof(LEDMessage *),
        NULL
    );

    LEDFreeQueueHandle = osMessageQueueNew(
        LED_MESSAGE_COUNT,
        sizeof(LEDMessage *),
        NULL
    );

    if ((LEDQueueHandle == NULL) ||
        (LEDFreeQueueHandle == NULL))
    {
        Error_Handler();
    }

    for (uint32_t i = 0; i < LED_MESSAGE_COUNT; i++)
    {
        LEDMessage *message = &ledMessagePool[i];

        if (osMessageQueuePut(
                LEDFreeQueueHandle,
                &message,
                0,
                0) != osOK)
        {
            Error_Handler();
        }
    }
}
```

初始化完成后：

```text
ledMessagePool：
┌────────────┐
│ message[0] │ ← FreeQueue保存其地址
├────────────┤
│ message[1] │ ← FreeQueue保存其地址
├────────────┤
│ ...        │
├────────────┤
│ message[7] │ ← FreeQueue保存其地址
└────────────┘
```

此时：

```
FreeQueue中有8个空闲指针
LEDQueue中没有待处理消息
```

该初始化函数应在任务开始使用队列之前调用。

---

#### 3. 生产者从内存池借用对象

按键任务首先从空闲队列获得一个 `LEDMessage *`：

```c
void StartKeyTask(void *argument)
{
    LEDState state = LEDState_Off;

    for (;;)
    {
        if (isKey1Clicked())
        {
            state = (state == LEDState_Off)
                  ? LEDState_On
                  : LEDState_Off;

            LEDMessage *message = NULL;

            osStatus_t status = osMessageQueueGet(
                LEDFreeQueueHandle,
                &message,
                NULL,
                0
            );

            if (status == osOK)
            {
                message->color = LEDColor_Red;
                message->state = state;

                status = osMessageQueuePut(
                    LEDQueueHandle,
                    &message,
                    0,
                    0
                );

                if (status != osOK)
                {
                    /*
                     * 发送失败，消息没有交给消费者，
                     * 必须归还内存池。
                     */
                    osMessageQueuePut(
                        LEDFreeQueueHandle,
                        &message,
                        0,
                        0
                    );
                }
            }
            else
            {
                /*
                 * 没有空闲消息对象。
                 * 可以计数、报警或丢弃本次消息。
                 */
            }
        }

        osDelay(10);
    }
}
```

所有权变化为：

```text
FreeQueue
    ↓ osMessageQueueGet()
KeyTask拥有message
    ↓ osMessageQueuePut()
LEDQueue拥有message
```

发送成功后，`KeyTask` 不能继续修改这个 `message`，因为它已经交给接收任务。

---

#### 4. 消费者处理后归还对象

LED 任务从工作队列取得指针：

```c
void StartLEDTask(void *argument)
{
    LEDMessage *message = NULL;

    for (;;)
    {
        if (osMessageQueueGet(
                LEDQueueHandle,
                &message,
                NULL,
                osWaitForever) == osOK)
        {
            SetLED(
                message->color,
                message->state
            );

            /*
             * 消息处理完毕，归还空闲队列。
             */
            if (osMessageQueuePut(
                    LEDFreeQueueHandle,
                    &message,
                    0,
                    osWaitForever) != osOK)
            {
                Error_Handler();
            }

            message = NULL;
        }
    }
}
```

所有权变化为：

```text
LEDQueue
    ↓ osMessageQueueGet()
LEDTask拥有message
    ↓ 处理完成
FreeQueue重新保存message
```

消息对象可以不断循环使用：

```
空闲 → 被生产者使用 → 等待处理 → 被消费者使用 → 空闲
```

整个运行期间不需要：

```c
pvPortMalloc();
vPortFree();
```

#### 必须明确的所有权规则

静态内存池能否安全运行，关键不只是“内存是静态的”，而是所有权是否明确。

建议规定：

| 阶段               | 消息所有者      | 谁可以修改       |
| ------------------ | --------------- | ---------------- |
| 指针位于 FreeQueue | 内存池          | 无任务           |
| 生产者取出后       | 生产者          | 生产者           |
| 成功放入 LEDQueue  | LEDQueue/消费者 | 生产者不可再修改 |
| 消费者取出后       | 消费者          | 消费者           |
| 归还 FreeQueue 后  | 内存池          | 消费者不可再访问 |

静态内存池更适合：

- 消息结构体很大；
- DMA 缓冲区不能复制；
- 图像、音频或串口数据块；
- 希望避免运行时动态分配；
- 需要严格控制最大并发消息数量；
- 数据要在多个处理阶段之间传递。

选择原则可以概括为：

```
小型结构体        → 队列按值传递
大型数据块        → 静态内存池 + 指针队列
只关心最新状态    → 长度为1的按值队列
连续字节流        → Stream Buffer或环形缓冲区
```

静态内存池真正解决的并不只是“避免 `malloc`”，而是通过固定数量的数据块和明确的所有权转移，使内存占用、最坏情况和消息生命周期都可预测。

## 信号量

### 二进制信号量和计数信号量

二进制信号量传递是或否，不传递数据。
计数信号量可以传递数量。
盘点事件：事件发生几次就执行几次
资源管理：对有限的资源量进行管理，资源耗尽时，新的资源申请任务就进入阻塞态直到有资源恢复。

### 互斥锁

为了减少优先级反转，引入了互斥锁和优先级继承。

## 任务通知

任务通知可以模拟二进制信号量和计数信号量，但是通常是一对一通信：

- 发送方知道目标任务句柄；
- 通知直接发送给特定任务；
- 不适合存储多条结构体消息；
- 不能让多个任务共同接收同一条通知。

## 中断与 FromISR

普通任务可以阻塞，但是中断不能阻塞，因为 ISR 不是任务，没有可以被调度器保存并等待恢复的普通任务上下文。
中断必须使用对应的 `FromISR` 接口：

```c
xQueueSendFromISR();
xQueueReceiveFromISR();
xSemaphoreGiveFromISR();
vTaskNotifyGiveFromISR();
xTaskNotifyFromISR();
xStreamBufferSendFromISR();
```

`FromISR` 接口具有以下特点：

- 不会阻塞；
- 操作失败时立即返回；
- 可以告诉调用者是否唤醒了更高优先级任务；
- 经过专门设计，可在允许的中断优先级中调用。

### ISR 中不能使用 Mutex

Mutex 具有“所有者”概念：

```
哪个任务获取Mutex
哪个任务负责释放Mutex
```

ISR 不是普通任务，不能阻塞等待 Mutex，也不应成为 Mutex 所有者。因此不要在 ISR 中使用：

```c
xSemaphoreTakeFromISR(mutex, ...);  // 不应这样设计
xSemaphoreGiveFromISR(mutex, ...);
```

ISR 和任务共享少量数据时，常用方法包括：

- 临界区；
- 原子变量；
- 队列；
- Stream Buffer；
- 任务通知；
- 双缓冲；
- DMA 缓冲区加所有权转移。

## 事件组

### 本质：一组可独立设置的二进制状态位

一个事件标志组通常包含若干 bit，每个 bit 表示一个事件或条件：

```
bit 0：ADC数据就绪
bit 1：IMU初始化完成
bit 2：无线连接成功
bit 3：系统发生错误
```

例如：

```c
#define FLAG_ADC_READY   (1U << 0)
#define FLAG_IMU_READY   (1U << 1)
#define FLAG_RADIO_READY (1U << 2)
#define FLAG_ERROR       (1U << 3)
```

当前标志值可能为：

```
0000 0000    没有事件
0000 0011    ADC和IMU都已就绪
0000 0111    ADC、IMU和无线模块都已就绪
```

CMSIS-RTOS 2 使用以下对象：

```c
osEventFlagsId_t eventHandle;
```

创建：

```c
eventHandle = osEventFlagsNew(NULL);
```

设置标志：

```c
osEventFlagsSet(eventHandle, FLAG_ADC_READY);
```

等待标志：

```c
osEventFlagsWait(...);
```

事件标志组不是数据容器，它主要表达：

> 某个事件是否发生，或者某个条件是否成立。

### 自动清除与保留标志

任务等待成功后，相关事件位可以有两种处理方式。

#### 自动清除

默认情况下，等待成功后，满足条件的目标标志通常会被清除。

可以理解为：

```
事件发生
  ↓
任务收到事件
  ↓
事件被消费
  ↓
标志清零
```

适合表达一次性事件：

- 一帧数据到达；
- 一次 ADC 转换完成；
- 一次按键事件；
- 一次通信完成。

#### 不自动清除

使用：

```
osFlagsNoClear
```

等待成功后保留标志。

适合表达持续状态：

- 系统已经初始化；
- 网络目前已连接；
- 存储器目前可用；
- 某功能当前已启用。

此时需要在状态失效时显式清除：

```c
osEventFlagsClear();
```

因此需要先确定每个 bit 表示的是：

```
一次事件
```

还是：

```
持续状态
```

这是设计事件标志组时最重要的区分之一。

### 事件标志不能准确累计重复事件

假设 ADC 连续完成三次，三次都设置同一个 bit：

```
第一次：FLAG_ADC_READY = 1
第二次：FLAG_ADC_READY = 1
第三次：FLAG_ADC_READY = 1
```

最终仍然只是：

```
FLAG_ADC_READY = 1
```

事件标志只表示“发生过或条件成立”，不能说明发生了几次。

因此：

| 需求               | 推荐机制         |
| ------------------ | ---------------- |
| 是否发生过         | 事件标志         |
| 发生了多少次       | 计数信号量       |
| 每次事件都附带数据 | 消息队列         |
| 唤醒一个固定任务   | 线程标志或信号量 |
| 多个条件组合等待   | 事件标志组       |

如果三次 ADC 完成都必须分别处理，就不应该只使用一个事件位。

### 多个任务可以等待同一个事件标志组

事件标志组是独立的内核对象，因此多个任务可以同时等待它：

```
                    ┌→ TaskA等待FLAG_READY
Event Flags对象 ────┼→ TaskB等待FLAG_ERROR
                    └→ TaskC等待READY | ERROR
```

设置一次标志后，所有等待条件得到满足的任务都可能进入就绪态。

但如果任务等待成功后自动清除标志，就要考虑：

- 哪个任务先消费该标志；
- 其他任务是否还能观察到它；
- 是否确实需要广播；
- 是否应该设置 `osFlagsNoClear`。

如果一个事件必须可靠地通知多个任务，通常需要更明确的广播设计，而不能仅凭默认自动清除行为推断所有任务都会收到。

### 事件标志组的典型使用场景

#### 系统初始化同步

```
IMU初始化完成     → bit 0
无线模块初始化完成 → bit 1
Flash初始化完成   → bit 2

主业务任务等待 bit 0、1、2 全部置位
```

#### 通信状态机

```
接收完成 → bit 0
发送完成 → bit 1
超时     → bit 2
错误     → bit 3
```

#### 多传感器数据同步

```
IMU数据就绪   → bit 0
超声数据就绪  → bit 1

融合任务等待两个bit全部置位
```

但如果各传感器数据产生速度不同，还需要队列或双缓冲保存实际数据。事件标志本身不能保存采样值，也不能防止数据被覆盖。

## 后续学习目录

1. 任务通知
2. 中断与 FromISR 接口
3. vTaskDelayUntil 与周期任务
4. 事件组
5. 软件定时器
6. Stream Buffer 和 Message Buffer
7. 任务栈与 FreeRTOS 堆
8. 临界区与原子操作
9. 死锁、饥饿、优先级反转
10. 调试、断言和运行时间统计
11. 低功耗与 Tickless Idle

## Q&A

### 1. FreeRTOS 为什么不建议使用动态内存？

FreeRTOS 并不是禁止使用动态内存，而是在实时嵌入式系统中通常不建议在运行过程中频繁进行动态内存申请和释放。主要原因是动态内存可能产生内存碎片，申请时间具有一定的不确定性，同时还存在内存申请失败和内存泄漏的问题，这些都会影响系统长期运行的稳定性和实时性。因此对于任务、队列、信号量等生命周期固定的对象，通常更推荐静态分配；如果需要动态分配，可以在系统初始化阶段完成，并根据需求选择合适的 FreeRTOS heap 实现，例如常见的 `heap_4.c`。
