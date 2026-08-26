# STM32F103C8T6 FreeRTOS 移植工程

这是一个基于 Keil MDK 的 STM32F103C8T6（Cortex-M3）FreeRTOS 示例工程。工程使用 STMicroelectronics STM32F10x Standard Peripheral Library，移植 FreeRTOS Kernel V11.3.1，并通过一个任务驱动 GPIOC Pin 13 LED 周期闪烁，用于验证调度器、系统节拍和任务上下文切换是否正常工作。

## 1. 软件架构

工程采用“应用层 + RTOS 内核 + Cortex-M3 移植层 + 芯片支持层”的分层结构：

```text
应用层
  User/main.c                 创建任务、初始化 GPIO、启动调度器
  User/stm32f10x_it.c         用户异常和外设中断模板
          |
          v
FreeRTOS 内核层
  freeRTOS/src/               任务、队列、定时器、事件组、流缓冲等内核模块
  freeRTOS/inc/               FreeRTOS 公共头文件和 API
          |
          +------------------+
          v                  v
移植与内存层                  芯片支持层
  freeRTOS/port/              Cortex-M3 port.c/portmacro.h、heap_4.c
  Start/                      启动文件、CMSIS Cortex-M3、系统时钟
  Library/                    STM32F10x 外设驱动库
```

### 启动和运行链路

1. 复位后由 `Start/startup_stm32f10x_md.s` 建立向量表、初始化栈和数据段，然后进入系统启动代码。
2. `Start/system_stm32f10x.c` 提供 CMSIS 系统初始化和时钟相关支持，随后调用 `main()`。
3. `User/main.c` 开启 GPIOC 时钟，将 PC13 配置为 50 MHz 推挽输出，并创建 `myTask`。
4. `vTaskStartScheduler()` 启动 FreeRTOS，之后 CPU 的执行主体由任务调度器接管。
5. FreeRTOS 使用 SysTick 产生系统节拍；通过 PendSV 完成任务上下文切换，通过 SVC 启动第一个任务。这些异常由 `freeRTOS/port/port.c` 提供，应用中断模板中的同名处理函数保持注释，避免重复定义。
6. `myTask` 每 500 ms 翻转一次 PC13，验证任务延时和周期调度。

## 2. 关键配置

配置文件为 `freeRTOS/FreeRTOSConfig.h`，当前主要参数如下：

| 配置项 | 当前值 | 作用 |
| --- | ---: | --- |
| `configTICK_RATE_HZ` | `100` | RTOS 节拍频率为 100 Hz，1 tick 理论上为 10 ms |
| `configUSE_PREEMPTION` | `1` | 启用抢占式调度 |
| `configUSE_TIME_SLICING` | `0` | 同优先级就绪任务不会仅因 Tick 到来而轮转 |
| `configMAX_PRIORITIES` | `5` | 可用优先级为 0 至 4，数值越大优先级越高 |
| `configMINIMAL_STACK_SIZE` | `128` | 空闲任务默认栈深度，单位为 word |
| `configTOTAL_HEAP_SIZE` | `4096` | FreeRTOS 动态堆大小为 4096 字节 |
| `configSUPPORT_STATIC_ALLOCATION` | `1` | 支持静态创建 RTOS 对象 |
| `configSUPPORT_DYNAMIC_ALLOCATION` | `1` | 支持动态创建 RTOS 对象 |
| `configUSE_TIMERS` | `1` | 启用软件定时器服务任务 |
| `configUSE_EVENT_GROUPS` | `1` | 启用事件组 |
| `configUSE_STREAM_BUFFERS` | `1` | 启用流缓冲和消息缓冲 |
| `configASSERT` | 已启用 | 断言失败后关闭中断并停在死循环，便于调试 |

本工程实际加入构建的内存管理实现是 `freeRTOS/port/heap_4.c`。它支持动态分配和释放，并会合并相邻空闲块；`heap_1.c` 至 `heap_5.c` 同时存在于目录中，但未全部加入工程，切换实现时应在 Keil 工程中替换源文件，避免多个实现同时参与链接。

## 3. 当前应用示例

`main.c` 当前只有一个应用任务：

```c
xTaskCreate(myTask, "my_Task", 128, NULL, 2, &myTaskHandler);
```

- 任务名：`my_Task`
- 栈深度：128 word（Cortex-M3 上通常为 512 字节，具体以编译器类型宽度为准）
- 优先级：2
- 任务行为：PC13 输出低电平 500 ms，再输出高电平 500 ms，循环执行
- 调度器启动后，`main()` 中的无限循环通常不会再被执行

