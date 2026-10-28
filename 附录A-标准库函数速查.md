# 附录 A 标准库函数速查

> **本附录收录本教程正文（第 2–6 章）中实际出现过的全部标准库函数。**
>
> 不是完整的标准库手册——ST 标准外设库 V3.5.0 有上百个函数，其中绝大多数本教程用不到。**这里只列你见过、或者马上要用到的那些。**
>
> 版本：**STM32F10x Standard Peripheral Library V3.5.0**。函数原型均取自该版本源码。

## A.1 怎么用这个附录

**三种用法：**

| 你想干什么 | 翻到 |
|---|---|
| 想起一个函数的参数顺序，但记不清了 | 下面的分组速查表，或 **A.11 字母索引** |
| 知道要做什么，但不知道用哪个函数 | **A.10 按任务查找** |
| 想看某个函数的详细讲解 | 表里的「出处」列，翻回正文对应章节 |

**阅读约定：**

- `GPIOx` 代表 `GPIOA` ~ `GPIOD`，`ADCx` 代表 `ADC1`/`ADC2`，`USARTx` 代表 `USART1`~`USART3`——**函数原型里的 x 是占位，调用时要写具体的外设名。**
- 「出处」列里的 `4.3.3` 表示第 4 章 4.3.3 节。

**两个最重要的类型别名**（几乎所有函数都会用到）：

| 类型 | 取值 | 出处 |
|---|---|---|
| `FunctionalState` | `ENABLE` / `DISABLE` | —— |
| `FlagStatus` | `SET` / `RESET` | —— |
| `ITStatus` | `SET` / `RESET` | —— |
| `BitAction` | `Bit_SET` / `Bit_RESET` | —— |

> ⚠️ **`SET`/`RESET` 和 `Bit_SET`/`Bit_RESET` 是两套不同的东西**，别混用：
>
> - `FlagStatus` / `ITStatus` 用 **`SET` / `RESET`**（判断状态标志、中断标志）
> - `BitAction` 用 **`Bit_SET` / `Bit_RESET`**（读写引脚电平，见 A.3.3）
>
> 混用通常只是编译警告，但会让你困惑。**判断标志用 SET，写电平用 Bit_SET。**

---

## A.2 时钟与外设使能（RCC）

**每个外设的第一行代码，都在这里。** 第 4 章 4.1.2 的口诀：**凡是操作某个外设，第一行先想它挂在哪条总线上，先开时钟。**

### A.2.1 开时钟

| 函数 | 说明 | 出处 |
|---|---|---|
| `void RCC_APB2PeriphClockCmd(uint32_t RCC_APB2Periph, FunctionalState NewState)` | 开 **APB2**（72 MHz）上的外设时钟 | 4.3.1 |
| `void RCC_APB1PeriphClockCmd(uint32_t RCC_APB1Periph, FunctionalState NewState)` | 开 **APB1**（36 MHz）上的外设时钟 | **6.4.1** ⚠️ |
| `void RCC_AHBPeriphClockCmd(uint32_t RCC_AHBPeriph, FunctionalState NewState)` | 开 **AHB** 上的外设时钟（**只用于 DMA**） | 5.9.2 |

**哪个外设用哪个函数——本教程涉及的部分：**

| 总线 | 外设 | 用哪个函数 |
|---|---|---|
| **APB2** | `GPIOA`/`GPIOB`/`GPIOC`/`GPIOD`、`ADC1`、`ADC2`、`USART1`、`AFIO` | `RCC_APB2PeriphClockCmd()` |
| **APB1** | **`DAC`**、`TIM2`~`TIM7`、`USART2`/`USART3` | `RCC_APB1PeriphClockCmd()` |
| **AHB** | **`DMA1`**、`DMA2` | `RCC_AHBPeriphClockCmd()` |

**常用写法**（多个外设可以合并，只要在同一条总线上）：

```c
/* 一个外设 */
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

/* 同一条总线上的两个外设，用 | 合并 */
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_GPIOB, ENABLE);
RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 | RCC_APB2Periph_GPIOB, ENABLE);

/* 不同总线，必须分开写 */
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);      // GPIO 在 APB2
RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);        // DAC  在 APB1
```

