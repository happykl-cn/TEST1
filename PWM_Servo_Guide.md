# 开发实例在 PA0 引脚实现 PWM 控制舵机指南

## 📋 概述

本指南将帮助你在 STM32F103 的 **PA0** 引脚上配置 PWM 输出，用于控制舵机。

### 舵机控制原理

- **频率**：50Hz（周期 20ms）
- **脉宽范围**：
  - 0.5ms（0°）
  - 1.5ms（90°）
  - 2.5ms（180°）
- **占空比范围**：2.5% - 12.5%

### 引脚映射

对于 STM32F103，**PA0** 可以映射到 **TIM2_CH1**（定时器2通道1）。

---

## 📖 原理篇：定时器与 PWM 详解

### 🕐 什么是定时器（Timer）？

定时器是微控制器中的一个重要外设，可以理解为**一个自动计数的"秒表"**。

#### 定时器的基本概念

想象一下你有一个秒表：
- **计数器（CNT）**：就像秒表上的数字，从 0 开始不断递增
- **预分频器（PSC）**：控制"秒表"走多快（比如每 1 微秒走一步，还是每 10 微秒走一步）
- **自动重装载寄存器（ARR）**：设置"秒表"的最大值，到达这个值后自动归零重新开始

#### 定时器的工作原理

```
系统时钟 (72MHz)
    ↓
[预分频器 PSC] → 定时器时钟 (1MHz)
    ↓
[计数器 CNT] → 0, 1, 2, 3, ..., ARR, 0, 1, 2, ...
    ↓
到达 ARR 后自动归零，产生更新事件
```

**举例说明**：
- 系统时钟：72MHz（每秒 72,000,000 次）
- 预分频器 PSC = 71：定时器时钟 = 72MHz ÷ (71+1) = 1MHz（每秒 1,000,000 次）
- 自动重装载 ARR = 19999：计数器从 0 数到 19999，然后归零
- 一个周期时间 = (19999+1) ÷ 1MHz = 20ms
- 频率 = 1 ÷ 20ms = 50Hz

#### 定时器的功能

定时器不仅可以计时，还可以：
1. **产生 PWM 信号**（脉宽调制）
2. **捕获输入信号**（测量脉冲宽度）
3. **产生定时中断**（定时执行某些任务）
4. **输出比较**（在特定时间点触发事件）

---

### 📊 什么是 PWM（脉宽调制）？

PWM（Pulse Width Modulation，脉宽调制）是一种**通过改变脉冲宽度来控制信号的技术**。

#### PWM 的基本概念

想象你在快速开关一盏灯：
- **周期（Period）**：开关一次完整循环的时间（比如 20ms）
- **高电平时间（Pulse Width）**：灯亮的时间（比如 1.5ms）
- **低电平时间**：灯灭的时间（18.5ms）
- **占空比（Duty Cycle）**：高电平时间占整个周期的百分比

```
占空比 = (高电平时间 / 周期) × 100%
```

#### PWM 波形示意图

```
高电平 ─────┐         ┌─────────
            │         │
            │         │
低电平      └─────────┘         └───
            ←── 周期 ──→
            ←高电平→
```

**示例**（50Hz PWM，周期 20ms）：
- **0° 位置**：高电平 0.5ms，占空比 2.5%
- **90° 位置**：高电平 1.5ms，占空比 7.5%
- **180° 位置**：高电平 2.5ms，占空比 12.5%

#### PWM 如何控制舵机？

舵机内部有一个**控制电路**，它会：
1. **检测 PWM 信号的高电平时间**
2. **根据高电平时间确定目标角度**
3. **驱动电机转动到对应角度**

```
PWM 信号 → 舵机控制电路 → 电机 → 舵机转动
```

**为什么是 50Hz？**
- 舵机需要**持续接收 PWM 信号**来保持位置
- 50Hz（每 20ms 一次）是标准频率，既能保持稳定，又不会太频繁

#### 定时器如何产生 PWM？

定时器通过**比较寄存器（CCR）**来控制 PWM：

