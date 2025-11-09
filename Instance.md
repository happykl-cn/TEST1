# STM32 定时器中断控制舵机 - 完全小白教程

> 本文档面向完全不了解 STM32 的初学者，详细解释定时器中断、回调函数和程序执行流程

---

## 📋 目录

- [STM32CubeMX 配置详解](#stm32cubemx-配置详解)
- [项目概述](#项目概述)
- [核心概念：什么是中断？](#核心概念什么是中断)
- [核心概念：什么是回调函数？](#核心概念什么是回调函数)
- [硬件配置说明](#硬件配置说明)
- [代码逐行详解](#代码逐行详解)
- [完整程序执行流程](#完整程序执行流程)
- [时序图详解](#时序图详解)
- [常见问题解答](#常见问题解答)

---

## STM32CubeMX 配置详解

> 这是最重要的基础章节！STM32CubeMX 的配置决定了整个项目的基础架构

### 🤔 什么是 STM32CubeMX？

**STM32CubeMX** 是 ST 官方提供的**图形化配置工具**，可以理解为：

**生活中的类比**：
- 就像**装修设计软件**，你不需要自己画图纸、计算材料
- 只需要在界面上点选：这里要装灯、那里要装开关
- 软件自动生成施工图纸和材料清单

**STM32CubeMX 的作用**：
- 你不需要手写复杂的寄存器配置代码
- 只需要在图形界面上选择：这个引脚做什么、时钟怎么配置
- 软件自动生成初始化代码

### 🎯 为什么需要 STM32CubeMX？

**不用 CubeMX（手写代码）**：
```c
// 需要写几十行代码来配置一个引脚
RCC->APB2ENR |= RCC_APB2ENR_IOPAEN;  // 使能 GPIOA 时钟
GPIOA->CRL &= ~(0xF << 0);            // 清除配置
GPIOA->CRL |= (0x3 << 0);              // 配置为输出
GPIOA->BSRR = (1 << 0);                // 设置引脚
// ... 还有很多行
```

**用 CubeMX**：
```
1. 点击 PA0 引脚
2. 选择 "TIM2_CH1"
3. 点击 "Generate Code"
4. 代码自动生成！
```

---

## 📌 引脚配置详解

### 🔌 引脚是什么？

**生活中的类比**：
- 引脚就像**房子的插座**
- 每个引脚可以连接不同的设备
- 但每个引脚的功能是**有限的**，不能随意使用

**STM32F103 的引脚特点**：
- 有多个引脚（如 PA0, PA1, PB0, PC13 等）
- 每个引脚可以配置为**多种功能**
- 但**同一时刻只能使用一种功能**

### 🎨 引脚功能类型

#### 1. GPIO（通用输入输出）

**作用**：最简单的数字输入/输出

**可以做什么**：
- **输出模式**：控制 LED 亮灭、继电器开关
- **输入模式**：读取按键状态、传感器信号

**配置示例**（PC13 控制 LED）：
```
在 CubeMX 中：
1. 找到 PC13 引脚
2. 选择 "GPIO_Output"
3. 生成代码后，可以用：
   HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);   // 点亮
   HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET); // 熄灭
```

#### 2. 复用功能（Alternate Function）

**作用**：引脚连接到内部外设（定时器、串口、SPI 等）

**为什么需要？**
- 定时器、串口等外设需要**特定的引脚**
- 不是所有引脚都能连接所有外设
- 需要查看**引脚映射表**（Pinout）

**配置示例**（PA0 作为 TIM2_CH1，PWM 输出）：
```
在 CubeMX 中：
1. 找到 PA0 引脚
2. 选择 "TIM2_CH1"（定时器2通道1）
3. 生成代码后，引脚自动连接到 TIM2
4. 可以用定时器控制这个引脚输出 PWM
```

#### 3. 模拟功能（Analog）

**作用**：用于 ADC（模数转换）输入

**可以做什么**：
- 读取模拟传感器（温度、光照、电压等）
- 读取电位器位置

**配置示例**（PA0 作为 ADC 输入）：
```
在 CubeMX 中：
1. 找到 PA0 引脚
2. 选择 "ADC1_IN0"（ADC1 通道0）
3. 配置 ADC 参数（分辨率、采样时间等）
4. 生成代码后，可以读取模拟电压值
```

### 📊 引脚配置的实际影响

#### 配置前（CubeMX 中）

```
PA0: [未配置] - 默认可能是浮空输入，不稳定
PC13: [未配置] - 无法使用
```

#### 配置后（CubeMX 中）

```
PA0: [TIM2_CH1] - 已配置为定时器2通道1
PC13: [GPIO_Output] - 已配置为输出模式
```

#### 生成的代码（自动生成）

**在 `tim.c` 中**：
```c
void HAL_TIM_MspPostInit(TIM_HandleTypeDef* timHandle)
{
    // PA0 自动配置为 TIM2_CH1
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;  // 复用推挽输出
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}
```

**在 `gpio.c` 中**：
```c
void MX_GPIO_Init(void)
{
    // PC13 自动配置为输出
    GPIO_InitStruct.Pin = GPIO_PIN_13;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;  // 推挽输出
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
}
```

### ⚠️ 引脚冲突问题

**问题**：同一个引脚不能同时用于多个功能

**示例**：
```
错误配置：
- PA0 同时配置为 TIM2_CH1 和 GPIO_Output
→ 冲突！只能选择一个

正确配置：
- PA0 配置为 TIM2_CH1（用于 PWM）
- PA1 配置为 GPIO_Output（用于 LED）
→ 不冲突，各自工作
```

### 🔍 如何查看引脚功能？

**方法1：在 CubeMX 中查看**
- 点击引脚，查看可用的功能列表
- 已使用的功能会显示为**黄色**

**方法2：查看数据手册**
- STM32F103 数据手册中有**引脚映射表**
- 显示每个引脚支持哪些功能

**方法3：查看代码**
- 生成的代码中会显示引脚配置
- `tim.c` 和 `gpio.c` 中有详细配置

---

## ⏰ 时钟配置详解

### 🕐 时钟是什么？

**生活中的类比**：
- 时钟就像**心脏的跳动**
- 没有时钟，CPU 无法工作
- 时钟频率 = 心跳速度，频率越高，CPU 工作越快

**STM32 的时钟系统**：
```
外部晶振 (8MHz)
    ↓
[PLL 锁相环] → 倍频到 72MHz
    ↓
系统时钟 (SYSCLK = 72MHz)
    ↓
分频到各个总线：
- AHB 总线：72MHz（给 CPU、内存）
- APB1 总线：36MHz（给 TIM2、TIM3 等）
- APB2 总线：72MHz（给 GPIO、TIM1 等）
```

### 🎯 为什么需要配置时钟？

**默认情况**（不配置）：
- STM32 上电后使用**内部 8MHz 时钟**（HSI）
- 性能较低，精度不高

**配置后**：
- 使用**外部 8MHz 晶振**（HSE）
- 通过 **PLL 倍频到 72MHz**
- 性能提升 9 倍！

### 📊 时钟配置详解

#### 1. 时钟源选择（Clock Source）

**HSE（高速外部时钟）**：
- 使用**外部晶振**（通常是 8MHz）
- **优点**：精度高、稳定
- **缺点**：需要外部晶振硬件

**HSI（高速内部时钟）**：
- 使用**内部 RC 振荡器**（8MHz）
- **优点**：不需要外部硬件
- **缺点**：精度较低（±1%）

**配置示例**（在 CubeMX 中）：
```
RCC → High Speed Clock (HSE) → Crystal/Ceramic Resonator
→ 启用外部 8MHz 晶振
```

#### 2. PLL 配置（锁相环）

**PLL 的作用**：
- 将 8MHz 的外部时钟**倍频**到 72MHz
- 就像**齿轮箱**，把慢速转成快速

**配置示例**（在 CubeMX 中）：
```
Clock Configuration 标签页：
- PLL Source: HSE
- PLL Multiplier: x9
- 计算：8MHz × 9 = 72MHz
```

**生成的代码**：
```c
RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;  // PLL 源：外部晶振
RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;          // 倍频：×9
// 结果：8MHz × 9 = 72MHz
```

#### 3. 系统时钟配置（System Clock）

**SYSCLK（系统时钟）**：
- CPU 的工作频率
- **STM32F103 最大 72MHz**

**配置示例**（在 CubeMX 中）：
```
Clock Configuration:
- System Clock Mux: PLLCLK
→ 系统时钟使用 PLL 输出（72MHz）
```

#### 4. 总线时钟配置（Bus Clock）

**AHB 总线**：
- 连接 CPU、内存、DMA
- 通常等于系统时钟（72MHz）

**APB1 总线**：
- 连接 TIM2、TIM3、USART2 等
- **最大 36MHz**（系统时钟 ÷ 2）

**APB2 总线**：
- 连接 GPIO、TIM1、USART1 等
- 通常等于系统时钟（72MHz）

**配置示例**（在 CubeMX 中）：
```
Clock Configuration:
- AHB Prescaler: /1  → 72MHz
- APB1 Prescaler: /2 → 36MHz
- APB2 Prescaler: /1 → 72MHz
```

**为什么 APB1 要分频？**
- STM32F103 规定：APB1 最大频率 36MHz
- 如果系统时钟是 72MHz，必须 ÷2

#### 5. 定时器时钟的特殊规则

**重要规则**：
- 如果 APB 预分频器 = 1，定时器时钟 = APB 时钟
- 如果 APB 预分频器 ≠ 1，定时器时钟 = APB 时钟 × 2

**实际计算**（本项目）：
```
系统时钟：72MHz
APB1 预分频器：÷2
APB1 时钟：36MHz
APB1 预分频器 ≠ 1
→ TIM2/TIM3 时钟 = 36MHz × 2 = 72MHz ✓
```

### 📈 时钟配置的完整流程

#### 在 CubeMX 中配置

**步骤1：选择时钟源**
```
RCC → High Speed Clock (HSE) → Crystal/Ceramic Resonator
```

**步骤2：配置 PLL**
```
Clock Configuration 标签页：
- PLL Source Mux: HSE
- PLL Multiplier: x9
```

**步骤3：选择系统时钟**
```
System Clock Mux: PLLCLK
```

**步骤4：配置总线分频**
```
AHB Prescaler: /1
APB1 Prescaler: /2
APB2 Prescaler: /1
```

**步骤5：查看结果**
```
右侧显示各时钟频率：
- SYSCLK: 72 MHz
- HCLK: 72 MHz
- PCLK1: 36 MHz
- PCLK2: 72 MHz
```

#### 生成的代码

**在 `main.c` 中的 `SystemClock_Config()` 函数**：
```c
void SystemClock_Config(void)
{
    // 1. 配置振荡器
    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;  // 使用外部晶振
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;                   // 启用 HSE
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;                // 启用 PLL
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;        // PLL 源：HSE
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;                // 倍频：×9
    
    // 2. 配置系统时钟
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;   // 系统时钟：PLL
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;          // AHB：÷1
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2;           // APB1：÷2
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;           // APB2：÷1
}
```

### ⚠️ 时钟配置的注意事项

#### 1. Flash 等待周期（Flash Latency）

**问题**：CPU 速度太快，Flash 跟不上

**解决**：配置 Flash 等待周期

**规则**：
```
系统时钟 ≤ 24MHz → Latency = 0
24MHz < 系统时钟 ≤ 48MHz → Latency = 1
48MHz < 系统时钟 ≤ 72MHz → Latency = 2
```

**配置**（CubeMX 自动处理）：
```
Flash Latency: 2 Wait States
→ 在代码中：FLASH_LATENCY_2
```

#### 2. 时钟精度

**外部晶振（HSE）**：
- 精度：±20ppm（百万分之二十）
- 适合：需要精确计时的应用

**内部振荡器（HSI）**：
- 精度：±1%（较差）
- 适合：对精度要求不高的应用

#### 3. 功耗考虑

**高频率（72MHz）**：
- 性能高，但功耗大
- 适合：需要高性能的应用

**低频率（8MHz）**：
- 性能低，但功耗小
- 适合：电池供电的应用

---

## 🔧 其他重要配置

### 1. 调试接口配置（SYS）

**作用**：配置如何连接调试器（ST-Link、J-Link）

**配置**：
```
SYS → Debug → Serial Wire (SWD)
→ 使用 SWD 接口调试
```

**为什么重要？**
- 不配置调试接口，无法下载程序
- 无法使用调试功能（断点、单步等）

### 2. 定时器配置（TIM）

**在 CubeMX 中配置 TIM2**：
```
TIM2 → Channel1 → PWM Generation CH1
→ 配置参数：
  - Prescaler: 35
  - Counter Period: 19999
  - Pulse: 1500
```

**生成的代码**：
```c
void MX_TIM2_Init(void)
{
    htim2.Init.Prescaler = 35;      // 预分频器
    htim2.Init.Period = 19999;       // 周期
    // ...
    sConfigOC.Pulse = 1500;          // 初始占空比
}
```

### 3. 中断优先级配置（NVIC）

**在 CubeMX 中配置**：
```
TIM3 → NVIC Settings → TIM3 global interrupt → Enable
→ Priority: 0 (最高优先级)
```

**生成的代码**：
```c
void HAL_TIM_Base_MspInit(TIM_HandleTypeDef* tim_baseHandle)
{
    HAL_NVIC_SetPriority(TIM3_IRQn, 0, 0);  // 设置优先级
    HAL_NVIC_EnableIRQ(TIM3_IRQn);          // 使能中断
}
```

### 4. 项目设置（Project Manager）

**Toolchain/IDE**：
```
Makefile
→ 生成 Makefile 项目（用于 VS Code）
```

**Code Generator**：
```
✅ Generate peripheral initialization as a pair of '.c/.h' files
✅ Keep User Code when re-generating
→ 保留用户代码，避免被覆盖
```

---

## 📊 配置与代码的对应关系

### 配置流程图

```
STM32CubeMX 配置
    ↓
[引脚配置] → 生成 gpio.c, tim.c 中的引脚初始化代码
[时钟配置] → 生成 main.c 中的 SystemClock_Config()
[定时器配置] → 生成 tim.c 中的 MX_TIM2_Init(), MX_TIM3_Init()
[中断配置] → 生成 stm32f1xx_it.c 中的中断处理函数
    ↓
点击 "Generate Code"
    ↓
自动生成所有初始化代码
    ↓
你只需要写业务逻辑代码
```

### 实际例子

#### 例子1：配置 PA0 为 PWM 输出

**CubeMX 操作**：
1. 点击 PA0 引脚
2. 选择 "TIM2_CH1"
3. 配置 TIM2 参数

**生成的代码**（`tim.c`）：
```c
void HAL_TIM_MspPostInit(TIM_HandleTypeDef* timHandle)
{
    // 自动生成：配置 PA0 为 TIM2_CH1
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;  // 复用推挽输出
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
}
```

**你不需要写这些代码！** CubeMX 自动生成。

#### 例子2：配置系统时钟为 72MHz

**CubeMX 操作**：
1. RCC → 启用 HSE
2. Clock Configuration → 配置 PLL ×9
3. 选择系统时钟源为 PLL

**生成的代码**（`main.c`）：
```c
void SystemClock_Config(void)
{
    // 自动生成：完整的时钟配置代码
    RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    // ... 几十行配置代码
}
```

**你不需要写这些代码！** CubeMX 自动生成。

---

## 🎯 配置检查清单

### ✅ 引脚配置检查

- [ ] 所有使用的引脚都已配置
- [ ] 没有引脚冲突（同一引脚配置了多个功能）
- [ ] 引脚功能正确（PWM 用定时器通道，LED 用 GPIO）

### ✅ 时钟配置检查

- [ ] HSE 已启用（如果使用外部晶振）
- [ ] PLL 配置正确（倍频到 72MHz）
- [ ] 系统时钟选择正确（PLLCLK）
- [ ] 总线分频正确（APB1 ÷2，其他 ÷1）
- [ ] Flash Latency 正确（72MHz 需要 2）

### ✅ 外设配置检查

- [ ] 定时器参数正确（PSC、ARR、Pulse）
- [ ] 中断已启用（如果需要中断）
- [ ] 调试接口已配置（SWD 或 JTAG）

### ✅ 项目设置检查

- [ ] Toolchain 选择正确（Makefile）
- [ ] 代码生成选项正确（保留用户代码）

---

## 💡 配置技巧

### 技巧1：使用引脚搜索

**在 CubeMX 中**：
- 按 `Ctrl+F` 搜索引脚功能
- 例如：搜索 "TIM2_CH1"，快速找到可用引脚

### 技巧2：查看冲突

**在 CubeMX 中**：
- 已配置的引脚显示为**绿色**
- 冲突的引脚显示为**红色**
- 悬停查看详细信息

### 技巧3：使用时钟树视图

**在 CubeMX 中**：
- Clock Configuration 标签页显示完整时钟树
- 右侧实时显示各时钟频率
- 红色表示配置错误，需要调整

### 技巧4：保存配置模板

**在 CubeMX 中**：
- File → Save Project
- 保存 `.ioc` 文件
- 下次可以直接打开，复用配置

---

## 🎓 总结

### 核心要点

1. **STM32CubeMX = 图形化配置工具**
   - 不需要手写复杂的寄存器配置
   - 图形界面选择，自动生成代码

2. **引脚配置 = 告诉芯片引脚做什么**
   - GPIO：简单输入/输出
   - 复用功能：连接内部外设
   - 模拟功能：ADC 输入

3. **时钟配置 = 告诉芯片跑多快**
   - 外部晶振 → PLL 倍频 → 系统时钟
   - 系统时钟分频到各个总线
   - 定时器时钟有特殊规则

4. **配置影响代码生成**
   - 每个配置都会生成对应的初始化代码
   - 你只需要写业务逻辑

### 配置流程

```
打开 CubeMX
    ↓
配置引脚（选择功能）
    ↓
配置时钟（选择频率）
    ↓
配置外设（定时器、串口等）
    ↓
生成代码
    ↓
在生成的代码基础上写业务逻辑
```

---

## 项目概述

### 🎯 项目功能

这个项目实现了：
1. **使用定时器中断自动控制舵机**：舵机在 0° 和 180° 之间自动来回摆动
2. **LED 闪烁**（代码已配置，可取消注释使用）

### 🔧 使用的硬件资源

| 资源 | 用途 | 说明 |
|------|------|------|
| **TIM2** | PWM 输出 | 产生 50Hz PWM 信号控制舵机（PA0 引脚） |
| **TIM3** | 定时中断 | 每 100ms 触发一次中断，更新舵机角度 |
| **PC13** | LED 控制 | 板载 LED（代码已配置，可取消注释） |

### 📊 工作原理简述

```
TIM3 定时器（每100ms中断一次）
    ↓
触发中断回调函数
    ↓
更新 TIM2 的 PWM 占空比
    ↓
舵机角度改变
    ↓
舵机自动来回摆动
```

---

## 核心概念：什么是中断？

### 🤔 生活中的类比

想象你在家里做作业，突然门铃响了：

**没有中断的情况**（轮询方式）：
```
你：写作业 → 检查门铃 → 写作业 → 检查门铃 → 写作业 → 检查门铃...
问题：你无法专心写作业，必须不停地检查门铃
```

**有中断的情况**：
```
你：专心写作业...
门铃：叮咚！（中断发生）
你：暂停作业，去开门（处理中断）
你：回到座位，继续写作业（恢复原任务）
```

### 💡 中断的本质

**中断 = 硬件自动"打断"CPU，让CPU去处理紧急事件**

- **正常情况**：CPU 按顺序执行程序（main 函数的 while 循环）
- **中断发生时**：CPU 暂停当前任务，跳转到中断处理函数
- **中断处理完**：CPU 自动回到原来的位置继续执行

### 🎯 为什么使用中断？

**不使用中断（轮询方式）**：
```c
while(1) {
    // 必须不停地检查定时器
    if (定时器到了) {
        更新舵机角度();
    }
    // 无法做其他事情，CPU 被占用
}
```

**使用中断**：
```c
// 主程序可以做其他事情
while(1) {
    // 可以处理其他任务
    // 定时器到了会自动调用中断函数
}

// 中断函数自动执行
void 中断函数() {
    更新舵机角度();  // 自动执行，不占用主循环
}
```

**优势**：
- ✅ CPU 可以做其他事情
- ✅ 定时精确（硬件自动计时）
- ✅ 代码结构清晰

---

## 核心概念：什么是回调函数？

### 🤔 生活中的类比

想象你点外卖：

```
你：点外卖，留下电话号码
外卖员：做好后，自动打电话给你（回调）
你：接到电话，去取外卖（执行回调函数）
```

### 💡 回调函数的本质

**回调函数 = 你提前写好一个函数，让系统在特定时候自动调用它**

- **你写的函数**：`HAL_TIM_PeriodElapsedCallback()`
- **系统自动调用**：当定时器溢出时，HAL 库自动调用这个函数
- **你不需要手动调用**：系统会在合适的时候自动执行

### 📝 回调函数的工作流程

```
1. 你定义回调函数
   ↓
2. HAL 库注册这个函数（自动完成）
   ↓
3. 定时器溢出，触发中断
   ↓
4. HAL 库的中断处理函数执行
   ↓
5. HAL 库自动调用你的回调函数
   ↓
6. 你的回调函数执行完毕
   ↓
7. 返回到主程序继续执行
```

---

## 硬件配置说明

### ⚙️ TIM2 配置（PWM 输出）

**作用**：产生 PWM 信号控制舵机

**配置参数**：
- **Prescaler (PSC)** = 35
- **Period (ARR)** = 19999
- **输出引脚**：PA0 (TIM2_CH1)

**计算**：
```
系统时钟：72MHz
APB1 时钟：36MHz
TIM2 时钟：72MHz（APB1 × 2）

定时器时钟 = 72MHz ÷ (35+1) = 2MHz
PWM 频率 = 2MHz ÷ (19999+1) = 100Hz

但实际应该是 50Hz，这里可能是配置有误，应该是：
PSC = 71, Period = 19999 → 50Hz
```

**实际配置**（根据代码）：
- 定时器时钟 = 72MHz ÷ 36 = 2MHz
- PWM 频率 = 2MHz ÷ 20000 = 100Hz（周期 10ms）

> **注意**：标准舵机需要 50Hz（20ms 周期），当前配置是 100Hz，可能需要调整

### ⚙️ TIM3 配置（定时中断）

**作用**：每 100ms 触发一次中断，更新舵机角度

**配置参数**：
- **Prescaler (PSC)** = 71
- **Period (ARR)** = 9999

**计算**：
```
定时器时钟 = 72MHz ÷ (71+1) = 1MHz
中断频率 = 1MHz ÷ (9999+1) = 100Hz
中断周期 = 1 ÷ 100Hz = 10ms
```

> **注意**：代码注释说每 100ms，但实际计算是 10ms。如果要 100ms，应该设置 Period = 99999

---

## 代码逐行详解

### 📄 main.c 文件解析

#### 1. 头文件包含

```c
#include "main.h"      // 主头文件，包含 HAL 库定义
#include "tim.h"       // 定时器相关函数和变量
#include "gpio.h"      // GPIO 相关函数
#include "stdbool.h"   // 布尔类型支持（true/false）
```

**作用**：引入必要的库和定义

---

#### 2. 中断回调函数（核心代码）

```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
```

**函数名解释**：
- `HAL_TIM_PeriodElapsedCallback` = HAL 库的定时器周期结束回调函数
- 这是 **HAL 库规定的函数名**，不能随意更改
- 当任何定时器溢出时，HAL 库会自动调用这个函数

**参数解释**：
- `htim`：指向触发中断的定时器句柄的指针
- 通过这个参数，可以知道是哪个定时器触发了中断

---

```c
if(htim == (&htim3))
```

**作用**：判断是否是 TIM3 触发的中断

**为什么需要判断？**
- 因为系统中可能有多个定时器（TIM2、TIM3）
- 这个回调函数会被所有定时器调用
- 我们需要只处理 TIM3 的中断

**`&htim3` 解释**：
- `htim3` 是定时器3的句柄（在 tim.c 中定义）
- `&htim3` 是取地址，得到指向 htim3 的指针
- 比较指针地址，判断是否是同一个定时器

---

```c
static uint16_t cnt = 1999;
```

**作用**：定义 PWM 占空比的计数值

**`static` 关键字**：
- 表示这个变量是**静态变量**
- 静态变量的特点：**函数执行完后，变量值不会消失**
- 下次调用函数时，变量保持上次的值

**为什么用 static？**
```c
// 不用 static（错误）
void 函数() {
    uint16_t cnt = 1999;  // 每次调用都重新初始化为 1999
    cnt++;  // 永远都是 1999 → 2000，无法累加
}

// 用 static（正确）
void 函数() {
    static uint16_t cnt = 1999;  // 第一次初始化为 1999
    cnt++;  // 第一次：1999 → 2000
            // 第二次：2000 → 2001（保持上次的值）
}
```

**初始值 1999**：
- 对应舵机的某个角度（需要根据实际 PWM 配置计算）
- 范围应该在 500-2500 之间（对应 0.5ms-2.5ms）

---

```c
static bool dir = 0;
```

**作用**：控制舵机转动方向

**`bool` 类型**：
- `true` (1)：增加角度
- `false` (0)：减少角度

**`static` 作用**：保持方向状态，不会每次重置

---

```c
__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, cnt);
```

**作用**：设置 TIM2 通道1 的 PWM 占空比

**函数解释**：
- `__HAL_TIM_SET_COMPARE`：HAL 库的宏定义，设置比较值
- `&htim2`：TIM2 定时器句柄的地址
- `TIM_CHANNEL_1`：通道1（对应 PA0 引脚）
- `cnt`：比较值（决定 PWM 占空比）

**工作原理**：
```
cnt 值越大 → PWM 高电平时间越长 → 舵机角度越大
cnt = 1999 → 某个角度
cnt = 3999 → 更大角度
```

---

```c
if(dir == 0)
{
    if (cnt < 3999)
        cnt+=10;
    else
        dir = 1;
}
```

**作用**：增加角度（dir = 0 时）

**逻辑解释**：
```
如果方向是增加（dir = 0）：
    如果当前角度还没到最大值（cnt < 3999）：
        角度增加 10（cnt += 10）
    否则（已经到最大值）：
        改变方向为减少（dir = 1）
```

**为什么每次加 10？**
- 控制舵机转动的速度
- 加得越大，转动越快
- 加得越小，转动越平滑

---

```c
else
{
    if (cnt > 1999)
        cnt-=10;
    else
        dir = 0;
}
```

**作用**：减少角度（dir = 1 时）

**逻辑解释**：
```
如果方向是减少（dir = 1）：
    如果当前角度还没到最小值（cnt > 1999）：
        角度减少 10（cnt -= 10）
    否则（已经到最小值）：
        改变方向为增加（dir = 0）
```

**完整循环**：
```
1999 → 2009 → 2019 → ... → 3999 (增加)
3999 → 3989 → 3979 → ... → 1999 (减少)
1999 → 2009 → ... (循环往复)
```

---

#### 3. main 函数解析

```c
int main(void)
```

**作用**：程序入口函数

**`void` 参数**：表示不需要参数

---

```c
HAL_Init();
```

**作用**：初始化 HAL 库

**具体做了什么**：
- 配置系统时钟基准（SysTick）
- 设置中断优先级分组
- 初始化底层硬件

---

```c
SystemClock_Config();
```

**作用**：配置系统时钟为 72MHz

**具体配置**：
- 启用外部晶振（HSE）
- 配置锁相环（PLL）
- 设置系统时钟、AHB、APB 时钟

---

```c
MX_GPIO_Init();
```

**作用**：初始化 GPIO 引脚

**具体配置**（在 gpio.c 中）：
- 使能 GPIO 时钟
- 配置 PC13 为输出模式（LED）
- 配置 PA0 为复用功能（TIM2_CH1，PWM 输出）

---

```c
MX_TIM2_Init();
```

**作用**：初始化 TIM2（PWM 输出）

**具体配置**（在 tim.c 中）：
- 设置预分频器和周期
- 配置 PWM 模式
- 配置 PA0 引脚为 TIM2_CH1

---

```c
MX_TIM3_Init();
```

**作用**：初始化 TIM3（定时中断）

**具体配置**（在 tim.c 中）：
- 设置预分频器和周期
- 配置中断使能

---

```c
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
```

**作用**：启动 TIM2 的 PWM 输出

**参数解释**：
- `&htim2`：TIM2 定时器句柄
- `TIM_CHANNEL_1`：通道1

**执行后**：
- TIM2 开始工作
- PA0 引脚开始输出 PWM 信号
- 舵机开始接收控制信号

---

```c
HAL_TIM_Base_Start_IT(&htim3);
```

**作用**：启动 TIM3 的定时中断

**`_IT` 后缀**：
- `IT` = Interrupt（中断）
- 表示启动定时器并启用中断

**执行后**：
- TIM3 开始计数
- 当计数器溢出时，触发中断
- 自动调用 `HAL_TIM_PeriodElapsedCallback()`

---

```c
while (1)
{
    // 主循环（当前为空）
}
```

**作用**：主程序循环

**为什么是空的？**
- 因为所有工作都在中断回调函数中完成
- 主循环可以添加其他任务（如 LED 闪烁、串口通信等）

**可以添加的代码**（取消注释）：
```c
HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);  // LED 闪烁
HAL_Delay(500);  // 延时 500ms
```

---

### 📄 tim.c 文件解析（关键部分）

#### TIM2 初始化

```c
htim2.Instance = TIM2;
htim2.Init.Prescaler = 35;
htim2.Init.Period = 19999;
```

**作用**：配置 TIM2 的时钟分频和周期

---

#### TIM3 初始化

```c
htim3.Instance = TIM3;
htim3.Init.Prescaler = 71;
htim3.Init.Period = 9999;
```

**作用**：配置 TIM3 的时钟分频和周期

**中断配置**（在 HAL_TIM_Base_MspInit 中）：
```c
HAL_NVIC_SetPriority(TIM3_IRQn, 0, 0);  // 设置中断优先级
HAL_NVIC_EnableIRQ(TIM3_IRQn);          // 使能中断
```

---

### 📄 stm32f1xx_it.c 文件解析

```c
void TIM3_IRQHandler(void)
{
    HAL_TIM_IRQHandler(&htim3);
}
```

**作用**：TIM3 的中断处理函数

**工作流程**：
1. TIM3 溢出，硬件触发中断
2. CPU 跳转到 `TIM3_IRQHandler()`
3. 调用 `HAL_TIM_IRQHandler()`（HAL 库函数）
4. HAL 库检查中断类型，调用相应的回调函数
5. 如果是周期溢出，调用 `HAL_TIM_PeriodElapsedCallback()`

---

## 完整程序执行流程

### 🚀 启动阶段

```
1. 上电/复位
   ↓
2. 启动文件执行（startup_stm32f103xb.s）
   - 初始化堆栈指针
   - 初始化程序计数器
   ↓
3. 调用 SystemInit()
   - 配置系统时钟
   ↓
4. 跳转到 main() 函数
```

### ⚙️ 初始化阶段

```
main() 函数开始执行
   ↓
HAL_Init()
   - 初始化 HAL 库
   - 配置 SysTick（用于 HAL_Delay）
   ↓
SystemClock_Config()
   - 配置系统时钟为 72MHz
   ↓
MX_GPIO_Init()
   - 配置 PC13 为输出（LED）
   - 配置 PA0 为复用功能（TIM2_CH1）
   ↓
MX_TIM2_Init()
   - 配置 TIM2 为 PWM 模式
   - PSC = 35, ARR = 19999
   ↓
MX_TIM3_Init()
   - 配置 TIM3 为定时器模式
   - PSC = 71, ARR = 9999
   - 使能中断
   ↓
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1)
   - 启动 TIM2，开始输出 PWM
   - PA0 引脚开始输出 PWM 信号
   ↓
HAL_TIM_Base_Start_IT(&htim3)
   - 启动 TIM3，开始计数
   - 使能中断
   ↓
进入 while(1) 主循环
```

### 🔄 运行阶段（中断驱动）

```
主循环 while(1)
   ↓
（CPU 可以执行其他任务，或进入低功耗模式）
   ↓
TIM3 计数器：0 → 1 → 2 → ... → 9999 → 溢出
   ↓
硬件自动触发中断
   ↓
CPU 暂停主循环，跳转到 TIM3_IRQHandler()
   ↓
TIM3_IRQHandler() 调用 HAL_TIM_IRQHandler(&htim3)
   ↓
HAL 库检查中断类型，发现是周期溢出
   ↓
HAL 库自动调用 HAL_TIM_PeriodElapsedCallback(&htim3)
   ↓
执行回调函数中的代码：
   1. 判断是 TIM3 触发（if(htim == &htim3)）
   2. 更新 PWM 占空比（__HAL_TIM_SET_COMPARE）
   3. 根据方向增加或减少角度值
   ↓
回调函数执行完毕
   ↓
CPU 自动返回到主循环，继续执行
   ↓
等待下一次中断...
```

### 📊 完整时序图

```
时间轴 →
─────────────────────────────────────────────────────────
主程序:  [初始化] [启动PWM] [启动TIM3] [主循环] [主循环] ...
─────────────────────────────────────────────────────────
TIM3计数: 0 ──→ 9999 ──→ 溢出 ──→ 0 ──→ 9999 ──→ ...
            │              │              │
            └─中断触发     └─中断触发     └─中断触发
─────────────────────────────────────────────────────────
中断处理:              [回调函数]        [回调函数]
                      更新PWM          更新PWM
─────────────────────────────────────────────────────────
TIM2 PWM:  [持续输出PWM信号，占空比根据cnt值变化]
PA0输出:   ┌─┐        ┌──┐       ┌────┐      ┌──┐
           │ │        │  │       │    │      │  │
           └─┴────────┴──┴───────┴────┴──────┴──┴───
           cnt=1999  cnt=2009  cnt=3999  cnt=3989
─────────────────────────────────────────────────────────
舵机动作:  角度A      角度B     角度C      角度B
          (来回摆动)
─────────────────────────────────────────────────────────
```

---

## 时序图详解

### 🕐 详细时间线

假设 TIM3 每 10ms 中断一次（根据当前配置）：

```
时间(ms)  事件                        说明
─────────────────────────────────────────────────────────
0         系统启动                   上电
1         HAL_Init()                 初始化 HAL 库
2         SystemClock_Config()       配置时钟
3         MX_GPIO_Init()             配置 GPIO
4         MX_TIM2_Init()             配置 TIM2
5         MX_TIM3_Init()             配置 TIM3
6         HAL_TIM_PWM_Start()        启动 PWM 输出
7         HAL_TIM_Base_Start_IT()    启动 TIM3 中断
8         进入主循环                  while(1)
9         TIM3 计数中...              0 → 9999
10        TIM3 溢出，触发中断         第一次中断
11        执行回调函数                cnt = 1999 → 2009
12        返回主循环                  继续执行
13        TIM3 计数中...              0 → 9999
20        TIM3 溢出，触发中断         第二次中断
21        执行回调函数                cnt = 2009 → 2019
22        返回主循环                  继续执行
...       ...                        ...
```

### 📈 角度变化图

```
角度值 (cnt)
    ↑
3999│                    ╱╲
    │                   ╱  ╲
    │                  ╱    ╲
    │                 ╱      ╲
    │                ╱        ╲
    │               ╱          ╲
    │              ╱            ╲
1999│─────────────╱              ╲─────────────
    │
    └─────────────────────────────────────────→ 时间
     增加方向(dir=0)    减少方向(dir=1)
```

---

## 常见问题解答

### ❓ 为什么使用两个定时器？

**TIM2**：专门用于 PWM 输出
- 硬件自动生成 PWM 信号
- 不占用 CPU 资源
- 精度高，稳定

**TIM3**：专门用于定时中断
- 定期触发中断
- 在中断中更新 PWM 占空比
- 实现自动控制

### ❓ 为什么用中断而不是主循环？

**主循环方式的问题**：
```c
while(1) {
    if (时间到了) {
        更新角度();
    }
    // 必须不停地检查，浪费 CPU
}
```

**中断方式的优势**：
- ✅ 精确：硬件自动计时，不依赖软件延时
- ✅ 高效：CPU 可以做其他事情
- ✅ 实时：中断立即响应

### ❓ static 变量为什么重要？

**不用 static**：
```c
void 函数() {
    uint16_t cnt = 1999;  // 每次都是 1999
    cnt++;  // 永远无法累加
}
```

**用 static**：
```c
void 函数() {
    static uint16_t cnt = 1999;  // 保持上次的值
    cnt++;  // 可以累加：1999 → 2009 → 2019 → ...
}
```

### ❓ 如何调整舵机转动速度？

**方法1：改变每次增减的步长**
```c
cnt += 10;  // 慢速（每次 10）
cnt += 50;  // 快速（每次 50）
```

**方法2：改变中断频率**
```c
// TIM3 配置
Period = 9999;   // 10ms 中断（快速）
Period = 99999;  // 100ms 中断（慢速）
```

### ❓ 如何调整舵机角度范围？

**修改 cnt 的范围**：
```c
if (cnt < 3999)  // 最大值
if (cnt > 1999)  // 最小值
```

**对应角度计算**：
```
如果 PWM 配置是 50Hz（20ms 周期）：
- cnt = 500  → 0.5ms → 0°
- cnt = 1500 → 1.5ms → 90°
- cnt = 2500 → 2.5ms → 180°

当前配置（100Hz，10ms 周期）：
- cnt = 1999 → 约 1ms → 某个角度
- cnt = 3999 → 约 2ms → 更大角度
```

### ❓ 如何添加 LED 闪烁？

**取消注释主循环中的代码**：
```c
while (1)
{
    HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);  // 翻转 LED 状态
    HAL_Delay(500);  // 延时 500ms
}
```

**或者在中断回调函数中添加**：
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if(htim == (&htim3)) {
        // 更新舵机
        // ...
        
        // LED 闪烁
        HAL_GPIO_TogglePin(GPIOC, GPIO_PIN_13);
    }
}
```

---

## 🎯 总结

### 核心要点

1. **中断 = 硬件自动打断 CPU，执行紧急任务**
2. **回调函数 = 你写的函数，系统自动调用**
3. **static 变量 = 保持上次的值，不会重置**
4. **TIM2 = PWM 输出，控制舵机**
5. **TIM3 = 定时中断，定期更新角度**

### 程序流程

```
初始化 → 启动定时器 → 进入主循环
         ↓
    定时器溢出
         ↓
    触发中断
         ↓
    执行回调函数（更新角度）
         ↓
    返回主循环
         ↓
    循环往复
```

### 关键代码位置

| 功能 | 文件 | 位置 |
|------|------|------|
| 中断回调函数 | main.c | 第 59 行 |
| 定时器初始化 | tim.c | MX_TIM2_Init(), MX_TIM3_Init() |
| 中断处理 | stm32f1xx_it.c | TIM3_IRQHandler() |
| 主程序 | main.c | main() 函数 |

---

**祝你学习愉快！** 🎉

如有疑问，请参考：
- [STM32 HAL 库文档](https://www.st.com/resource/en/user_manual/um1725-description-of-stm32f1-hal-and-lowlayer-drivers-stmicroelectronics.pdf)
- [STM32F103 参考手册](https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf)