> ⚠️ **★ 全教程唯一的 APB1 就是 DAC。** 前面所有外设都在 APB2，写到第 6 章极易顺手打成 `APB2`。
>
> **后果**：程序能编译、能下载，但**引脚上一点电压都没有**。因为你根本没开 DAC 的时钟（`RCC_APB2Periph_DAC` 这个宏不存在，写成 APB2 会编译失败——真正危险的是**忘了写这一行**）。详见 6.4.1。

### A.2.2 ADC 专用时钟

| 函数 | 说明 | 出处 |
|---|---|---|
| `void RCC_ADCCLKConfig(uint32_t RCC_PCLK2)` | 设置 ADC 的时钟分频 | 5.4.4 |

**参数**（**ADC 时钟不能超过 14 MHz**）：

| 取值 | 结果（PCLK2 = 72 MHz） | 是否可用 |
|---|---|---|
| `RCC_PCLK2_Div2` | 36 MHz | ❌ 超限 |
| `RCC_PCLK2_Div4` | 18 MHz | ❌ 超限 |
| **`RCC_PCLK2_Div6`** | **12 MHz** | ✅ **本教程用这个** |
| `RCC_PCLK2_Div8` | 9 MHz | ✅ 可用但慢 |

> **注意它不叫 `RCC_ADC...` 而叫 `RCC_PCLK2...`**——因为它改的是"APB2 分频后送给 ADC 的时钟"，不是 ADC 自己的配置。
>
> **位置有讲究**：在 `ADC_Init()` **之前**调用。

---

## A.3 GPIO（第 4 章）

### A.3.1 初始化与配置

| 函数 | 说明 | 出处 |
|---|---|---|
| `void GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct)` | **配置引脚**（最常用） | 4.3.2 |
| `void GPIO_StructInit(GPIO_InitTypeDef* GPIO_InitStruct)` | 把结构体填成默认值 | 4.3.5 |
| `void GPIO_DeInit(GPIO_TypeDef* GPIOx)` | 把整个端口恢复成复位默认值 | 4.3.5 |
| `void GPIO_PinLockConfig(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)` | 锁定引脚配置，禁止此后修改 | 4.3.5 |
| `void GPIO_PinRemapConfig(uint32_t GPIO_Remap, FunctionalState NewState)` | 引脚重映射（本教程实验不涉及） | 4.3.5 |

**`GPIO_InitTypeDef` 的三个成员：**

| 成员 | 取值 |
|---|---|
| `GPIO_Pin` | `GPIO_Pin_0` ~ `GPIO_Pin_15`，可用 `\|` 合并 |
| `GPIO_Mode` | 八种模式之一（见 4.2.1 表） |
| `GPIO_Speed` | `GPIO_Speed_10MHz` / `_2MHz` / `_50MHz`，**只在输出模式下有效** |

**固定套路**：

```c
GPIO_InitTypeDef GPIO_InitStructure;

GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_3;
GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_Out_PP;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOA, &GPIO_InitStructure);      // 别忘了 &
```

### A.3.2 输出：写引脚

| 函数 | 说明 | 实际做什么 | 出处 |
|---|---|---|---|
| `void GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)` | 输出**高** | 写 `BSRR` | 4.3.3 |
| `void GPIO_ResetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)` | 输出**低** | 写 `BRR` | 4.3.3 |
| `void GPIO_WriteBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, BitAction BitVal)` | 按变量值输出 | 内部判断 | 4.3.3 |
| `void GPIO_Write(GPIO_TypeDef* GPIOx, uint16_t PortVal)` | 一次写一整个端口 | 直接写 `ODR` | 4.3.3 |

```c
GPIO_SetBits(GPIOA, GPIO_Pin_3);                              // PA3 → 高
GPIO_ResetBits(GPIOA, GPIO_Pin_3);                            // PA3 → 低
GPIO_ResetBits(GPIOA, GPIO_Pin_3 | GPIO_Pin_4 | GPIO_Pin_5);  // 三脚同时拉低
```