多数 STM32 开发板上的 PC13 LED 为低电平点亮，因此代码中的 `GPIO_ResetBits()` 通常表现为点亮，`GPIO_SetBits()` 表现为熄灭；若板卡 LED 逻辑相反，现象会相反，但不影响 RTOS 验证。

## 4. 目录说明

| 目录或文件 | 说明 |
| --- | --- |
| `User/` | 应用入口、任务和用户中断模板 |
| `freeRTOS/inc/` | FreeRTOS Kernel 公共头文件 |
| `freeRTOS/src/` | FreeRTOS Kernel C 源码 |
| `freeRTOS/port/` | Cortex-M3 移植代码和堆实现 |
| `freeRTOS/FreeRTOSConfig.h` | 本工程 RTOS 配置 |
| `Library/` | STM32F10x Standard Peripheral Library |
| `Start/` | 启动文件、CMSIS 内核文件和系统文件 |
| `Project.uvprojx` | Keil uVision 工程文件 |
| `Objects/`、`Listings/` | 编译生成物和列表文件，不属于源码架构 |

## 5. 构建、下载与验证

1. 使用 Keil MDK 打开 `Project.uvprojx`。
2. 确认已安装 STM32F1 Device Family Pack，并选择 `STM32F103C8` 目标器件。
3. 执行 **Build**，确认 FreeRTOS 内核、移植层、启动文件和应用代码均参与编译。
4. 连接 ST-Link，执行下载并复位芯片。
5. 观察 PC13 LED 是否以约 1 秒为一个完整周期闪烁（亮 500 ms、灭 500 ms）。
6. 调试时可在 `myTask()`、`vTaskStartScheduler()` 或 `freeRTOS/port/port.c` 设置断点，观察任务启动和上下文切换。

## 6. 移植检查项

### 时钟频率必须一致

当前工程存在一个需要在实际硬件上确认的配置差异：

- `Project.uvprojx` 的目标配置声明 `CLOCK(12000000)`。
- `freeRTOS/FreeRTOSConfig.h` 设置 `configCPU_CLOCK_HZ` 为 `20000000`。

`configCPU_CLOCK_HZ` 应与实际驱动 SysTick 的内核时钟一致，否则 `configTICK_RATE_HZ` 对应的真实时间会产生偏差。烧录前请根据 `system_stm32f10x.c` 的实际时钟树和芯片配置统一这两个设置；README 不自动替用户修改，因为实际板卡外部晶振和系统时钟可能不同。

### 中断优先级规则

凡是在中断服务函数中调用 FreeRTOS `FromISR` API 的中断，必须遵守 `configMAX_SYSCALL_INTERRUPT_PRIORITY` 的限制。Cortex-M3 的优先级数值越小表示优先级越高，具体分组和优先级位数应与 NVIC 配置保持一致。不要在高于允许等级的中断中调用普通 FreeRTOS API。

### 栈和堆检查

当前未启用 `configCHECK_FOR_STACK_OVERFLOW`，也未启用 `configUSE_MALLOC_FAILED_HOOK`。扩展任务或增加队列后，建议开启栈溢出检测、实现内存申请失败钩子，并根据 `uxTaskGetStackHighWaterMark()` 调整任务栈大小。4 KB 堆在当前单任务示例中足够，但不应视为复杂应用的固定值。

## 7. 扩展建议

- 将 GPIO 初始化、业务逻辑和任务函数拆分到独立模块，避免 `main.c` 持续膨胀。
- 任务间共享数据优先使用队列、任务通知、信号量或事件组，不要依赖无保护的全局变量。
- 需要固定内存占用时使用静态任务和静态 RTOS 对象；需要灵活创建对象时继续使用动态分配，但应监控堆剩余空间。
- 增加串口、定时器或外部中断时，在 ISR 中只做必要的硬件处理，并使用 `FromISR` API 将工作交给任务完成。
- 不建议把耗时循环、阻塞式延时或复杂业务逻辑放入中断服务函数。

## 8. 依赖与许可

工程依赖 Keil MDK/ARMCC、STM32F1 Device Family Pack、STM32F10x Standard Peripheral Library 和 FreeRTOS Kernel。FreeRTOS Kernel 源文件包含其原始 MIT 许可声明；使用和再发布时请同时遵守各依赖组件自身的许可条款。