```
计数器 CNT:  0 ──→ 500 ──→ 1500 ──→ 19999 ──→ 0 ──→ ...
             │      │       │        │
             │      │       │        │
比较值 CCR:  1500 (设置的值)
             │
             └─→ 当 CNT < CCR 时，输出高电平
                 当 CNT ≥ CCR 时，输出低电平
```

**工作流程**：
1. 计数器从 0 开始递增
2. 当计数器值 < CCR 时，输出**高电平**
3. 当计数器值 ≥ CCR 时，输出**低电平**
4. 计数器到达 ARR 后归零，开始新的周期

**改变角度**：
- 改变 CCR 的值 → 改变高电平时间 → 改变舵机角度
- CCR = 500 → 高电平 0.5ms → 0°
- CCR = 1500 → 高电平 1.5ms → 90°
- CCR = 2500 → 高电平 2.5ms → 180°

---

### 🔄 整个程序运行流程

#### 1. 系统启动阶段

```
上电/复位
    ↓
启动文件 (startup_stm32f103xb.s)
    ↓
初始化堆栈指针 (SP)
    ↓
初始化程序计数器 (PC) → 跳转到 Reset_Handler
    ↓
调用 SystemInit() → 配置系统时钟
    ↓
调用 main() 函数
```

**关键点**：
- 启动文件负责最基础的初始化
- 系统时钟配置（72MHz）
- 跳转到用户程序入口 `main()`

#### 2. 初始化阶段（main 函数开始）

```c
int main(void)
{
    // 步骤 1: HAL 库初始化
    HAL_Init();
    // ↓ 初始化 SysTick（用于 HAL_Delay）
    // ↓ 配置中断优先级
    
    // 步骤 2: 系统时钟配置
    SystemClock_Config();
    // ↓ 配置 HSE（外部晶振）
    // ↓ 配置 PLL（锁相环）
    // ↓ 设置系统时钟为 72MHz
    
    // 步骤 3: 外设初始化
    MX_GPIO_Init();      // 初始化 GPIO
    MX_TIM2_Init();      // 初始化定时器2（PWM）
    // ↓ 配置引脚功能
    // ↓ 配置定时器参数（PSC, ARR, CCR）
    // ↓ 使能定时器时钟
    
    // 步骤 4: 启动 PWM
    HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
    // ↓ 启动定时器计数器
    // ↓ 使能 PWM 输出
    
    // 步骤 5: 进入主循环
    while(1) { ... }
}
```

#### 3. PWM 生成流程（硬件自动执行）

```
定时器硬件自动工作（无需 CPU 干预）
    ↓
系统时钟 (72MHz) → 预分频器 (PSC=71) → 定时器时钟 (1MHz)
    ↓
计数器 CNT 从 0 开始递增
    ↓
比较器比较：CNT vs CCR
    ↓
    ├─ CNT < CCR → 输出高电平 (PA0 = 1)
    └─ CNT ≥ CCR → 输出低电平 (PA0 = 0)
    ↓
CNT 到达 ARR (19999) → 自动归零 → 开始新周期
    ↓
重复上述过程（50Hz，每 20ms 一个周期）
```

**关键点**：
- PWM 生成是**硬件自动完成**的，不需要 CPU 持续参与
- CPU 只需要在需要改变角度时，修改 CCR 的值
- 定时器会持续输出 PWM 信号，直到被停止

#### 4. 控制舵机角度流程

```
用户调用 Servo_SetAngle(90)
    ↓
计算 CCR 值：pulse = 500 + (90 * 2000) / 180 = 1500
    ↓
写入 CCR 寄存器：__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, 1500)
    ↓
定时器硬件检测到 CCR 改变
    ↓
下一个 PWM 周期开始使用新的 CCR 值
    ↓
PWM 高电平时间变为 1.5ms
    ↓
舵机接收到新的 PWM 信号
    ↓
舵机转动到 90° 位置
```

#### 5. 完整时序图