> **★ 优先用 `SetBits`/`ResetBits`，别直接改 `ODR`。** 它们走的是 `BSRR`/`BRR`，是**原子操作**——一条指令完成，不会被打断。写成 `GPIOA->ODR |= GPIO_Pin_3` 是"读-改-写"三条指令，中断里改同一个寄存器就会被覆盖。详见 4.3.3。

### A.3.3 输入：读引脚

| 函数 | 读的是 | 返回值 | 用途 | 出处 |
|---|---|---|---|---|
| `uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)` | **IDR** | `0`/`1` | **读按键、读外部信号** ✅ | 4.3.4 |
| `uint16_t GPIO_ReadInputData(GPIO_TypeDef* GPIOx)` | IDR | 16 位 | 一次读整个端口 | 4.3.4 |
| `uint8_t GPIO_ReadOutputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)` | **ODR** | `0`/`1` | 确认自己写了什么 | 4.3.4 |
| `uint16_t GPIO_ReadOutputData(GPIO_TypeDef* GPIOx)` | ODR | 16 位 | 同上 | 4.3.4 |

```c
if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_11) == Bit_RESET)   // KEY2 按下（低电平）
{
    /* ... */
}
```

> ⚠️ **★ 本教程重复最多的一个坑：读按键必须用 `ReadInputDataBit`。**
>
> 写成 `ReadOutputDataBit` **不会报错**，但读到的永远是你自己上一次写出去的值——**现象是"按键完全没反应"**。
>
> **口诀：读外面的用 `ReadInput`，看自己的用 `ReadOutput`。**（4.1.4 / 4.3.4 / 4.6⑥）

---

## A.4 USART 串口（第 4 章 4.9.2）

| 函数 | 说明 | 出处 |
|---|---|---|
| `void USART_Init(USART_TypeDef* USARTx, USART_InitTypeDef* USART_InitStruct)` | 配置串口参数 | 4.9.2 |
| `void USART_Cmd(USART_TypeDef* USARTx, FunctionalState NewState)` | **使能串口**（不写不工作） | 4.9.2 |
| `void USART_SendData(USART_TypeDef* USARTx, uint16_t Data)` | 发一个字节 | 4.9.2 |
| `uint16_t USART_ReceiveData(USART_TypeDef* USARTx)` | 收一个字节 | —— |
| `FlagStatus USART_GetFlagStatus(USART_TypeDef* USARTx, uint16_t USART_FLAG)` | 查状态标志 | 4.9.2 |

**`USART_InitTypeDef` 的六个成员**（本教程的设置）：

| 成员 | 本教程取值 |
|---|---|
| `USART_BaudRate` | `9600`（要和串口助手一致） |
| `USART_WordLength` | `USART_WordLength_8b` |
| `USART_StopBits` | `USART_StopBits_1` |
| `USART_Parity` | `USART_Parity_No` |
| `USART_HardwareFlowControl` | `USART_HardwareFlowControl_None` |
| `USART_Mode` | `USART_Mode_Tx \| USART_Mode_Rx` |

**常用标志：**

| 标志 | 含义 |
|---|---|
| `USART_FLAG_TXE` | 发送数据寄存器空（可以放下一个字节了） |
| `USART_FLAG_RXNE` | 接收数据寄存器非空（收到东西了） |
| `USART_FLAG_TC` | 发送**完成**（整个帧发完了） |

**`printf` 重定向**（第 4 章 4.9.2，第 5、6 章一直在用）：

```c
int fputc(int ch, FILE *f)
{
    USART_SendData(USART1, (uint8_t)ch);
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    return ch;
}
```

**三个配套前提**（缺一不可，详见 4.9.2）：

1. 必须 `#include <stdio.h>`
2. Keil 里勾选 **`Use MicroLIB`**（Target 选项卡）——否则程序会卡死
3. **PA9 配成 `GPIO_Mode_AF_PP`**（复用推挽），不是 `Out_PP`

---

## A.5 ADC（第 5 章）

### A.5.1 初始化

