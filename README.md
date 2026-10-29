# 电控组第二次培训作业

- 姓名：**丁澳**
- 学号：***20263001694***

## 一、任务完成情况

| 任务 | 要求 | 完成 |
|---|---|---|
| 基础任务 | CubeMX 配置三个 LED 引脚并使用 User Label 命名为 LED0 / LED1 / LED2，CLion 编写 500 ms 三灯流水灯 | ✅ |
| 加分项 | 摆脱 `HAL_Delay()` 死等，改用非阻塞延时 | ✅ `HAL_GetTick()` 时间戳 + 状态机 |
| 文档 | README.md + 验收视频 | ✅ 见仓库 |

## 二、硬件连接

芯片：STM32F103C8T6（Blue Pill 最小系统板），主频 72 MHz（HSE 8 MHz × 9）。

| LED | 引脚 | 接法 |
|---|---|---|
| LED0 | PA0 | PA0 ── 470 Ω ──▶\|── GND |
| LED1 | PA1 | PA1 ── 470 Ω ──▶\|── GND |
| LED2 | PA2 | PA2 ── 470 Ω ──▶\|── GND |

三颗灯均为**高电平点亮**（引脚输出 1 时点亮），限流电阻 470 Ω。

## 三、CubeMX 配置

1. 选择芯片 `STM32F103C8Tx`，RCC 中 HSE 选择 Crystal/Ceramic Resonator，时钟树配置为 HSE 8 MHz × PLL9 = 72 MHz；
2. Pinout 视图中把 **PA0、PA1、PA2** 设为 `GPIO_Output`；
3. 在 `System Core → GPIO` 中把三个脚的 **User Label 分别改为 `LED0`、`LED1`、`LED2`**，参数为：
   - GPIO output level：Low
   - GPIO mode：Output Push Pull
   - GPIO Pull-up/Pull-down：No pull-up and no pull-down
   - Maximum output speed：Low
4. 生成代码后，CubeMX 会把这些标签生成成 `main.h` 里的宏定义：

```c
/* Private defines -----------------------------------------------------------*/
#define LED0_Pin GPIO_PIN_0
#define LED0_GPIO_Port GPIOA
#define LED1_Pin GPIO_PIN_1
#define LED1_GPIO_Port GPIOA
#define LED2_Pin GPIO_PIN_2
#define LED2_GPIO_Port GPIOA
```

工程代码里所有引脚访问都直接使用上述宏，没有出现裸写的 `GPIO_PIN_x`。

## 四、实现思路

### 4.1 用表格描述三个灯的状态

把三颗灯组成一个数组，下标即当前点亮的灯：

```c
/* 一颗 LED 的端口 + 引脚 */
typedef struct {
  GPIO_TypeDef *port;
  uint16_t      pin;
} led_t;

/* LED0/LED1/LED2 的宏由 CubeMX 根据 User Label 生成在 main.h 里 */
static const led_t LEDS[] = {
  { LED0_GPIO_Port, LED0_Pin },
  { LED1_GPIO_Port, LED1_Pin },
  { LED2_GPIO_Port, LED2_Pin },
};
#define LED_COUNT (sizeof(LEDS) / sizeof(LEDS[0]))

static uint32_t led_index = 0;   /* 当前点亮的灯 */
static uint32_t led_tick  = 0;   /* 上次切换时的时间戳（ms） */
```

### 4.2 非阻塞延时（加分项）原理

**`HAL_Delay()` 的问题**：它内部就是一个 `while` 循环，反复读 SysTick 计数直到时间到，这 500 ms 里 CPU 被完全占住，不能做任何其它事情。

**本方案的做法**：SysTick 每 1 ms 中断一次，把全局变量 `uwTick` 加一；`HAL_GetTick()` 只是把这个变量读出来返回，本身不等待。于是主循环只需要比较"当前时间戳"和"上次切换的时间戳"：

```c
while (1)
{
  /* 非阻塞延时：只比较 SysTick 时间戳，不做死等 */
  if (HAL_GetTick() - led_tick >= LED_STEP_MS) {
    led_tick += LED_STEP_MS;   /* 固定步长累加，长时间运行也不会累积漂移 */

    /* 熄灭当前灯，点亮下一颗，循环往复 */
    HAL_GPIO_WritePin(LEDS[led_index].port, LEDS[led_index].pin, GPIO_PIN_RESET);
    led_index = (led_index + 1) % LED_COUNT;
    HAL_GPIO_WritePin(LEDS[led_index].port, LEDS[led_index].pin, GPIO_PIN_SET);
  }

  /* 这里可以放其它任务，不会被上面的延时阻塞 */
}
```

其中 `LED_STEP_MS` 定义为 `500u`。

三个关键点：

1. **不阻塞**：条件不满足时 `if` 直接跳过，主循环立刻进入下一轮，可以继续处理按键、串口等其它任务，CPU 不再空转等待。
2. **周期不漂移**：用 `led_tick += LED_STEP_MS` 而不是 `led_tick = HAL_GetTick()`。后者会把"判断那一刻的微小延迟"累加进周期，长时间运行后闪烁周期会越来越长；前者以固定步长推进，周期恒为 500 ms。
3. **溢出安全**：`HAL_GetTick() - led_tick` 是无符号 32 位减法，`uwTick` 在 49.7 天回绕时结果依然正确。

### 4.3 状态机

`led_index` 就是状态变量，取值 0 / 1 / 2，每 500 ms 执行一次状态转移：

```
0 → 1 → 2 → 0 → …
```

每次转移的动作是"熄灭旧的一颗、点亮新的一颗"，因此任意时刻只有一颗灯亮，形成流水效果。

## 五、编译与烧录

- IDE：CLion 2026.2.2
- 工具链：arm-none-eabi-gcc 10.3.1（CLion 自带）
- 构建系统：CMake + Ninja
- 烧录：OpenOCD 0.12.0 + ST-LINK V2

构建产物资源占用：

```
Memory region         Used Size  Region Size  %age Used
             RAM:        2664 B        20 KB     13.01%
           FLASH:        4940 B        64 KB      7.54%
```

## 六、目录结构

```
333/
├── 333.ioc                 # CubeMX 工程文件（含 LED0/LED1/LED2 标签）
├── CMakeLists.txt          # CLion 工程
├── STM32F103C8TX_FLASH.ld  # 链接脚本
├── Core/
│   ├── Inc/                # main.h 中有 LEDx_Pin / LEDx_GPIO_Port 宏
│   ├── Src/                # main.c（流水灯逻辑）、gpio.c（引脚初始化）
│   └── Startup/            # 启动文件
├── Drivers/                # HAL 库 + CMSIS
└── README.md
```

## 七、验收视频

`demo.mp4`：CLion 编译通过 → 启动烧录 → 开发板三灯 500 ms 间隔循环点亮。
