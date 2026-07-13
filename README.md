# 蓝牙智能手表项目总体预览

## 一、项目概览

基于 STM32F103C8T6 + FreeRTOS 的智能手表原型，集成 OLED 显示、MPU6050 姿态检测、HC-05 蓝牙通信和旋转编码器交互。

| 项目 | 内容 |
|------|------|
| 主控 | STM32F103C8T6 (Cortex-M3, 72MHz) |
| 系统 | FreeRTOS + CMSIS-RTOS v1 |
| 工具链 | CubeMX + ARM GCC + CMake + Ninja |
| 工程文件 | `bluetooth.ioc` |

---

## 二、硬件连接

| 外设 | 接口 | 引脚 |
|------|------|------|
| SSD1306 OLED (0.96") | I2C1（地址 0x3C） | PB6(SCL) / PB7(SDA) |
| MPU6050 六轴传感器 | I2C1（地址 0x68/0x69） | PB6(SCL) / PB7(SDA) |
| HC-05 蓝牙 | USART2（9600 baud） | PA2(TX) / PA3(RX) |
| HC-05 STATE | GPIO 输入 | PB0 |
| 旋转编码器 | TIM3 编码器模式 | PA6 / PA7 |
| 编码器按键 | GPIO 输入（按键消抖） | PA15 |
| 调试接口 | SWD | SWDIO / SWCLK |

I2C1 总线上 OLED 和 MPU6050 地址不同，共享 SCL/SDA 两根线。

---

## 三、FreeRTOS 任务

| 任务 | 优先级 | 栈 | 周期 | 功能 |
|------|--------|----|------|------|
| `vTask_TimeKeep` | High | 256B | 1s | 更新软件 RTC，发消息队列 |
| `vTask_Sensor` | AboveNormal | 512B | 200ms | 读 MPU6050，运行计步算法 |
| `StartDisplayTask` | Normal | 1024B | 500ms | 接收队列数据，刷新 OLED |
| `vTask_Menu` | Normal | 512B | 20ms | 读编码器 + 按键，切换页面 |
| `vTask_Bluetooth` | BelowNormal | 512B | 500ms | 轮询串口，处理手机命令 |
| `StartDefaultTask` | Normal | 128B | - | 空闲 |

### 任务间通信

- **消息队列**：TimeKeep → Display（时间）、Sensor → Display（传感器数据）、Menu → Display（页面切换）
- **互斥量**：`oled_mutex` 保护 I2C1 总线，Sensor 和 Display 共享

---

## 四、用户界面（5 个页面）

| 页面 | 显示内容 |
|------|---------|
| 0 - 表盘 | 大号时间 (16x24 字体)、日期、温湿度 |
| 1 - 心率 | 心率数值 + BPM、最小值/最大值、心率波形 |
| 2 - 计步 | 步数图标 + 步数、距离 (km)、卡路里 (kcal) |
| 3 - 天气 | 太阳图标、温度、湿度、气压 |
| 4 - 设备信息 | 三轴加速度原始值、读取次数、I2C 状态 |

---

## 五、蓝牙命令

通过手机 App（Serial Bluetooth Terminal）发送单字符命令：

| 命令 | 回复 |
|------|------|
| `R` | `TIME:10:30:45` |
| `S` | `STEP:1243` |
| `H` | `HR:78` |

HC-05 配对密码：**1234**

---

## 六、核心模块

| 模块 | 文件 | 说明 |
|------|------|------|
| OLED 驱动 | `oled.c` / `oled.h` | SSD1306 驱动、帧缓冲、6x8/8x16/16x24 字体、基本绘图 |
| MPU6050 驱动 | `mpu6050.c` / `mpu6050.h` | I2C 读写、初始化、数据读取 |
| UI 框架 | `smartwatch_ui.c` / `.h` | 5 个页面绘制、状态栏、页面指示、步进计数器、软件 RTC |
| FreeRTOS | `freertos.c` | 所有任务、队列、互斥量、信号量的创建 |
| 蓝牙 | `freertos.c` vTask_Bluetooth | UART 收发、命令解析 |

---

## 七、已解决的关键问题

1. **MPU6050 连接失败** — I2C1 GPIO 缺少内部上拉（`GPIO_PULLUP`），WHO_AM_I 值 0x74 被错误拒认
2. **步数自行增加** — 阈值从 10M 调高到 40M（约 0.4g），增加全零数据过滤
3. **蓝牙无响应** — 波特率从 38400 改为 9600、解除 DMA RX 链接（与轮询冲突）、取消 STATE 引脚依赖

---

## 八、主程序启动流程

```
系统上电 → HAL 初始化 → 时钟配置 (72MHz)
  → GPIO / I2C1 / I2C2 / USART2 / TIM2 / TIM3 初始化
  → OLED 显示 "SmartWatch / Starting..."
  → 软件 RTC 初始化 (2026-07-09 10:30:00)
  → 计步器清零
  → MPU6050 自检（OLED 显示 OK / FAIL + WHO_AM_I 值）
  → TIM2 启动 (1Hz RTC), TIM3 启动 (编码器)
  → FreeRTOS 初始化（创建 6 个任务 + 4 个队列 + 2 个互斥量）
  → osKernelStart() 启动调度器
```