| 函数 | 说明 | 出处 |
|---|---|---|
| `void ADC_Init(ADC_TypeDef* ADCx, ADC_InitTypeDef* ADC_InitStruct)` | 配置 ADC | 5.4.6 |
| `void ADC_StructInit(ADC_InitTypeDef* ADC_InitStruct)` | 填默认值 | —— |
| `void ADC_Cmd(ADC_TypeDef* ADCx, FunctionalState NewState)` | **使能 ADC** | 5.4.6 |
| `void ADC_DeInit(ADC_TypeDef* ADCx)` | 恢复默认 | —— |
| `void ADC_RegularChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel, uint8_t Rank, uint8_t ADC_SampleTime)` | **指定通道、序列位置、采样时间** | 5.4.6 |

**`ADC_InitTypeDef` 的六个成员**（本教程单通道的设置）：

| 成员 | 本教程取值 | 说明 |
|---|---|---|
| `ADC_Mode` | `ADC_Mode_Independent` | 只用 ADC1 |
| `ADC_ScanConvMode` | `DISABLE` / `ENABLE` | 扫描（多通道时开） |
| `ADC_ContinuousConvMode` | `DISABLE` / `ENABLE` | 连续（5.9.2 开） |
| `ADC_ExternalTrigConv` | `ADC_ExternalTrigConv_None` | 软件触发 |
| `ADC_DataAlign` | `ADC_DataAlign_Right` | 右对齐 |
| `ADC_NbrOfChannel` | `1`（或多通道时填通道数） | 规则组通道数 |

**`ADC_RegularChannelConfig` 的四个参数：**

| 位置 | 填什么 | 例 |
|---|---|---|
| 1 | ADC 外设 | `ADC1` |
| 2 | 通道 | `ADC_Channel_7` |
| 3 | 在规则组里排第几（**从 1 开始**） | `1` |
| 4 | 采样时间 | `ADC_SampleTime_55Cycles5` |

**常用通道与引脚**（本板）：

| 引脚 | 板载网络名 | 通道常量 |
|---|---|---|
| PA7 | 板载 ADC1（热敏） | `ADC_Channel_7` |
| PB0 | 板载 ADC2（光敏） | `ADC_Channel_8` |
| PB1 | 板载 ADC3（电位器） | `ADC_Channel_9` |

### A.5.2 校准

| 函数 | 说明 | 出处 |
|---|---|---|
| `void ADC_ResetCalibration(ADC_TypeDef* ADCx)` | 复位校准寄存器 | 5.4.5 |
| `FlagStatus ADC_GetResetCalibrationStatus(ADC_TypeDef* ADCx)` | 查复位是否完成 | 5.4.5 |
| `void ADC_StartCalibration(ADC_TypeDef* ADCx)` | 启动校准 | 5.4.5 |
| `FlagStatus ADC_GetCalibrationStatus(ADC_TypeDef* ADCx)` | 查校准是否完成 | 5.4.5 |

**固定顺序**（**两个 `while` 不能省**）：

```c
ADC_Cmd(ADC1, ENABLE);                                  // ① 先使能
ADC_ResetCalibration(ADC1);
while (ADC_GetResetCalibrationStatus(ADC1) == SET);     // ② 等复位完成
ADC_StartCalibration(ADC1);
while (ADC_GetCalibrationStatus(ADC1) == SET);          // ③ 等校准完成
```

### A.5.3 触发与读值

| 函数 | 说明 | 出处 |
|---|---|---|
| `void ADC_SoftwareStartConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState)` | 软件触发一次转换 | 5.4.6 |
| `FlagStatus ADC_GetFlagStatus(ADC_TypeDef* ADCx, uint8_t ADC_FLAG)` | 查标志（主要用 `ADC_FLAG_EOC`） | 5.4.6 |
| `uint16_t ADC_GetConversionValue(ADC_TypeDef* ADCx)` | **读转换结果**（读 DR 会**自动清 EOC**） | 5.4.6 |

**标准读值函数**：

```c
uint16_t AD_GetValue(void)
{
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
    while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);
    return ADC_GetConversionValue(ADC1);
}
```

### A.5.4 中断与 DMA

