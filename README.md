# Embedded System Notes

个人嵌入式系统学习与源码分析仓库。

当前方向：

- FreeRTOS
- BLE
- SubG
- RF System
- Linux
- Embedded Debug
- Embedded Drivers
- Board Architecture / PinMux
- Wireless System Architecture

重点：
不是 API 使用，
而是研究"系统为什么这样运行"。

## 知识索引

### RTOS

- [001 — FreeRTOS 信号量内核深度分析](RTOS/001_FreeRTOS_Semaphore_Kernel_Deep_Dive.md)
- [001 v2 — FreeRTOS 信号量内核深度分析](RTOS/001_FreeRTOS_Semaphore_Kernel_Deep_Dive_v2.md)
- [002 — FreeRTOS Task / TCB / Scheduler 内核深度分析](RTOS/002_Task_Lifecycle_And_TCB.md)

### Drivers

- [001 — EFR32FG23 SSD1306 OLED / SWD PinMux / HR Display 工程实践](Drivers/OLED/001_EFR32FG23_SSD1306_SWD_PinMux_HR_Display.md)

### 本次新增主题

- OLED SSD1306 驱动移植
- EFR32FG23 SWD 引脚复用
- I2C0/I2C1 外设资源规划
- ZSPD4000 心率数据显示