```
时间轴 →
─────────────────────────────────────────────────────────
CPU 执行:  [初始化] [启动PWM] [设置角度] [延时] [设置角度] ...
          └─────────────┴─────────────┴─────────────┘
                        主循环
─────────────────────────────────────────────────────────
定时器 CNT: 0→500→1500→19999→0→500→1500→19999→0→...
─────────────────────────────────────────────────────────
PA0 输出:   ┌─┐                    ┌─┐
            │ │                    │ │
            └─┴────────────────────┴─┴──────────────
            ←0.5ms→              ←1.5ms→
            (0°)                  (90°)
─────────────────────────────────────────────────────────
舵机动作:   转动到0°              转动到90°
```

#### 6. 内存和寄存器映射

```
程序代码 (Flash)
    ├─ main() 函数
    ├─ Servo_SetAngle() 函数
    └─ HAL 库函数
    
运行时内存 (RAM)
    ├─ 变量：servo_angle
    └─ 定时器句柄：htim2
    
硬件寄存器 (外设)
    ├─ TIM2_CR1: 定时器控制寄存器
    ├─ TIM2_PSC: 预分频器寄存器 (71)
    ├─ TIM2_ARR: 自动重装载寄存器 (19999)
    ├─ TIM2_CCR1: 比较寄存器 (500-2500)
    └─ TIM2_CNT: 计数器寄存器 (0-19999)
```

---

### 🎯 关键理解点总结

1. **定时器 = 自动计数器**
   - 不需要 CPU 干预，硬件自动运行
   - 通过预分频器和自动重装载值控制频率

2. **PWM = 改变脉冲宽度**
   - 通过比较计数器和比较值产生 PWM
   - 改变比较值 = 改变占空比 = 控制舵机角度

3. **程序流程 = 初始化 + 循环**
   - 初始化：配置时钟、外设、启动 PWM
   - 循环：根据需求改变 PWM 占空比

4. **硬件 vs 软件**
   - **硬件**：定时器自动生成 PWM 信号（高效）
   - **软件**：只需要在需要时修改参数（灵活）

---

## 🔧 步骤一：在 STM32CubeMX 中配置

### 1.1 打开项目

1. 打开 `TEST1.ioc` 文件（用 STM32CubeMX 打开）

### 1.2 配置定时器

1. 在左侧引脚图中找到 **PA0** 引脚
2. 点击 **PA0**，选择 **TIM2_CH1**（定时器2通道1）
   - 如果看不到，可能需要先启用 TIM2

### 1.3 配置 TIM2 参数

1. 在左侧外设列表中找到 **TIM2**
2. 点击 **TIM2**，在右侧配置面板中：
   - **Mode** → **Channel1** → 选择 **PWM Generation CH1**
   - **Configuration** 标签页：
     - **Prescaler (PSC - 16 bits value)**：`71`（如果系统时钟是72MHz，72MHz / (71+1) = 1MHz）
     - **Counter Period (AutoReload Register - 16 bits value)**：`19999`（1MHz / 20000 = 50Hz，周期20ms）
     - **Pulse (16 bits value)**：`1500`（初始占空比，对应1.5ms，90°位置）
     - **auto-reload preload**：**Enable**

### 1.4 生成代码

1. 点击右上角 **GENERATE CODE**
2. 选择 **Yes** 保留用户代码

---

## 📝 步骤二：更新 Makefile

生成代码后，需要在 `Makefile` 中添加定时器相关的源文件。

### 2.1 添加 TIM 源文件

打开 `Makefile`，在 `C_SOURCES` 部分添加：

```makefile
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_tim.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_tim_ex.c \
```

**完整示例**（在 `stm32f1xx_hal_exti.c` 之后添加）：

```makefile
C_SOURCES =  \
Src/main.c \
Src/gpio.c \
Src/stm32f1xx_it.c \
Src/stm32f1xx_hal_msp.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_gpio_ex.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_rcc.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_rcc_ex.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_gpio.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_dma.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_cortex.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_pwr.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_flash.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_flash_ex.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_exti.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_tim.c \
Drivers/STM32F1xx_HAL_Driver/Src/stm32f1xx_hal_tim_ex.c \
Src/system_stm32f1xx.c \
Src/sysmem.c \
Src/syscalls.c
```