| 函数 | 说明 | 出处 |
|---|---|---|
| `void ADC_ITConfig(ADC_TypeDef* ADCx, uint16_t ADC_IT, FunctionalState NewState)` | 开中断（`ADC_IT_EOC`） | 5.9.1 |
| `ITStatus ADC_GetITStatus(ADC_TypeDef* ADCx, uint16_t ADC_IT)` | 查中断标志（**中断里用**） | 5.9.1 |
| `void ADC_ClearITPendingBit(ADC_TypeDef* ADCx, uint16_t ADC_IT)` | **清中断挂起位** | 5.9.1 |
| `void ADC_DMACmd(ADC_TypeDef* ADCx, FunctionalState NewState)` | 让 ADC 产生 DMA 请求 | 5.9.2 |

**DMA 通道映射**（硬件定死，改不了）：

| 外设 | DMA 通道 |
|---|---|
| **ADC1** | **DMA1 通道 1** |
| DAC 通道 1 | DMA1 通道 3 |
| DAC 通道 2 | DMA1 通道 4 |

---

## A.6 DAC（第 6 章）

| 函数 | 说明 | 出处 |
|---|---|---|
| `void DAC_Init(uint32_t DAC_Channel, DAC_InitTypeDef* DAC_InitStruct)` | 配置 DAC 通道 | 6.4.3 |
| `void DAC_Cmd(uint32_t DAC_Channel, FunctionalState NewState)` | **使能通道** | 6.4.4 |
| `void DAC_SetChannel1Data(uint32_t DAC_Align, uint16_t Data)` | 给**通道 1（PA4）**写数据 | 6.4.4 |
| `void DAC_SetChannel2Data(uint32_t DAC_Align, uint16_t Data)` | 给**通道 2（PA5）**写数据 | 6.4.4 |
| `void DAC_SoftwareTriggerCmd(uint32_t DAC_Channel, FunctionalState NewState)` | 软件触发（仅在 `DAC_Trigger_Software` 时需要） | 6.4.4 |
| `void DAC_DMACmd(uint32_t DAC_Channel, FunctionalState NewState)` | 让 DAC 产生 DMA 请求 | 6.7 |

**`DAC_InitTypeDef` 的四个成员**：

| 成员 | 本教程取值 | 说明 |
|---|---|---|
| `DAC_Trigger` | `DAC_Trigger_None` | 写完即输出 |
| | `DAC_Trigger_T6_TRGO` | **由 TIM6 触发**（6.7 输出波形用） |
| `DAC_WaveGeneration` | `DAC_WaveGeneration_None` | 不用片内波形发生器 |
| `DAC_LFSRUnmask_TriangleAmplitude` | `DAC_LFSRUnmask_Bit0` | 用不到，但**必须填** |
| `DAC_OutputBuffer` | `DAC_OutputBuffer_Enable` | 开输出缓冲 |

> ⚠️ **`DAC_SetChannel1Data` / `SetChannel2Data` 里的数字是通道号，不是外设编号。**
>
> - PA4 = 通道 1 → `DAC_SetChannel1Data()`
> - PA5 = 通道 2 → `DAC_SetChannel2Data()`
>
> 写错不会报错，只是输出到错误的引脚。

**关于 `DAC_Trigger_None` 要不要配软件触发**：

| 触发设置 | 还需要什么 |
|---|---|
| `DAC_Trigger_None` | **不需要**，`DAC_SetChannel1Data()` 写完就输出 ✅ 本教程 6.5/6.6 用 |
| `DAC_Trigger_Software` | 需要额外调 `DAC_SoftwareTriggerCmd()` |
| `DAC_Trigger_T6_TRGO` | 需要配 TIM6 |

---

## A.7 DMA（第 5、6 章）

| 函数 | 说明 | 出处 |
|---|---|---|
| `void DMA_DeInit(DMA_Channel_TypeDef* DMAy_Channelx)` | 复位通道（配置前先调） | 5.9.2 |
| `void DMA_Init(DMA_Channel_TypeDef* DMAy_Channelx, DMA_InitTypeDef* DMA_InitStruct)` | 配置 DMA 通道 | 5.9.2 |
| `void DMA_Cmd(DMA_Channel_TypeDef* DMAy_Channelx, FunctionalState NewState)` | 使能 DMA 通道 | 5.9.2 |

