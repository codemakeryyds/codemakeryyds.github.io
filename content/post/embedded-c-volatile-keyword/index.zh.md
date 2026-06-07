---
title: "volatile 不是万能锁：嵌入式 C 里它到底该怎么用"
description: "整理 volatile 在嵌入式 C 中的真实用途：硬件寄存器、ISR 标志位、const volatile，以及它不能替代原子操作、锁和 RTOS 同步机制的原因。"
date: 2026-06-07T00:00:00+08:00
slug: "embedded-c-volatile-keyword"
categories:
    - "随笔"
tags:
    - "嵌入式"
    - "C语言"
    - "volatile"
    - "STM32"
draft: false
---

`volatile` 这个关键字很容易被讲玄。

有些资料会把它讲成“保证线程安全”“解决缓存一致性”“禁止所有重排序”。看着很强，但真写嵌入式 C 的时候，这么理解反而容易出事。

我现在更愿意把它记成一句话：

> `volatile` 是写给编译器看的：这个变量可能会被当前代码之外的东西改掉，每次访问都老老实实去内存或寄存器读写，别自作聪明优化掉。

它很重要。

但它不是锁。

### 不加 volatile，编译器可能真的会“自作聪明”

先看一个很典型的硬件状态轮询：

```c
#define UART_STATUS_ADDR  0x4000A000u

uint8_t *status = (uint8_t *)UART_STATUS_ADDR;

while ((*status & 0x01u) == 0) {
    // wait until hardware sets bit0
}
```

从人的角度看，这段代码每次循环都应该重新读取 `0x4000A000`。

但编译器不一定这么想。

如果它只从 C 代码本身推理，`status` 指向的值好像没有被当前程序改过。那它就可能把读取结果缓存起来，甚至把循环优化成一个永远等不到变化的循环。

硬件当然可能会改这个地址。

问题是编译器不知道。

这时候就要写成：

```c
#define UART_STATUS_ADDR  0x4000A000u

volatile uint8_t *status = (volatile uint8_t *)UART_STATUS_ADDR;

while ((*status & 0x01u) == 0) {
    // every loop reads the register again
}
```

加上 `volatile` 后，编译器必须保留对这个地址的实际读取。对嵌入式来说，这是访问内存映射寄存器的基本常识。

### 最典型的用法：硬件寄存器

MCU 外设寄存器本质上就是一段固定地址上的特殊内存。

比如 UART 可能有控制寄存器、状态寄存器、数据寄存器：

```c
typedef struct {
    volatile uint32_t CR;   // control register
    volatile uint32_t SR;   // status register
    volatile uint32_t DR;   // data register
} UART_Registers;

#define UART1 ((UART_Registers *)0x4000A000u)
```

启用发送位：

```c
UART1->CR |= (1u << 3);
```

等待发送完成：

```c
while ((UART1->SR & (1u << 7)) == 0) {
    // wait TXE
}
```

这里的 `volatile` 很关键。

`SR` 可能被硬件改变，`DR` 写进去可能触发发送动作，`CR` 的某些 bit 可能影响外设行为。这些读写都有副作用，不能让编译器随便删、合并、缓存。

### const volatile：程序不能写，但硬件会改

有些寄存器对程序来说是只读的，比如状态寄存器。

这时可以用 `const volatile`：

```c
const volatile uint32_t *status_reg =
    (const volatile uint32_t *)0x40010000u;
```

这两个修饰词不是矛盾的：

- `const`：当前 C 程序不应该写它；
- `volatile`：它的值可能被硬件改掉，每次读都要重新读。

我觉得这个组合特别适合记住 `volatile` 的本质。

它不是说“变量可以随便变”，而是说“别假设它不会变”。

### ISR 和主循环共享 flag，也需要 volatile

另一个常见场景是中断和主循环之间传递标志位。

比如传感器数据准备好了，中断里置一个 flag：

```c
static volatile uint8_t sensor_ready = 0;

void EXTI_IRQHandler(void)
{
    sensor_ready = 1;
}

int main(void)
{
    while (1) {
        if (sensor_ready) {
            sensor_ready = 0;
            Sensor_ReadAndProcess();
        }
    }
}
```

`sensor_ready` 必须是 `volatile`。

否则编译器可能认为主循环里没人会改它，把它缓存到寄存器里。中断明明已经把 flag 置 1，主循环还是看不到。

这类 bug 很难查。

因为你看源码觉得没问题，调试时单步也可能没问题，一开优化就炸。

### 但 volatile 不保证原子性

这个点很容易踩坑。