### 2.2 检查头文件路径

确保 `Makefile` 中的 `C_INCLUDES` 包含：

```makefile
C_INCLUDES =  \
-IInc \
-IDrivers/STM32F1xx_HAL_Driver/Inc/Legacy \
-IDrivers/STM32F1xx_HAL_Driver/Inc \
-IDrivers/CMSIS/Device/ST/STM32F1xx/Include \
-IDrivers/CMSIS/Include
```

---

## 💻 步骤三：编写代码

### 3.1 在 main.h 中添加定时器句柄

打开 `Inc/main.h`，在 `/* USER CODE BEGIN ET */` 和 `/* USER CODE END ET */` 之间添加：

```c
/* USER CODE BEGIN ET */
extern TIM_HandleTypeDef htim2;
/* USER CODE END ET */
```

### 3.2 在 main.c 中启动 PWM

打开 `Src/main.c`，在 `/* USER CODE BEGIN 2 */` 和 `/* USER CODE END 2 */` 之间添加：

```c
/* USER CODE BEGIN 2 */
// 启动 TIM2 的 PWM 输出
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);
/* USER CODE END 2 */
```

### 3.3 创建舵机控制函数

在 `main.c` 的 `/* USER CODE BEGIN 4 */` 和 `/* USER CODE END 4 */` 之间添加舵机控制函数：

```c
/* USER CODE BEGIN 4 */

/**
 * @brief  设置舵机角度
 * @param  angle: 角度值 (0-180)
 * @retval None
 */
void Servo_SetAngle(uint8_t angle)
{
    // 限制角度范围
    if (angle > 180) angle = 180;
    
    // 计算占空比对应的计数值
    // 脉宽范围：0.5ms - 2.5ms
    // 对应计数值：500 - 2500 (基于1MHz的定时器时钟)
    // 公式：计数值 = 500 + angle * (2500-500) / 180
    uint16_t pulse = 500 + (angle * 2000) / 180;
    
    // 设置 PWM 占空比
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, pulse);
}

/* USER CODE END 4 */
```

### 3.4 在主循环中测试

在 `main.c` 的 `while(1)` 循环中添加测试代码：

```c
/* USER CODE BEGIN WHILE */
uint8_t angle = 0;
uint8_t direction = 0;  // 0: 增加, 1: 减少

while (1)
{
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
    // 设置舵机角度
    Servo_SetAngle(angle);
    HAL_Delay(20);  // 延时20ms，等待舵机响应
    
    // 角度变化（0° <-> 180°）
    if (direction == 0)
    {
        angle++;
        if (angle >= 180) direction = 1;
    }
    else
    {
        angle--;
        if (angle == 0) direction = 0;
    }
}
/* USER CODE END 3 */
```

---

## 🔍 步骤四：更新 VS Code 配置

### 4.1 更新 c_cpp_properties.json

如果 VS Code 显示错误，检查 `.vscode/c_cpp_properties.json` 中的 `includePath` 是否包含：

```json
"${workspaceFolder}/Drivers/STM32F1xx_HAL_Driver/Inc"
```

---

## 🧮 步骤五：PWM 参数计算

### 计算说明

假设系统时钟为 **72MHz**：

1. **定时器时钟**：
   - APB1 时钟 = 36MHz（72MHz / 2）
   - TIM2 在 APB1 上，如果 APB1 预分频器不为1，定时器时钟 = APB1 × 2 = 72MHz
   - 设置预分频器 PSC = 71 → 定时器时钟 = 72MHz / (71+1) = **1MHz**

2. **PWM 频率**：
   - 自动重装载值 ARR = 19999
   - PWM 频率 = 1MHz / (19999+1) = **50Hz**（周期 20ms）✓

3. **脉宽计算**：
   - 0.5ms = 500 个计数值
   - 1.5ms = 1500 个计数值
   - 2.5ms = 2500 个计数值

### 公式

```
计数值 = (目标脉宽(ms) / 周期(ms)) × (ARR + 1)
计数值 = (目标脉宽(ms) / 20) × 20000
```

---

## 📊 完整代码示例