**`DMA_InitTypeDef` 的成员**（★ 标出 ADC 和 DAC 的区别）：

| 成员 | ADC 采集（5.9.2） | DAC 输出波形（6.7） |
|---|---|---|
| `DMA_PeripheralBaseAddr` | `(uint32_t)&ADC1->DR` | `(uint32_t)&DAC->DHR12R1` |
| `DMA_MemoryBaseAddr` | `(uint32_t)数组` | `(uint32_t)数组` |
| **`DMA_DIR`** ★ | **`DMA_DIR_PeripheralSRC`**（外设是源） | **`DMA_DIR_PeripheralDST`**（外设是目的） |
| `DMA_BufferSize` | 通道数 | 波形点数 |
| `DMA_PeripheralInc` | `Disable`（外设地址不变） | `Disable` |
| `DMA_MemoryInc` | `Enable`（内存地址递增） | `Enable` |
| `DMA_PeripheralDataSize` | `HalfWord`（16 位） | `HalfWord` |
| `DMA_MemoryDataSize` | `HalfWord` | `HalfWord` |
| `DMA_Mode` | `Circular`（循环） | `Circular` |
| `DMA_Priority` | `Medium` | `High` |
| `DMA_M2M` | `Disable` | `Disable` |

> ⚠️ **★ `DMA_DIR` 最容易搞反。** 记法：**看箭头方向**。
>
> - `PeripheralSRC` = 外设是**源** → 数据"**从**外设出来" → **ADC 采集**
> - `PeripheralDST` = 外设是**目的** → 数据"**去**外设" → **DAC 输出**

---

## A.8 TIM 定时器（第 6 章 6.7）

**本教程只用 TIM6 做一件事：给 DAC 提供触发脉冲。** 这里只列用到的那几个函数。

| 函数 | 说明 | 出处 |
|---|---|---|
| `void TIM_TimeBaseInit(TIM_TypeDef* TIMx, TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct)` | 配置定时器 | 6.7 |
| `void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState)` | 启动定时器 | 6.7 |
| `void TIM_SelectOutputTrigger(TIM_TypeDef* TIMx, uint16_t TIM_TRGOSource)` | **选择 TRGO 触发源** | 6.7 |

**`TIM_TimeBaseInitTypeDef` 的常用成员**：

| 成员 | 说明 |
|---|---|
| `TIM_Prescaler` | 预分频（填 `N-1` 表示除以 N） |
| `TIM_Period` | 自动重装值（填 `N-1` 表示每 N 个计数溢出一次） |
| `TIM_ClockDivision` | 填 `TIM_CKD_DIV1` |
| `TIM_CounterMode` | 填 `TIM_CounterMode_Up` |
| `TIM_RepetitionCounter` | 写 `0`（高级定时器才用） |

**频率计算公式：**

```
   计数频率 = 72 MHz ÷ (TIM_Prescaler + 1)
   溢出频率 = 计数频率 ÷ (TIM_Period + 1)
```

**本教程 6.7 的参数**：`Prescaler = 72-1`（得 1 MHz）、`Period = 7812-1`（得 128 Hz 触发）。

**TRGO 触发的写法**：

```c
TIM_SelectOutputTrigger(TIM6, TIM_TRGOSource_Update);   // 每次更新事件输出一个脉冲
```

---

## A.9 NVIC 中断控制（第 5 章 5.9.1）

| 函数 | 说明 | 出处 |
|---|---|---|
| `void NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup)` | 设置优先级分组（整个工程调一次） | 5.9.1 |
| `void NVIC_Init(NVIC_InitTypeDef* NVIC_InitStruct)` | 配置一个中断通道 | 5.9.1 |

**`NVIC_InitTypeDef` 的四个成员**：