下面这段代码就算加了 `volatile`，也不是线程安全的：

```c
volatile uint32_t counter = 0;

void TaskA(void)
{
    counter++;
}

void TaskB(void)
{
    counter++;
}
```

`counter++` 不是一条神奇的原子指令。

它通常至少包含三步：

```text
read counter
counter + 1
write counter
```

两个任务同时执行，就可能都读到旧值，然后覆盖对方的结果。

`volatile` 只能要求编译器真的去读写 `counter`，不能保证这三步不被打断。

所以这类场景要用临界区、原子操作、互斥锁，或者 RTOS 提供的同步机制。

比如在 FreeRTOS 里，简单临界区可以这样写：

```c
taskENTER_CRITICAL();
counter++;
taskEXIT_CRITICAL();
```

裸机里也可能需要短暂关中断：

```c
__disable_irq();
counter++;
__enable_irq();
```

当然，关中断要克制，临界区越短越好。

### 多字节变量也要小心

还有一种容易忽略的情况：8 位 MCU 上读写 16 位或 32 位变量。

比如中断里更新一个 32 位时间戳，主循环里读取：

```c
volatile uint32_t tick_count;
```

`volatile` 能保证每次读都会发生，但不能保证一次 32 位读取不会被中断打断。

如果 CPU 需要分多次读这个变量，主循环可能读到一半旧值、一半新值。

这种时候还是要保护临界区：

```c
uint32_t GetTickSafe(void)
{
    uint32_t value;

    __disable_irq();
    value = tick_count;
    __enable_irq();

    return value;
}
```

所以不要看到 `volatile` 就放心。

它解决的是“编译器别优化掉访问”，不是“访问过程绝对安全”。

### RTOS 里不要滥用 volatile 传消息

很多人会在 RTOS 任务之间用全局 flag 通信：

```c
volatile uint8_t uart_rx_done = 0;
```

能不能用？

小项目可以。

但任务一多，flag 很快会变成一堆：

```text
uart_rx_done
sensor_ready
timeout_flag
low_power_request
error_flag
```

然后你会开始追：谁置位的？谁清零的？有没有丢事件？两个任务会不会同时改？

如果已经用了 RTOS，更推荐把事件交给 RTOS 机制：

- 只通知一个任务：task notification；
- 传递结构体事件：queue；
- 多个 bit 状态组合：event group / event flags；
- 保护共享资源：mutex / semaphore。

比如用 queue 表达按键事件：

```c
typedef enum {
    APP_EVT_KEY_SHORT,
    APP_EVT_KEY_LONG,
    APP_EVT_UART_RX,
} AppEventType;

typedef struct {
    AppEventType type;
    uint32_t data;
} AppEvent;
```

ISR 里只投递事件：

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    BaseType_t hpw = pdFALSE;

    if (GPIO_Pin == KEY_Pin) {
        AppEvent evt = {
            .type = APP_EVT_KEY_SHORT,
            .data = 0,
        };

        xQueueSendFromISR(app_event_queue, &evt, &hpw);
        portYIELD_FROM_ISR(hpw);
    }
}
```

业务任务慢慢处理：

```c
void AppTask(void *argument)
{
    AppEvent evt;

    for (;;) {
        if (xQueueReceive(app_event_queue, &evt, portMAX_DELAY) == pdPASS) {
            HandleAppEvent(&evt);
        }
    }
}
```

这比到处轮询 `volatile` flag 清楚很多。

### 我现在的使用原则

以后写嵌入式 C，我大概会按这个规则用 `volatile`：

```text
硬件寄存器：要用 volatile
ISR 和主循环共享的简单 flag：要用 volatile
只读但会被硬件改变的寄存器：const volatile
计数器自增、多任务共享结构体：volatile 不够
任务间复杂通信：优先用 queue / semaphore / event flags
临界区共享数据：用关中断、锁或原子操作保护
```

一句话总结：

> `volatile` 很有用，但它只解决一类问题。别拿它当万能同步工具。

### 写在最后

我觉得 `volatile` 最容易误用的原因，是它刚好出现在很多“共享变量”的场景里。

硬件会改寄存器，中断会改 flag，任务之间也会共享状态。看起来都像“变量被外部改了”，于是很多人下意识全加 `volatile`。

但真正要问的是：这个变量的问题到底是什么？

如果问题是“编译器可能优化掉访问”，`volatile` 对。

如果问题是“读写会被打断”“多个任务会同时改”“事件可能丢失”，那就不是 `volatile` 能单独解决的了。

这也是我现在对它的理解：该用的时候必须用，但不要指望它做超出能力范围的事。