### main.c 完整示例

```c
/* USER CODE BEGIN Includes */
#include <stdio.h>
/* USER CODE END Includes */

/* USER CODE BEGIN PV */
// 舵机角度变量
uint8_t servo_angle = 90;
/* USER CODE END PV */

/* USER CODE BEGIN 2 */
// 启动 TIM2 的 PWM 输出
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);

// 初始位置：90度
Servo_SetAngle(90);
HAL_Delay(1000);
/* USER CODE END 2 */

/* USER CODE BEGIN WHILE */
while (1)
{
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
    // 示例：舵机在 0° 和 180° 之间摆动
    for (uint8_t i = 0; i <= 180; i += 10)
    {
        Servo_SetAngle(i);
        HAL_Delay(500);
    }
    
    for (uint8_t i = 180; i >= 0; i -= 10)
    {
        Servo_SetAngle(i);
        HAL_Delay(500);
    }
}
/* USER CODE END 3 */

/* USER CODE BEGIN 4 */
void Servo_SetAngle(uint8_t angle)
{
    if (angle > 180) angle = 180;
    
    // 计算占空比对应的计数值
    // 脉宽范围：0.5ms - 2.5ms
    // 对应计数值：500 - 2500
    uint16_t pulse = 500 + (angle * 2000) / 180;
    
    // 设置 PWM 占空比
    __HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, pulse);
}
/* USER CODE END 4 */
```

---

## ✅ 步骤六：编译和测试

### 6.1 编译项目

```bash
make clean
make -j4
```

或使用 VS Code：
- `Ctrl+Shift+B` 或
- `Ctrl+P` → 输入 `task build`

### 6.2 烧录和调试

1. 连接 ST-Link 到开发板
2. 按 `F5` 启动调试
3. 观察舵机是否按预期运动

---

## 🔧 故障排除

### 问题1：编译错误 - 找不到 htim2

**解决**：
- 确保在 STM32CubeMX 中正确配置了 TIM2
- 重新生成代码
- 检查 `stm32f1xx_hal_msp.c` 中是否有 `MX_TIM2_Init()` 函数

### 问题2：舵机不转动

**检查清单**：
- [ ] 电源连接正确（舵机需要外部电源，通常5V）
- [ ] 信号线连接到 PA0
- [ ] 地线（GND）已连接
- [ ] PWM 已启动（`HAL_TIM_PWM_Start`）
- [ ] 定时器配置正确（50Hz，20ms周期）

### 问题3：舵机角度不准确

**调整**：
- 修改 `Servo_SetAngle` 函数中的脉宽范围
- 不同舵机的脉宽范围可能略有不同
- 可以尝试：0.5ms-2.5ms 或 1ms-2ms

### 问题4：舵机抖动

**解决**：
- 确保电源稳定（使用外部电源，不要用开发板的3.3V）
- 增加滤波电容
- 检查 PWM 信号是否稳定

---

## 📚 参考资源

- [STM32 HAL 库 TIM 文档](https://www.st.com/resource/en/user_manual/um1725-description-of-stm32f1-hal-and-lowlayer-drivers-stmicroelectronics.pdf)
- [STM32F103 数据手册](https://www.st.com/resource/en/datasheet/stm32f103c8.pdf)
- [STM32F103 参考手册](https://www.st.com/resource/en/reference_manual/rm0008-stm32f101xx-stm32f102xx-stm32f103xx-stm32f105xx-and-stm32f107xx-advanced-arm-based-32-bit-mcus-stmicroelectronics.pdf)

---

## 🎯 快速检查清单

- [ ] STM32CubeMX 中配置 TIM2_CH1 为 PWM
- [ ] 设置 PWM 频率为 50Hz（周期 20ms）
- [ ] 生成代码并保留用户代码
- [ ] 更新 Makefile 添加 TIM 源文件
- [ ] 在 main.c 中启动 PWM
- [ ] 实现 `Servo_SetAngle` 函数
- [ ] 编译成功
- [ ] 硬件连接正确
- [ ] 测试舵机运动

---

**祝你成功！** 🎉