| 成员 | 说明 |
|---|---|
| `NVIC_IRQChannel` | 中断号，如 `ADC1_2_IRQn` |
| `NVIC_IRQChannelPreemptionPriority` | **抢占优先级**（能打断别人） |
| `NVIC_IRQChannelSubPriority` | **响应优先级**（不能打断，只决定排队） |
| `NVIC_IRQChannelCmd` | `ENABLE` |

**本教程用过的中断号**：`ADC1_2_IRQn`（ADC1 和 ADC2 共用）。

> ⚠️ **中断服务函数的名字是固定的**，定义在启动文件 `startup_stm32f10x_md.s` 里。ADC1/ADC2 的中断函数必须叫 **`ADC1_2_IRQHandler`**，写错了会静默失效。详见 5.9.1。

---

## A.10 按任务查找

**"我想做 X，用哪个函数？"**

| 我想…… | 用什么 | 出处 |
|---|---|---|
| 让某个外设开始工作 | 先 `RCC_xxxPeriphClockCmd()` | 4.1.2 |
| 配置一个引脚 | `GPIO_Init()` | 4.3.2 |
| 让引脚输出高/低 | `GPIO_SetBits()` / `GPIO_ResetBits()` | 4.3.3 |
| 读一个按键/外部信号 | `GPIO_ReadInputDataBit()` | 4.3.4 |
| 用串口打印调试信息 | `USART_Init()` + `fputc()` 重定向 + MicroLIB | 4.9.2 |
| 采集一路模拟电压 | `ADC_Init()` + `ADC_RegularChannelConfig()` + `ADC_SoftwareStartConvCmd()` | 5.4.6 |
| 同时采集多路模拟量 | ADC 开扫描+连续 + `ADC_DMACmd()` + DMA | 5.9.2 |
| 让 ADC 转换完自动通知 CPU | `ADC_ITConfig()` + `NVIC_Init()` + 写 `ADC1_2_IRQHandler` | 5.9.1 |
| 输出一个指定电压 | `DAC_Init()` + `DAC_Cmd()` + `DAC_SetChannel1Data()` | 6.4.4 |
| 输出连续变化的波形 | TIM6 + DMA + `DAC_Trigger_T6_TRGO` + `DAC_DMACmd()` | 6.7 |
| 让 DMA 帮我搬数据 | `DMA_Init()` + `DMA_Cmd()` | 5.9.2 |
| 精确定时 / 定期触发 | `TIM_TimeBaseInit()` + `TIM_SelectOutputTrigger()` | 6.7 |

---

## A.11 按字母索引

| 函数 | 分组 | 出处 |
|---|---|---|
| `ADC_ClearITPendingBit` | ADC | 5.9.1 |
| `ADC_Cmd` | ADC | 5.4.6 |
| `ADC_DeInit` | ADC | —— |
| `ADC_DMACmd` | ADC | 5.9.2 |
| `ADC_GetCalibrationStatus` | ADC | 5.4.5 |
| `ADC_GetConversionValue` | ADC | 5.4.6 |
| `ADC_GetFlagStatus` | ADC | 5.4.6 |
| `ADC_GetITStatus` | ADC | 5.9.1 |
| `ADC_GetResetCalibrationStatus` | ADC | 5.4.5 |
| `ADC_ITConfig` | ADC | 5.9.1 |
| `ADC_Init` | ADC | 5.4.6 |
| `ADC_RegularChannelConfig` | ADC | 5.4.6 |
| `ADC_ResetCalibration` | ADC | 5.4.5 |
| `ADC_SoftwareStartConvCmd` | ADC | 5.4.6 |
| `ADC_StartCalibration` | ADC | 5.4.5 |
| `ADC_StructInit` | ADC | —— |
| `DAC_Cmd` | DAC | 6.4.4 |
| `DAC_DMACmd` | DAC | 6.7 |
| `DAC_Init` | DAC | 6.4.3 |
| `DAC_SetChannel1Data` | DAC | 6.4.4 |
| `DAC_SetChannel2Data` | DAC | 6.4.4 |
| `DAC_SoftwareTriggerCmd` | DAC | 6.4.4 |
| `DMA_Cmd` | DMA | 5.9.2 |
| `DMA_DeInit` | DMA | 5.9.2 |
| `DMA_Init` | DMA | 5.9.2 |
| `GPIO_DeInit` | GPIO | 4.3.5 |
| `GPIO_Init` | GPIO | 4.3.2 |
| `GPIO_PinLockConfig` | GPIO | 4.3.5 |
| `GPIO_PinRemapConfig` | GPIO | 4.3.5 |
| `GPIO_ReadInputData` | GPIO | 4.3.4 |
| `GPIO_ReadInputDataBit` | GPIO | 4.3.4 |
| `GPIO_ReadOutputData` | GPIO | 4.3.4 |
| `GPIO_ReadOutputDataBit` | GPIO | 4.3.4 |
| `GPIO_ResetBits` | GPIO | 4.3.3 |
| `GPIO_SetBits` | GPIO | 4.3.3 |
| `GPIO_StructInit` | GPIO | 4.3.5 |
| `GPIO_Write` | GPIO | 4.3.3 |
| `GPIO_WriteBit` | GPIO | 4.3.3 |
| `NVIC_Init` | NVIC | 5.9.1 |
| `NVIC_PriorityGroupConfig` | NVIC | 5.9.1 |
| `RCC_ADCCLKConfig` | RCC | 5.4.4 |
| `RCC_AHBPeriphClockCmd` | RCC | 5.9.2 |
| `RCC_APB1PeriphClockCmd` | RCC | **6.4.1** |
| `RCC_APB2PeriphClockCmd` | RCC | 4.3.1 |
| `TIM_Cmd` | TIM | 6.7 |
| `TIM_SelectOutputTrigger` | TIM | 6.7 |
| `TIM_TimeBaseInit` | TIM | 6.7 |
| `USART_Cmd` | USART | 4.9.2 |
| `USART_GetFlagStatus` | USART | 4.9.2 |
| `USART_Init` | USART | 4.9.2 |
| `USART_ReceiveData` | USART | —— |
| `USART_SendData` | USART | 4.9.2 |

---

## A.12 GPIO 八种模式速查

**这是全书最需要常回头查的一张表**（第 4 章 4.2.1 的浓缩版）。

| 模式 | 枚举值 | 用途 | 本板例子 |
|---|---|---|---|
| 浮空输入 | `GPIO_Mode_IN_FLOATING` | 外部已有上下拉的输入 | —— |
| 上拉输入 | `GPIO_Mode_IPU` | 外部经开关接 GND | **按键 PA11/PA12/PB5**、串口 RX PA10 |
| 下拉输入 | `GPIO_Mode_IPD` | 外部经开关接 VCC | —— |
| 模拟输入 | `GPIO_Mode_AIN` | **ADC 采集 / DAC 输出** | **PA7/PB0/PB1（ADC）**、**PA4/PA5（DAC）** |
| 推挽输出 | `GPIO_Mode_Out_PP` | 主动驱动负载 | **LED PA3/PA4/PA5、蜂鸣器 PA8** |
| 开漏输出 | `GPIO_Mode_Out_OD` | 电平转换、线与 | —— |
| 复用推挽 | `GPIO_Mode_AF_PP` | 引脚交给外设（推挽） | **串口 TX PA9** |
| 复用开漏 | `GPIO_Mode_AF_OD` | 引脚交给外设（开漏） | **硬件 I²C PB10/PB11** |

**逐个复制的模板：**

```c
/* 推挽输出（LED、蜂鸣器） */
GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_x;
GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_Out_PP;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOx, &GPIO_InitStructure);

/* 上拉输入（按键） */
GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_x;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
GPIO_Init(GPIOx, &GPIO_InitStructure);

/* 模拟输入（ADC / DAC，速度那一行可省略） */
GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_x;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
GPIO_Init(GPIOx, &GPIO_InitStructure);

/* 复用推挽（串口 TX） */
GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_x;
GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_AF_PP;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOx, &GPIO_InitStructure);
```

---

[返回目录](README.md) · [附录B 常见问题 FAQ](附录B-常见问题FAQ.md) · [附录C 板载引脚总表](附录C-板载引脚总表.md)
