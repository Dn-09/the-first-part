# 第 5 章 ADC

第 4 章的结尾留了一句话：**GPIO 只有 0 和 1，它答不出"有多亮"。**

这一章就是来回答那个问题的。

**ADC（Analog-to-Digital Converter，模拟-数字转换器）** 把引脚上**连续变化的电压**变成一个**数字**。有了它，单片机才能真正"感知"这个世界——温度、光照、压力、音量、电池电量，全都变成了电压，然后全都变成了数字。

本章结构：

| 小节 | 内容 |
|---|---|
| 5.1 | ADC 是什么：逐次逼近原理、12 位、0~4095 |
| 5.2 | F103C8T6 的 ADC 资源与通道表 ★ 含命名冲突专节 |
| 5.3 | 四种转换模式 |
| 5.4 | 标准库 ADC 编程流程 |
| 5.5 ~ 5.7 | 实验 6 / 7 / 8：电位器 / 光敏 / 热敏（含温度换算） |
| 5.8 | 提高精度：采样时间 / 校准 / 多次平均 |
| 5.9 | 进阶：ADC + 中断 / ADC + DMA 多通道扫描 |
| 5.10 | 综合实战：简易电压表 |

---

## 5.1 ADC 是什么

### 5.1.1 接第 4 章：数字量的尽头

回顾第 4 章做过的两个输入实验：

| 实验 | 读到了什么 | 读不到什么 |
|---|---|---|
| 4.6 按键 | 按下 / 没按下 | 按多重 |
| 4.8 红外 | 挡住 / 没挡住 | 挡了多少 |

原因很直接：**GPIO 读回来的是一个位，只有 0 或 1 两种可能。** 引脚电压 0.1 V 和 1.6 V，在 GPIO 眼里都是"低"，因为它后面那个施密特触发器（4.1.3 讲过）**主动把电压整形掉了**。

而现实世界里的物理量几乎都是**连续的**：

```
温度     -40 ──────────────────────── 125 ℃
            ↑ 中间有无数个可能的值

GPIO     ──────────────────────────  ← 无法表示
ADC      0 ──────────────────── 4095  ← 可以
```

**ADC 就是把左边那根连续的线，切成 4096 段，告诉你"现在落在第几段"。**

### 5.1.2 逐次逼近型 ADC

![逐次逼近型 ADC 原理](images/05-adc/ADC_逐次逼近原理.jpg)

*（图：逐次逼近型 ADC 原理）*

STM32 用的是**逐次逼近型（Successive Approximation Register，SAR）**。它的思路可以用一个游戏说清楚：

> **猜数字。** 我想一个 0~4095 之间的整数，你每次只能问"比某个数大吗"，**最多问多少次能猜中？**
>
> 答案是 **12 次**——每次都猜当前范围的正中间，把范围砍一半：
>
> ```
> 范围 0~4095，猜 2048 → "小了"  → 范围缩到 2049~4095
> 范围 2049~4095，猜 3072 → "大了" → 范围缩到 2049~3071
> 范围 2049~3071，猜 2560 → ...
> ```
>
> `4096 = 2¹²`，每问一次砍一半，**12 次就能精确命中**。

**这就是"12 位 ADC"的由来，也是"逐次逼近"的字面意思。**

硬件上对应的是：

```
   输入电压 VIN
        │
        ▼
   ┌─────────┐      逐次逼近寄存器
   │ 比较器  │◄──── (SAR)：先试 2048，再试 1024……
   └────┬────┘            │
        │                 ▼
        │           ┌──────────┐
        │           │  内部 DAC │  ← 猜的那个值，转成电压
        │           └────┬─────┘
        └────────────────┘
        "猜的值比 VIN 大还是小？"
```

> **片内的这个 DAC，就是第 6 章要讲的那个 DAC 的"内部版本"。** 第 6 章你会自己做一个反过来的过程——**给定数字，输出模拟电压**。两个章节合起来看，就是完整的一对。

**逐次逼近需不需要时间？** 需要。每猜一次要一个时钟周期，12 位至少要 12 次比较——这就是 5.4 节要算的**转换时间**。

### 5.1.3 12 位 = 0~4095

| 项目 | 数值 |
|---|---|
| 位数 | **12 位** |
| 转换结果范围 | **0 ~ 4095**（= 2¹² − 1） |
| 输入电压范围 | **0 ~ 3.3 V**（本板，VREF 取 VDDA） |
| 分辨率 | 3.3 V ÷ 4095 ≈ **0.806 mV** |

**换算公式**（本章会反复用到，背下来更省事）：

```
        电压 = ADC值 × 3.3 / 4095
        ADC值 = 电压 × 4095 / 3.3
```

| ADC 值 | 电压 |
|---|---|
| 0 | 0 V |
| 1024 | 0.825 V |
| 2048 | 1.650 V |
| 3072 | 2.475 V |
| 4095 | 3.3 V |

> ⚠️ **3.3 V 是"参考电压 VREF"，不是"电源电压"的随便一个值。** F103C8T6 的 VREF 直接连到 VDDA（模拟电源），本板 VDDA 就是 3.3 V。
>
> **所以换算的准确性，直接依赖你板上 3.3 V 轨的准确度。** 如果 LDO 实际输出 3.28 V，你还按 3.3 算，就会有 0.6% 的系统误差。**要精确测量，先拿万用表量一下 VDDA 的真实值**（5.8 节细说）。

### 5.1.4 量化误差：为什么"精确"是有极限的

ADC 的输出是**整数**，而输入是**连续**的。这中间必然丢东西。

比如真实电压是 1.6505 V，ADC 只能告诉你 2048（对应 1.65000 V）或者 2049。**误差最大 ±0.5 LSB ≈ ±0.4 mV。**

这叫**量化误差**，是 ADC 天生的，**无法消除**，只能选位数更高的 ADC 来减小。

反过来还有个**向下取整**的问题：ADC 的输出实际上是 `floor(VIN / 3.3 × 4095)` 附近的值。所以**用 4095 还是 4096 做除数，本身就是个近似的选择**——本教程统一用 4095（与多数教程一致），它带来的误差远小于量化误差本身，不必纠结。

### 5.1.5 采样时间：为什么不是"瞬间读到"

逐次逼近的第一步不是比较，而是**采样**——把一个电容充到输入电压。

```
   ① 采样      ② 保持      ③ 量化编码
  开关闭合，   开关断开，   12 次逐次逼近
  电容充电     电压保持     （5.1.2 的猜数字）
   ↓↓↓          ↓↓↓          ↓↓↓
```

**电容充电是需要时间的**，而且外部信号源的内阻越大，充得越慢。

**这就是"采样时间"这个参数的意义**：给它留够充电时间，否则电容还没充满就开始比较，结果当然偏小。

| 信号源内阻 | 需要的采样时间 |
|---|---|
| 很低（运放输出、电位器抽头） | 短（1.5 ~ 7.5 周期就够） |
| 较高（**分压电路**：热敏 10K+10K、光敏 100K） | **长**（本教程统一用 55.5 周期） |

> **★ 本板三路信号源都是分压电路，内阻不算低。** 本教程一律用 `ADC_SampleTime_55Cycles5`（最长档），**省得你为每个通道单独推算**。5.8 节会解释这个选择的代价。

---

## 5.2 F103C8T6 的 ADC 资源

![ADC 框图](images/05-adc/ADC_框图.png)

*（图：ADC 框图）*

### 5.2.1 资源一览

| 项目 | STM32F103C8T6 |
|---|---|
| ADC 外设个数 | **ADC1、ADC2**（没有 ADC3） |
| 分辨率 | 12 位 |
| 外部输入通道 | **10 个** |
| 内部信号源 | 内部温度传感器、VREFINT |
| 转换时间 | 最快 **1 µs** |
| 参考电压 | VREF+ = VDDA = **3.3 V** |

> **"10 个外部通道"是怎么来的？** LQFP48 封装引出来的模拟引脚只有 **PA0~PA7**（8 个）和 **PB0、PB1**（2 个），一共 10 个。
>
> F103 系列更多引脚的型号（如 C8T6 的"大哥"RCT6）有 16 个外部通道，但**你的板子上只有 10 个**。

### 5.2.2 ★ 专节：板载 ADC1/2/3 ≠ 外设 ADC1/ADC2

**这一节必须读完再往下走，否则后面看代码会混乱。**

本板原理图上，那三路模拟信号被命名成了 `ADC1`、`ADC2`、`ADC3`：

```
   3V3 ── R59(热敏) ──┬── 网络名 ADC1 ── R48 10K ── GND
                      │
                      └───────────────► PA7

   3V3 ── R60(光敏) ──┬── 网络名 ADC2 ── R49 100K ── GND
                      │
                      └───────────────► PB0

   3V3 ── 电位器 ──────── 网络名 ADC3 ── 抽头 ──► PB1
```

而芯片**内部**也有两个叫 `ADC1` / `ADC2` 的东西——**转换器模块本身**。

**两边同名，但完全是两回事：**

| | 板载 ADC1/2/3 | 外设 ADC1/ADC2 |
|---|---|---|
| 是什么 | **原理图上三根导线的名字** | **芯片内部的两个转换器模块** |
| 在哪 | 只在原理图上 | 只在代码里 |
| 数量 | 3 个 | 2 个 |
| 关系 | 只是"第 1/2/3 路信号"的意思 | 真正干转换活的硬件 |

**本教程的区分写法**（[附录C](附录C-板载引脚总表.md) C.3.2 有同样的约定）：

| 指的是什么 | 怎么写 | 例子 |
|---|---|---|
| 原理图上那三根导线 | **板载 ADC1 / 板载 ADC2 / 板载 ADC3** | 「板载 ADC1 接的是热敏电阻」 |
| 芯片内部的模块 | **外设 ADC1 / 外设 ADC2** | 「PA7 是外设 ADC1 的通道 7」 |
| 代码里 | **只有外设** | `ADC_Init(ADC1, ...)` 里的 `ADC1` 永远是外设 |

> **★ 一句话记住：代码里出现的 `ADC1` 不可能是板载网络名。**
>
> 板载网络名是**一根导线的名字，不是一个变量，从来不会出现在 C 代码里**。看代码时看到 `ADC1`，直接理解成"ADC 外设"，不需要犹豫。
>
> 反过来看原理图时看到 `ADC1`，按上下文判断——旁边画着电阻和分压，那就是那根线。

> **历史说明**：本教程早期曾打算把板载网络名统一改称「模拟通道 A/B/C」来回避冲突。**该方案已废弃**——改名之后，读者看原理图还得先做一次名字映射，反而多一道弯。**加前缀区分更直接。**

### 5.2.3 本板三路信号：引脚、通道、拓扑

**下表是本章所有实验的依据**，与 [附录C](附录C-板载引脚总表.md) C.3.1 一致：

| 板载网络名 | 引脚 | 外设通道号 | 库里的常量 | 信号源与分压拓扑 |
|---|---|---|---|---|
| **板载 ADC1** | **PA7** | 通道 7 | `ADC_Channel_7` | `3V3 → R59(10K NTC) → 节点 → R48(10K) → GND` |
| **板载 ADC2** | **PB0** | 通道 8 | `ADC_Channel_8` | `3V3 → R60(GL5528) → 节点 → R49(100K) → GND` |
| **板载 ADC3** | **PB1** | 通道 9 | `ADC_Channel_9` | `3V3 → 电位器 → GND`，抽头输出 |

> **★ 通道号不用背，但"信号在分压的哪一边"必须记住**——它决定了电压随物理量**升高还是降低**。

**从拓扑推出的两条结论**（第 5.6、5.7 节的换算全靠它）：

```
   热敏电阻 R59 接在高边（靠近 3V3），固定的 R48 在低边：

        V = 3.3 × R48 / (R59 + R48) = 3.3 × 10K / (R59 + 10K)
                                                  ↑
              温度升高 → NTC 阻值下降 → 分母变小 → V ↑
```

| 物理量 | 传感器阻值 | 节点电压 |
|---|---|---|
| **温度 ↑** | R59 **↓** | **↑** |
| **光照 ↑** | R60 **↓** | **↑** |

**两者都是"物理量越强，ADC 读到电压越高"，极性一致**——这条结论让你不用为每个传感器单独记一次方向。

> **怎么验证我没写反？** 记住 NTC = Negative Temperature Coefficient，"负温度系数"就是**温度升、阻值降**。NTC 在高边，阻值降则分压到低边的电压升高。一步一步推，不会错。

---

## 5.3 四种转换模式

ADC 有**两个独立的选择**，组合出四种模式：

| 选择 | 取值 | 含义 |
|---|---|---|
| **扫描模式** `ScanConvMode` | `DISABLE` | 只转规则组里的**第 1 个**通道 |
| | `ENABLE` | 依次转规则组里的**全部**通道 |
| **连续模式** `ContinuousConvMode` | `DISABLE` | 转完一轮就**停**，等下次触发 |
| | `ENABLE` | 转完一轮**自动开始下一轮** |

四种组合：

| 模式 | 扫描 | 连续 | 行为 | 用在哪 |
|---|---|---|---|---|
| ① 单次 + 非扫描 | ✗ | ✗ | 触发一次 → 转 1 个通道 → 停 | **实验 6/7/8** |
| ② 连续 + 非扫描 | ✗ | ✓ | 反复转同 1 个通道 | 只关心一路、要连续刷新 |
| ③ 单次 + 扫描 | ✓ | ✗ | 触发一次 → 转 N 个通道 → 停 | 多路巡测，事件驱动 |
| ④ 连续 + 扫描 | ✓ | ✓ | 反复转 N 个通道 | **5.9 节 + DMA** |

> **本章大部分实验用模式 ①**，因为它最直观：**我让你转你才转，转完给我一个值。**
>
> **模式 ④ 配合 DMA 是实战中最常用的组合**——CPU 完全不用管，数据自己流进数组。5.9 节会做。

### 5.3.1 触发控制

![ADC 触发控制](images/05-adc/ADC_触发控制.png)

*（图：触发控制）*

两种触发方式：

| 触发源 | `ADC_ExternalTrigConv` 填什么 | 说明 |
|---|---|---|
| **软件触发** | `ADC_ExternalTrigConv_None` | 写一行代码让它转一次 |
| 外部触发 | `ADC_ExternalTrigConv_T1_CC1` 等 | 定时器/外部信号自动触发 |

**本章统一用软件触发**（`None`）——因为我们要自己控制节奏。

> **外部触发用在哪？** 第 6 章 DAC + 定时器输出波形时，需要**定时器精确地按固定频率触发**，那时才会用到。这里知道它存在即可。

### 5.3.2 数据对齐

12 位的转换结果，要放进一个 **16 位**的寄存器，多出 4 位怎么办？

![数据右对齐](images/05-adc/ADC_数据右对齐.png)

*（图：右对齐——D11~D0 放在低 12 位，高 4 位补 0）*

![数据左对齐](images/05-adc/ADC_数据左对齐.png)

*（图：左对齐——D11~D0 放在高 12 位，低 4 位补 0）*

| 对齐方式 | 读到的值 | 说明 |
|---|---|---|
| **右对齐** | 0 ~ 4095 | 直接用，最直观 |
| 左对齐 | 0 ~ 65520（步进 16） | 相当于"值 × 16"，**丢掉低 4 位精度** |

**本教程一律用右对齐**（`ADC_DataAlign_Right`）。

> **左对齐有什么用？** 它让 12 位结果直接落在 16 位变量的高位——**如果你只想取 8 位精度**，读高字节就行，省一次移位。属于优化手段，初学阶段完全不需要。

---

## 5.4 标准库 ADC 编程流程

### 5.4.1 七步 + 校准

```
① 开时钟        ADC1（APB2）  +  对应 GPIO 端口
② 配 ADCCLK     72 MHz ÷ 6 = 12 MHz（上限 14 MHz）
③ 配 GPIO       ★ 模拟输入 GPIO_Mode_AIN
④ 配 ADC        ADC_Init()：模式 / 扫描 / 连续 / 触发 / 对齐 / 通道数
⑤ 配规则组      ADC_RegularChannelConfig()：哪个通道、排第几、采样多久
⑥ 使能 + 校准   ADC_Cmd() → 复位校准 → 启动校准（顺序不能变）
⑦ 触发 + 读数   ADC_SoftwareStartConvCmd() → 等 EOC → ADC_GetConversionValue()
```

**①②③ 是第 4 章早就熟悉的套路**（开时钟 → 配引脚），只有 ③ 的模式是新东西。

### 5.4.2 ★ 第一个坑：GPIO 必须配 `GPIO_Mode_AIN`

第 4 章 4.2.3 就埋下了这一条，现在收回来：

```c
GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_1;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;   // ★ 模拟输入
GPIO_Init(GPIOB, &GPIO_InitStructure);
```

**为什么不能用上拉/下拉/浮空输入？**

三种情况的后果并不相同，要分开看（4.2.2 讲过）：

| 配成 | 内部电阻 | 后果 |
|---|---|---|
| 上拉输入 `IPU` | 40 kΩ 接到 3.3 V | 并联到信号上，读数**偏高**，且基本不随外界变化 |
| 下拉输入 `IPD` | 40 kΩ 接到 GND | 并联到信号上，读数**偏低**，甚至接近 0 |
| 浮空输入 `IN_FLOATING` | 两个都不接 | 没有 40 kΩ 并联，但数字输入通路仍开着、引脚高阻，读数会**漂移、不稳** |

以上拉输入为例，并联关系是这样的：

```
   3V3 ── R59(10K) ──┬──► PA7
                     │
                   R48 10K         ┌── 如果配成上拉输入，
                     │             │   这里还会并一个 40K 到 3V3！
                    GND            └──► 电压立刻被拉高，读数全错
```

**现象（上拉）**：读到的值明显偏高、且基本不随外界变化（因为 40 kΩ 比信号源的阻抗小得多，它说了算）。

**记住：ADC 引脚只配 `GPIO_Mode_AIN`，没有第二种选择。**

> 这也解释了 4.2.3 那句话——`AIN` 模式不只是"配成输入"，它**把引脚后面的整条数字通路都关掉了**，让电压原封不动进 ADC。

### 5.4.3 为什么 ADC 和 GPIO 用的是同一个开时钟函数

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 | RCC_APB2Periph_GPIOB, ENABLE);
```

**因为 ADC 也在 APB2 总线上**，和 GPIO 一样。

第 4 章 4.2.1 讲过"总线决定开时钟函数"：

| 总线 | 外设 | 函数 |
|---|---|---|
| **APB2**（72 MHz） | GPIO、**ADC**、USART1、SPI1 | `RCC_APB2PeriphClockCmd()` |
| APB1（36 MHz） | **DAC**、TIM2~7、USART2 | `RCC_APB1PeriphClockCmd()` |

**一行代码开了两个外设的时钟，这是允许的**（用 `|` 合并），而且它们本来就在同一条总线上。

> **★ 记住这张表——第 6 章的 DAC 挂 APB1，是全书唯一一个要用 `APB1` 版本的地方**，也是最容易写错的地方。现在先记住"ADC 和 GPIO 一样在 APB2"。

### 5.4.4 ADCCLK 为什么是 12 MHz

```c
RCC_ADCCLKConfig(RCC_PCLK2_Div6);    // PCLK2 = 72 MHz, 72 ÷ 6 = 12 MHz
```

**ADC 的时钟不能超过 14 MHz**（数据手册规定）。而 APB2 是 72 MHz，必须分频：

| 分频 | 结果 | 是否可用 |
|---|---|---|
| `RCC_PCLK2_Div2` | 36 MHz | ❌ 超限 |
| `RCC_PCLK2_Div4` | 18 MHz | ❌ 超限 |
| **`RCC_PCLK2_Div6`** | **12 MHz** | ✅ **本教程用这个** |
| `RCC_PCLK2_Div8` | 9 MHz | ✅ 可用，但慢 |

**用 `Div6` 拿 12 MHz，是"不超限的前提下最快"的选择。**

**转换时间怎么算：**

```
   总转换时间 = 采样时间 + 12.5 个 ADC 周期
                        ↑          ↑
              5.1.5 讲的充电时间   逐次逼近的 12 次比较（+1 个周期）
```

用本教程的采样时间 55.5 周期：

```
   TCONV = (55.5 + 12.5) / 12 MHz = 68 / 12 MHz ≈ 5.67 µs
```

**约 5.7 µs 转一次——每秒能转约 17 万次。** 对本章所有实验都绰绰有余。

> **想要 1 µs？** 把采样时间改成 `ADC_SampleTime_1Cycles5`，`(1.5+12.5)/12MHz = 1.17 µs`。**但前提是你的信号源内阻足够低**——本板的分压电路不满足，改成最短采样时间会**读数偏低**。5.8 节细说。

### 5.4.5 校准：为什么必须在使能之后

```c
ADC_Cmd(ADC1, ENABLE);                      // ① 先使能

ADC_ResetCalibration(ADC1);                 // ② 复位校准寄存器
while (ADC_GetResetCalibrationStatus(ADC1) == SET);   //    等它完成

ADC_StartCalibration(ADC1);                 // ③ 启动校准
while (ADC_GetCalibrationStatus(ADC1) == SET);        //    等它完成
```

**这三步的顺序是固定的**，为什么？

逐次逼近 ADC 内部有一组**电容阵列**，用来"猜"电压。这些电容的实际值不可能完全等于设计值，会有微小偏差——偏差不修正，转换结果就不准。

**校准就是让 ADC 自己测量这组电容的误差，算出一个修正码。** 官方的建议是：**每次上电后执行一次校准。**

| 步骤 | 为什么这么排 |
|---|---|
| 先 `ADC_Cmd(ENABLE)` | 校准电路需要 ADC 处于**已使能**状态才能工作 |
| 两个 `while` 等待 | 校准需要若干个 ADC 时钟周期，**不等的话后续转换结果不可靠** |

> ⚠️ **两个 `while` 千万别省。** 不等待的现象是"读数有固定偏差"或者"偶尔跳一个很大的值"，很难查。
>
> **还有一条**：数据手册要求启动校准前，ADC 必须**断电至少两个 ADC 时钟周期**。对刚上电的程序来说，`ADC_Cmd(ENABLE)` 之前的这段时间天然满足，不用特别处理。

### 5.4.6 完整初始化代码 + 读值函数

以 **PB1（板载 ADC3，电位器）** 为例：

```c
#include "stm32f10x.h"

/* ---------- ADC 初始化 ---------- */
void AD_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    ADC_InitTypeDef  ADC_InitStructure;

    /* ① 开时钟：ADC1 和 GPIOB 都在 APB2 上 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 | RCC_APB2Periph_GPIOB, ENABLE);

    /* ② 配置 ADCCLK：72 MHz ÷ 6 = 12 MHz */
    RCC_ADCCLKConfig(RCC_PCLK2_Div6);

    /* ③ PB1 配成模拟输入 ★ 只能用 AIN */
    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_1;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOB, &GPIO_InitStructure);

    /* ④ 配置 ADC1 */
    ADC_InitStructure.ADC_Mode               = ADC_Mode_Independent;      // 单独用 ADC1
    ADC_InitStructure.ADC_ScanConvMode       = DISABLE;                  // 非扫描
    ADC_InitStructure.ADC_ContinuousConvMode = DISABLE;                  // 单次
    ADC_InitStructure.ADC_ExternalTrigConv   = ADC_ExternalTrigConv_None; // 软件触发
    ADC_InitStructure.ADC_DataAlign          = ADC_DataAlign_Right;      // 右对齐
    ADC_InitStructure.ADC_NbrOfChannel       = 1;                        // 只转 1 个通道
    ADC_Init(ADC1, &ADC_InitStructure);

    /* ⑤ 规则组第 1 个位置 = 通道 9（PB1），采样 55.5 周期 */
    ADC_RegularChannelConfig(ADC1, ADC_Channel_9, 1, ADC_SampleTime_55Cycles5);

    /* ⑥ 使能 + 校准 */
    ADC_Cmd(ADC1, ENABLE);

    ADC_ResetCalibration(ADC1);
    while (ADC_GetResetCalibrationStatus(ADC1) == SET);

    ADC_StartCalibration(ADC1);
    while (ADC_GetCalibrationStatus(ADC1) == SET);
}

/* ---------- 读一次 ADC，返回 0~4095 ---------- */
uint16_t AD_GetValue(void)
{
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);                     // 软件触发一次转换
    while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);     // 等 EOC（转换完成）
    return ADC_GetConversionValue(ADC1);                        // 读结果
}
```

**`ADC_RegularChannelConfig()` 的四个参数**是本章唯一的长参数列表，记清楚：

| 位置 | 参数 | 本例 | 说明 |
|---|---|---|---|
| 1 | ADC 外设 | `ADC1` | 用哪个 ADC |
| 2 | 通道 | `ADC_Channel_9` | 哪个通道（对应 PB1） |
| 3 | 序列位置 | `1` | 在规则组里排第几 |
| 4 | 采样时间 | `ADC_SampleTime_55Cycles5` | 采样多久 |

**`AD_GetValue()` 里的两行**就是"触发-等待"的经典配合：

```
   写 ADC_SoftwareStartConvCmd(ENABLE)   →  ADC 开始转换
                 │
                 ▼
   while (EOC == RESET)  ← 死等，直到转换完成
                 │
                 ▼
   ADC_GetConversionValue()  ← 读数据寄存器
```

> **EOC 标志需要手动清除吗？** 不需要。SPL 的 `ADC_GetConversionValue()` 读的是 `ADC1->DR`，**读 DR 就会自动清 EOC**，所以读值之后可以直接开始下一次转换。

---

## 5.5 实验 6：电位器电压采集

> **这是本章最重要的一个实验**，因为它**可验证**——电位器拧到哪，万用表量得到；ADC 读出来的应该和万用表基本一致。
>
> **先把这一个做扎实，实验 7、8 就都是它的变体。**

### ① 原理图

![板载三路模拟源原理图](images/05-adc/板载三路模拟源原理图.png)

*（图：板载三路模拟信号源）*

电位器那一路：

```
     3V3
      │
      ├──── 电位器 pin3
      │
      │        ┌── pin2（抽头）──────► PB1
      │        │
      ├──── pin1
      │
     GND
```

**电位器就是一个可变分压器**：抽头在内部电阻上滑动，抽头到两端的比例可调，所以输出电压是 **0 ~ 3.3 V 连续可调**——这是三路信号里唯一能做到满量程的。

### ② 硬件连接

**不需要接线。** 电位器已经焊在板上，抽头直接连 PB1。

### ③ 原理讲解

分压公式（电位器本质是滑动变阻器串成两个电阻）：

```
   V_PB1 = 3.3 × R下 / (R上 + R下)
```

- 抽头滑到**最上**（pin3 侧）→ R上 = 0 → **V = 3.3 V**
- 抽头滑到**最下**（pin1 侧）→ R上 = 全阻 → **V = 0 V**
- 抽头在**中间** → **V ≈ 1.65 V**

**所以它最适合入门**：你可以把抽头拧到任意位置，然后用万用表量出真实电压，和 ADC 读数对照。

### ④ 完整代码

**复用第 4 章 4.9.2 的串口代码**（这是第 4 章给你的"调试基础设施"，现在开始派上用场）：

```c
#include "stm32f10x.h"
#include <stdio.h>

/* ========== 延时：与第 4 章 4.5 相同（循环次数需按 4.5 的方法校准） ========== */
void Delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 12000; j++);
}

/* ========== 串口部分：与第 4 章 4.9.2 完全相同 ========== */
void USART1_Init(uint32_t bound)
{
    GPIO_InitTypeDef  GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_USART1, ENABLE);

    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    USART_InitStructure.USART_BaudRate            = bound;
    USART_InitStructure.USART_WordLength          = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits            = USART_StopBits_1;
    USART_InitStructure.USART_Parity              = USART_Parity_No;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_InitStructure.USART_Mode                = USART_Mode_Tx | USART_Mode_Rx;
    USART_Init(USART1, &USART_InitStructure);

    USART_Cmd(USART1, ENABLE);
}

int fputc(int ch, FILE *f)
{
    USART_SendData(USART1, (uint8_t)ch);
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    return ch;
}

/* ========== ADC 部分 ========== */
void AD_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    ADC_InitTypeDef  ADC_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 | RCC_APB2Periph_GPIOB, ENABLE);

    RCC_ADCCLKConfig(RCC_PCLK2_Div6);                   // 12 MHz

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_1;          // PB1
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;       // ★ 模拟输入
    GPIO_Init(GPIOB, &GPIO_InitStructure);

    ADC_InitStructure.ADC_Mode               = ADC_Mode_Independent;
    ADC_InitStructure.ADC_ScanConvMode       = DISABLE;
    ADC_InitStructure.ADC_ContinuousConvMode = DISABLE;
    ADC_InitStructure.ADC_ExternalTrigConv   = ADC_ExternalTrigConv_None;
    ADC_InitStructure.ADC_DataAlign          = ADC_DataAlign_Right;
    ADC_InitStructure.ADC_NbrOfChannel       = 1;
    ADC_Init(ADC1, &ADC_InitStructure);

    ADC_RegularChannelConfig(ADC1, ADC_Channel_9, 1, ADC_SampleTime_55Cycles5);

    ADC_Cmd(ADC1, ENABLE);

    ADC_ResetCalibration(ADC1);
    while (ADC_GetResetCalibrationStatus(ADC1) == SET);
    ADC_StartCalibration(ADC1);
    while (ADC_GetCalibrationStatus(ADC1) == SET);
}

uint16_t AD_GetValue(void)
{
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
    while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);
    return ADC_GetConversionValue(ADC1);
}

/* ========== 主函数 ========== */
int main(void)
{
    uint16_t adc;
    float    voltage;                   // 单位：V

    USART1_Init(9600);
    AD_Init();

    /* ★ 串口里的提示文字一律用英文——原因见本节 ⑥ 避坑提示 */
    printf("ADC ready - PB1 (board ADC3, potentiometer)\r\n");
    printf("Turn the pot and watch the value change\r\n\r\n");

    while (1)
    {
        adc     = AD_GetValue();
        voltage = adc * 3.3f / 4095.0f;         // 换算公式（5.1.3）

        printf("ADC = %4d   V = %.3f V\r\n", adc, voltage);

        Delay_ms(100);                           // 每 100 ms 打一次，别刷太快
    }
}
```

> **上面这个 `Delay_ms()` 和 `USART1_Init()` 都是从第 4 章原样搬过来的**——**这就是第 4 章的价值**：调试基础设施搭好一次，后面每章都能直接用。

### ⑤ 现象验证

**✅ 串口输出随电位器扭动而变化：**

```
ADC ready - PB1 (board ADC3, potentiometer)
Turn the pot and watch the value change

ADC = 2047   V = 1.650 V
ADC = 2048   V = 1.650 V
ADC = 3821   V = 3.079 V     ← 往一个方向拧到底
ADC = 4095   V = 3.300 V     ← 拧到最上
ADC =   12   V = 0.010 V     ← 拧到最下
```

**★ 关键验证：用万用表量 PB1 对 GND 的电压，和串口打印的 `V` 对照。**

两者应该**基本一致**（差几毫伏到十几毫伏都正常）。

**如果对不上**：

| 症状 | 原因 |
|---|---|
| ADC 读数固定不变 | 引脚没配 `AIN`（5.4.2 的坑） |
| 读数偏高很多且不随电位器变 | 配成了**上拉输入**，40 kΩ 并到了 3.3 V |
| 读数偏低很多甚至接近 0 | 配成了**下拉输入**，40 kΩ 并到了 GND |
| 读数忽高忽低、漂移不稳 | 配成了**浮空输入**，数字通路还开着 |
| 拧到底也到不了 4095 | 电位器两端的实际电压不是干净的 0 和 3.3 V（见下条） |
| 万用表和 ADC 差几十毫伏 | VREF 实际不是 3.300 V（5.8 节） |

### ⑥ 避坑提示

> ⚠️ **`GPIO_Mode_AIN` 是本实验的命门。** 本章三个实验都是同一个坑，5.4.2 已经详细讲过。**每次配 ADC 引脚，先确认这一行。**

> ⚠️ **别把 `ADC_Channel_9` 写成 `ADC_Channel_1`。** 通道号和引脚号**没有简单对应关系**——PB1 是**通道 9**，不是通道 1。查 5.2.3 的表，不要望文生义。

> ⚠️ **串口刷太快会看不清。** 不加延时的话每秒会打印上万行，串口助手会滚成一片。加个 100 ms 延时正好。

> ⚠️ **量电压时表笔别打滑。** PB1 和相邻引脚挨得很近，打滑短接到 3.3 V 或 GND 一般不会烧（有保护二极管），但可能让你困惑半天。**养成一只手扶板、一只手拿表笔的习惯。**

> ⚠️ **★ 为什么代码里的串口提示全用英文？**
>
> 串口打印中文，需要**三个编码全部一致**：Keil 保存源文件的编码、编译器处理字符串常量的编码、串口助手显示用的编码。任何一环不一致，屏幕上就是一堆乱码——**而乱码会让你误以为是 ADC 出了问题**，白查半天。
>
> **初学阶段用英文提示，直接绕开这个坑。** 本章所有 `printf` 的格式串都是纯 ASCII。
>
> **如果你确实想打印中文**：① Keil 里 `Edit → Configuration → Encoding` 选 `UTF-8`（或 `Chinese GB2312`）；② 串口助手里把编码设成**同样的一种**。两边一致了就能正常显示。第 2 章 2.10 的故障排查里有这一条。

---

## 5.6 实验 7：光敏电阻

### ① 原理图

见上节的板载三路模拟源原理图，光敏那一路：

```
   3V3 ── R60(GL5528) ──┬── 板载 ADC2 ──► PB0
                        │
                      R49 100K
                        │
                       GND
```

### ② 硬件连接

**不需要接线。** 光敏电阻 R60 已焊在板上，分压节点直连 PB0。

### ③ 原理讲解

**GL5528** 是光敏电阻的常见型号，它的阻值随光照强度变化：

| 光照条件 | 阻值（典型） |
|---|---|
| 很亮（室内灯光直射） | 约 10 ~ 20 kΩ |
| 一般室内 | 约 50 ~ 100 kΩ |
| 遮住 / 黑暗 | 约 1 MΩ 以上 |

**和热敏电阻一样接在高边**（5.2.3 已推过）：

```
   V_PB0 = 3.3 × 100K / (R60 + 100K)
                  ↑
      光照增强 → R60 减小 → 分母减小 → 电压升高
```

**估算一下量程**（让你心里有个数）：

| 情况 | R60 | V_PB0 |
|---|---|---|
| 亮 | 20 kΩ | 3.3 × 100/120 = **2.75 V** |
| 中等 | 100 kΩ | 3.3 × 100/200 = **1.65 V** |
| 暗 | 1 MΩ | 3.3 × 100/1100 = **0.30 V** |

**所以光敏这一路的输出范围大约是 0.3 ~ 2.8 V，覆盖不了满量程**——这是电路设计决定的，不是 bug。

> **为什么选 100 kΩ 做分压电阻？** 为了让"室内常见光照"这个区间落在 ADC 量程的中段。如果选 10 kΩ，室内光照下 R60 远大于它，电压会被压到接近 0，反而不好分辨。

### ④ 完整代码

**做法：把实验 6 的整个代码复制过来，只改下面 3 处。** 改完就是一份完整、可编译的程序——不需要参考任何别的东西。

```c
/* ═══════ 改动 1：AD_Init() 里的引脚，由 PB1 改为 PB0 ═══════ */
    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_0;          // ← 原为 GPIO_Pin_1
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOB, &GPIO_InitStructure);

/* ═══════ 改动 2：AD_Init() 里的通道，由 9 改为 8 ═══════ */
    ADC_RegularChannelConfig(ADC1, ADC_Channel_8, 1, ADC_SampleTime_55Cycles5);

/* ═══════ 改动 3：main() 里的打印文字（仅为提示清晰） ═══════ */
    printf("ADC ready - PB0 (board ADC2, photoresistor)\r\n");
```

**其余部分一字不改**——包括时钟（PB0 也在 GPIOB 上，`RCC_APB2Periph_GPIOB` 不用动）、`AD_GetValue()`、换算公式、延时。

**如果还想更进一步，把光敏电阻的阻值也算出来：**

```c
/* adc → 光敏电阻阻值（Ω） */
float AD_ToLightResistance(uint16_t adc)
{
    float v_mv;

    v_mv = adc * 3300.0f / 4095.0f;         // 转成毫伏

    if (v_mv < 1.0f) return 1e9f;           // 防止除零

    /* V = 3.3 × 100K / (R60 + 100K)  →  R60 = 100K × (3300 / v_mv - 1) */
    return 100000.0f * (3300.0f / v_mv - 1.0f);
}
```

> **注意这个推导和实验 6 是同一个套路**：传感器在高边 → `R传感 = R固定 × (VREF / V节点 − 1)`。**换哪个传感器都是这个式子，只是 R固定 不同（热敏 10K、光敏 100K）。**

### ⑤ 现象验证

**✅ 用手遮住光敏电阻，串口数值下降；用手机手电筒照它，数值上升。**

预期范围：**约 370 ~ 3400**（对应 0.3 ~ 2.75 V）。

**★ 有个很有意思的观察点**：把手机屏幕凑近光敏电阻，然后**调屏幕亮度**——你会发现 ADC 值几乎是**连续地**跟着变。

**这就是第 4 章做不到的事。** 第 4 章你只能知道"亮/暗"，现在你能知道"有多亮"。

### ⑥ 避坑提示

> ⚠️ **别在阳光直射下做实验。** 阳光的红外成分很强，光敏电阻的响应特性和室内灯光差别很大，读数会和你预期不一致。**室内正常光照条件下实验最稳定。**

> ⚠️ **手遮的时候别碰到引脚。** 手指的电阻在几十 kΩ ~ MΩ 量级，**和光敏电阻是同一个数量级**，碰到引脚会直接把电压带偏。遮光要悬空遮，不要贴上去。

> ⚠️ **读数在小范围来回跳是正常的。** 这是量化误差 + 电源纹波的共同结果。要更稳就在软件里做平均（5.8 节）。

---

## 5.7 实验 8：热敏电阻 + 温度换算

### ① 原理图

见板载三路模拟源原理图，热敏那一路：

```
   3V3 ── R59(MF52A103F3950) ──┬── 板载 ADC1 ──► PA7
                               │
                             R48 10K
                               │
                              GND
```

### ② 硬件连接

**不需要接线。** 热敏电阻 R59 已焊在板上，分压节点直连 PA7。

### ③ 原理讲解

前两个实验都是"读一个数、看一眼"。**这个实验不一样——它要把 ADC 值变成一个有物理意义的温度值。**

整个过程分三步：

```
   ADC 值  ──①──►  节点电压 V  ──②──►  NTC 阻值 R  ──③──►  温度 T
   (0~4095)         (0~3.3V)          (Ω)                 (℃)
```

**第 ① 步**：5.1.3 的换算公式，已经会了。

**第 ② 步**：由分压公式反解。热敏电阻在**高边**：

```
   V = 3.3 × R48 / (R59 + R48)          R48 = 10 kΩ

   → R59 + 10K = 3.3 × 10K / V

   → R59 = 10K × (3.3 / V − 1)          ← 记住这个式子
```

**第 ③ 步：B 参数方程**

NTC 的阻值和温度不是线性关系，而是指数关系。工程上最常用的近似是 **B 参数方程**：

```
   1/T  =  1/T₀  +  (1/B) × ln(R/R₀)

   其中：
     T  = 当前温度（单位：开尔文 K）
     T₀ = 参考温度 298.15 K（即 25 ℃）
     R  = 当前阻值（Ω）
     R₀ = 25 ℃ 时的阻值 = 10 kΩ      ← 型号 "103" 就是这个意思
     B  = 材料常数 = 3950            ← 型号 "F3950" 就是这个意思
```

**型号 `MF52A103F3950` 本身就是参数表**，拆开看：

| 片段 | 含义 |
|---|---|
| `MF52` | 系列（环氧封装 NTC） |
| `A` | 精度等级 |
| `103` | **R₀ = 10 × 10³ = 10 kΩ** |
| `F` | 25 ℃ 时阻值容差 ±1% |
| `3950` | **B = 3950 K** |

**看到型号就知道参数，不用翻手册**——这是个很实用的习惯。

**解出 T（单位要转回摄氏度）：**

```
   T(K) = 1 / ( 1/298.15 + ln(R/10000) / 3950 )
   温度(℃) = T(K) − 273.15
```

**验证一下这个公式对不对**（养成验算习惯）：
- 当 R = 10000 Ω 时，`ln(1) = 0` → `T = 1/(1/298.15) = 298.15 K = 25 ℃` ✅

### ④ 完整代码

```c
#include "stm32f10x.h"
#include <stdio.h>
#include <math.h>               // ★ 用 logf() 需要这一行

/* ========== 延时：同实验 6 ========== */
void Delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 12000; j++);
}

/* ========== 串口：同实验 6 ========== */
void USART1_Init(uint32_t bound)
{
    GPIO_InitTypeDef  GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_USART1, ENABLE);

    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    USART_InitStructure.USART_BaudRate            = bound;
    USART_InitStructure.USART_WordLength          = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits            = USART_StopBits_1;
    USART_InitStructure.USART_Parity              = USART_Parity_No;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_InitStructure.USART_Mode                = USART_Mode_Tx | USART_Mode_Rx;
    USART_Init(USART1, &USART_InitStructure);

    USART_Cmd(USART1, ENABLE);
}

int fputc(int ch, FILE *f)
{
    USART_SendData(USART1, (uint8_t)ch);
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    return ch;
}

/* ---------- ADC 初始化：注意三处与实验 6 不同 ---------- */
void AD_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    ADC_InitTypeDef  ADC_InitStructure;

    /* 改动 1：PA7 在 GPIOA 上，改成开 GPIOA */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 | RCC_APB2Periph_GPIOA, ENABLE);

    RCC_ADCCLKConfig(RCC_PCLK2_Div6);

    /* 改动 2：引脚改为 PA7 */
    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    ADC_InitStructure.ADC_Mode               = ADC_Mode_Independent;
    ADC_InitStructure.ADC_ScanConvMode       = DISABLE;
    ADC_InitStructure.ADC_ContinuousConvMode = DISABLE;
    ADC_InitStructure.ADC_ExternalTrigConv   = ADC_ExternalTrigConv_None;
    ADC_InitStructure.ADC_DataAlign          = ADC_DataAlign_Right;
    ADC_InitStructure.ADC_NbrOfChannel       = 1;
    ADC_Init(ADC1, &ADC_InitStructure);

    /* 改动 3：通道由 9 改为 7 */
    ADC_RegularChannelConfig(ADC1, ADC_Channel_7, 1, ADC_SampleTime_55Cycles5);

    ADC_Cmd(ADC1, ENABLE);

    ADC_ResetCalibration(ADC1);
    while (ADC_GetResetCalibrationStatus(ADC1) == SET);
    ADC_StartCalibration(ADC1);
    while (ADC_GetCalibrationStatus(ADC1) == SET);
}

uint16_t AD_GetValue(void)
{
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
    while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);
    return ADC_GetConversionValue(ADC1);
}

/* ========== 新增：把 ADC 值换算成温度 ========== */
/*   返回摄氏度；超出合理范围时返回 -999 表示读数异常           */
float AD_ToTemperature(uint16_t adc)
{
    float v_mv;         // 节点电压，毫伏
    float r_ntc;        // 热敏电阻阻值，欧姆
    float t_kelvin;     // 开尔文温度

    /* ① ADC 值 → 电压（毫伏） */
    v_mv = adc * 3300.0f / 4095.0f;

    /* 边界保护：避免后面除零或 log 出负数 */
    if (v_mv < 1.0f || v_mv > 3299.0f)
        return -999.0f;

    /* ② 电压 → 阻值：R59 = 10K × (3300 / v_mv − 1) */
    r_ntc = 10000.0f * (3300.0f / v_mv - 1.0f);

    /* ③ 阻值 → 温度：B 参数方程 */
    t_kelvin = 1.0f / (1.0f / 298.15f + logf(r_ntc / 10000.0f) / 3950.0f);

    /* ④ 开尔文 → 摄氏度 */
    return t_kelvin - 273.15f;
}

/* ========== 主函数 ========== */
int main(void)
{
    uint16_t adc;
    float    voltage;
    float    temperature;

    USART1_Init(9600);
    AD_Init();

    printf("ADC ready - PA7 (board ADC1, thermistor)\r\n\r\n");

    while (1)
    {
        adc         = AD_GetValue();
        voltage     = adc * 3.3f / 4095.0f;
        temperature = AD_ToTemperature(adc);

        if (temperature > -900.0f)
            printf("ADC = %4d   V = %.3f V   T = %.1f C\r\n",
                   adc, voltage, temperature);
        else
            printf("ADC = %4d   V = %.3f V   T = invalid\r\n",
                   adc, voltage);

        Delay_ms(100);
    }
}
```

> ⚠️ **`#include <math.h>` 别忘了**，否则 `logf()` 无法识别。Keil 的 MicroLIB（第 4 章 4.9.2 让你勾的那个）**包含 `logf()`**，不需要额外配置。

### ⑤ 现象验证

**✅ 串口输出带温度值：**

```
ADC ready - PA7 (board ADC1, thermistor)

ADC = 2047   V = 1.650 V   T = 25.0 C
ADC = 2040   V = 1.644 V   T = 25.4 C
```

**★ 怎么验证准不准？** 手边能用的参照物：

| 方法 | 做法 |
|---|---|
| **手指捏住热敏电阻** | 体温约 33~35 ℃，读数应该**上升几度**（要等几秒，热敏有热惯性） |
| **对着哈气** | 温度快速上升，然后慢慢回落 |
| **放空调出风口** | 温度下降 |
| **对照室温** | 房间常温应该在 **20~28 ℃** 之间 |

**读数在 25 ℃ 附近、用手一捏就上升——这个实验就算做成了。**

> **为什么指尖温度是 33 ℃ 而不是 37 ℃？** 手指表面的皮肤温度比核心体温低，而且隔着电阻的封装，测到的会更低。**能观察到明显的上升趋势就够了**，不必强求绝对值对得准。

### ⑥ 避坑提示

> ⚠️ **不要指望这个测温有多准。** B 参数方程用的是**标称参数**（R₀ = 10 kΩ、B = 3950），而实际元件的阻值有 ±1% 容差、B 值有 ±1% 容差；再加上 VREF 的实际值、ADC 的偏移误差——**综合下来几度的误差很正常**。
>
> **想准一点**：① 用万用表实测 VDDA 的真实值，替换代码里的 3300.0f（5.8 节）；② 用查表法代替 B 参数方程（用实测数据标定）。

> ⚠️ **热惯性会让读数滞后好几秒。** 热敏电阻有封装，热传导需要时间。**用手捏住后不要立刻读数**，等 3~5 秒稳定下来。

> ⚠️ **`logf()` 的参数不能是 0 或负数。** 这就是为什么上面加了 `v_mv < 1.0f` 的保护——如果引脚没接好、读数为 0，`r_ntc` 会变成很大的数或者负数，`logf` 会返回无意义的值（甚至 NaN）。**任何做数学运算的地方，都要先想清楚边界情况。**

---

## 5.8 提高精度

到这一步你已经能读出电压和温度了。这一节讲**怎么让读数更准、更稳**——都是实战里必须知道的。

### 5.8.1 采样时间要匹配信号源内阻

5.1.5 讲过原理：采样就是一个电容充电的过程。**充电需要时间，时间够不够取决于信号源内阻。**

本板三路信号源的等效内阻（戴维南等效 = 两个分压电阻并联）：

| 通道 | 拓扑 | 等效源阻抗 | 需要的采样时间 |
|---|---|---|---|
| PB1 电位器 | 抽头将电阻分成两半 | 最坏约几 kΩ（中位时） | 短 |
| PA7 热敏 | 10K ∥ 10K（25 ℃ 时） | **5 kΩ** | 中 |
| PB0 光敏 | 20K ∥ 100K（亮时） | **约 17 kΩ** | **长** |
| PB0 光敏 | 1M ∥ 100K（暗时） | **约 91 kΩ** | **很长** |

**光敏在暗环境下的等效源阻抗接近 100 kΩ**，这是三路里最苛刻的。

| 采样时间 | 周期数 | 12 MHz 下耗时 | 适合的源阻抗 |
|---|---|---|---|
| `ADC_SampleTime_1Cycles5` | 1.5 | 1.17 µs | < 1 kΩ |
| `ADC_SampleTime_7Cycles5` | 7.5 | 1.67 µs | ~ 10 kΩ |
| `ADC_SampleTime_28Cycles5` | 28.5 | 3.42 µs | ~ 40 kΩ |
| **`ADC_SampleTime_55Cycles5`** | **55.5** | **5.67 µs** | **~ 80 kΩ 以上** |

**本教程统一用 55.5**，就是为了同时覆盖最苛刻的光敏通道。

**采样时间不够会怎样？** 电容没充满就开始比较，结果**总是偏低**，而且**偏低量随信号源阻抗变化**——你会看到一个"读数偏小且不线性"的系统性误差，非常难查。

> **代价是什么？** 慢。55.5 周期意味着每秒最多转 17 万次，如果你要做高速采集（比如示波器），这个速度不够。
>
> **那时的做法**：加一级电压跟随器（运放）把源阻抗降到接近 0，然后就能用最短采样时间跑满速。**本教程的实验不需要。**

### 5.8.2 VREF 不是理想的 3.3 V

**所有换算都建立在"VREF = 3.300 V"这个假设上。** 但实际呢？

```
   常见的 3.3 V LDO（如 AMS1117-3.3）
   ├── 典型精度    ±1%    → 3.267 ~ 3.333 V
   └── 加上负载调整率、温漂，实际可能差 ±2%
```

**±1% 的 VREF 误差 = ±1% 的测量误差。** 对你测的 1.650 V 来说，就是 ±16 mV。

**怎么校准？**

1. 用万用表（要比 ADC 准得多）量**板上 3.3 V 轨对 GND 的实际电压**，比如量到 **3.284 V**
2. 把代码里的 `3.3f` 和 `3300.0f` 全部替换成 `3.284f` 和 `3284.0f`

```c
/* 校准前 */
voltage = adc * 3.3f / 4095.0f;

/* 校准后（假设实测 VDDA = 3.284 V） */
#define VREF_V      3.284f
#define VREF_MV     3284.0f

voltage = adc * VREF_V / 4095.0f;
```

> **★ 定义成宏，改一处就全改了。** 这比在代码里到处找 `3.3f` 可靠得多——尤其是当你有四个换算函数的时候。

### 5.8.3 上电校准必须做

5.4.5 已经讲过。**再强调一次：两个 `while` 等待不能省。** 这是"读数有固定偏差"最常见的来源。

### 5.8.4 多次平均：把随机噪声压下去

即使什么都不动，ADC 读数也会在小范围跳动（±3 ~ ±5）。这是**量化误差 + 电源纹波 + 热噪声**的共同结果。

**最简单的办法：多读几次取平均。**

```c
/* 读 times 次，返回平均值 */
uint16_t AD_GetValueAverage(uint8_t times)
{
    uint32_t sum = 0;
    uint8_t  i;

    for (i = 0; i < times; i++)
        sum += AD_GetValue();

    return (uint16_t)(sum / times);
}
```

**为什么有效？** 假设噪声是随机的（有大有小），平均之后正负抵消，**理论上能降低到原来的 1/√N**：

| 次数 N | 噪声降到 | 耗时（55.5 周期采样） |
|---|---|---|
| 1 | 100% | 5.7 µs |
| 4 | 50% | 23 µs |
| **16** | **25%** | **91 µs** |
| 64 | 12.5% | 364 µs |

**本教程推荐 N = 16**：抖动明显减小，耗时还不到 0.1 ms，对本章实验毫无负担。

**用起来只需要改一行：**

```c
/* 原来 */
adc = AD_GetValue();

/* 改成 */
adc = AD_GetValueAverage(16);
```

> **★ 但平均有个前提：被测量的值在采样期间必须基本不变。** 如果你在扭电位器，16 次平均会把"扭动过程"也平均进去，反应变迟钝。
>
> **规律**：**测稳态量（温度、光照）用平均；测变化的量（旋钮、波形）不要平均。**

### 5.8.5 中值滤波：对付"偶发跳变"

平均对付的是**密集的小噪声**，但对**偶发的单点大跳变**（比如电源瞬间扰动、串口发送时的干扰）效果不好——一个离谱的值会把平均值整个拉偏。

**中值滤波**正好补这个短板：**取 N 次读数，排序，取中间那个。**

- N 取奇数（常用 3、5、7）
- **单个离群值完全不影响结果**

| 滤波方式 | 对付小噪声 | 对付偶发大跳变 |
|---|---|---|
| 算术平均 | ✅ 好 | ❌ 差 |
| 中值 | ❌ 一般 | ✅ **好** |
| 先中值再平均 | ✅ | ✅ | 

> **实际工程里常用"中值 + 平均"两级串联**。本教程的实验中平均就够了；如果你发现数值偶尔跳一下，再叠加中值。

---

## 5.9 进阶：中断与 DMA

到 5.8 为止，读 ADC 的方式都是**轮询**：

```c
ADC_SoftwareStartConvCmd(ADC1, ENABLE);
while (ADC_GetFlagStatus(ADC1, ADC_FLAG_EOC) == RESET);   // ← CPU 在这里空等
return ADC_GetConversionValue(ADC1);
```

**CPU 在 `while` 里空转等 5.7 µs，什么都干不了。** 单通道低频采集无所谓，但有两个场景会出问题：

1. **主循环里有别的活要干**（比如同时刷新 OLED、响应按键）
2. **要多路高速采集**（三路连续扫描）

这一节给出两种解法。

> **本节属于进阶内容。** 如果前面的实验还没做通，**先跳过，做完 5.10 再回来**。中断和 DMA 各自都是一整套知识（NVIC 优先级、DMA 通道映射），本教程只讲"ADC 用它们时该怎么做"，不做全面展开。

### 5.9.1 ADC + 中断：转完了叫我

**思路**：把"等 EOC"这件事交给中断。CPU 正常干别的活，转换完成时硬件自动打断 CPU 去读值。

**初始化增加的部分：**

```c
void AD_Init_IT(void)
{
    /* ... 前面的 ADC 初始化（5.4.6）完全不变 ... */

    /* ★ 改动 1：让 ADC 在 EOC 时产生中断 */
    ADC_ITConfig(ADC1, ADC_IT_EOC, ENABLE);

    /* ★ 改动 2：配置 NVIC，允许 ADC1_2 的中断进入 CPU */
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    NVIC_InitTypeDef NVIC_InitStructure;
    NVIC_InitStructure.NVIC_IRQChannel                   = ADC1_2_IRQn;   // ADC1 和 ADC2 共用
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 2;             // 抢占优先级
    NVIC_InitStructure.NVIC_IRQChannelSubPriority        = 2;             // 响应优先级
    NVIC_InitStructure.NVIC_IRQChannelCmd                = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
}
```

**中断服务函数**（名字是**固定的**，不能自己起）：

```c
volatile uint16_t g_adc_value = 0;      // 中断里写，主循环里读
volatile uint8_t  g_adc_ready = 0;      // 标志：有新数据了

void ADC1_2_IRQHandler(void)
{
    if (ADC_GetITStatus(ADC1, ADC_IT_EOC) == SET)
    {
        g_adc_value = ADC_GetConversionValue(ADC1);     // 读值（顺便清 EOC）
        ADC_ClearITPendingBit(ADC1, ADC_IT_EOC);        // 清中断标志
        g_adc_ready = 1;                                 // 通知主循环
    }
}
```

**主循环变成这样：**

```c
while (1)
{
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);     // 触发一次，然后**不等**

    /* ── 这里可以干别的活：刷 OLED、读按键、算东西…… ── */

    if (g_adc_ready)
    {
        g_adc_ready = 0;
        printf("ADC = %d\r\n", g_adc_value);
    }
}
```

**三个必须记住的细节：**

| 细节 | 原因 |
|---|---|
| `volatile` 关键字 | 变量会被**中断**修改。不加 `volatile`，编译器可能把它优化成"读一次就缓存住"，主循环永远看到旧值（第 3 章 3.7.3 讲过 `volatile`） |
| 中断名必须叫 `ADC1_2_IRQHandler` | 这个名字定义在启动文件 `startup_stm32f10x_md.s` 里，**写错了会静默失效**（链接到一个空实现，不报错） |
| 中断里清标志 | `ADC_GetConversionValue()` 读 DR 会清 EOC，但**中断挂起位**还得单独清，否则会重复进中断 |

> **★ 中断函数里不要做慢操作。** 不要在里面 `printf`、不要 `Delay_ms`。中断应当"**快进快出**"——只把数据存下来、置个标志，剩下的交给主循环。

### 5.9.2 ADC + DMA：CPU 完全不参与

中断虽然解放了 CPU 的等待，但**每次转换还是要打断 CPU 一次**。三路扫描、每秒几十万次转换的场合，中断开销本身就受不了了。

**DMA（Direct Memory Access，直接存储器存取）** 是终极方案：**让硬件自己把数据从 ADC 搬到内存，CPU 一次都不用被打断。**

> **DMA 的本质是一个"搬运工协处理器"。** 你告诉它：从哪搬（源地址）、搬到哪（目的地址）、搬多少（长度）、怎么搬（递增不递增、循环不循环），然后它就自己干活了。

**核心代码**（三路扫描 + DMA，一次把 PA7 / PB0 / PB1 全采回来）：

> **关于代码完整性**：下面这段只列出**与 5.5 不同的部分**——`Delay_ms()`、`USART1_Init()`、`fputc()` 这三样请直接照抄 5.5 节。**5.10 节末尾有一份把三者全部合在一起的、可直接编译的完整程序**，如果这里看不明白，跳到 5.10 看全貌。

```c
#include "stm32f10x.h"
#include <stdio.h>

#define ADC_CH_COUNT    3

/* ★ 这个数组会一直被 DMA 自动更新，主循环直接读就行 */
static uint16_t g_adc_buf[ADC_CH_COUNT];
/* 下标含义： [0]=PA7 热敏   [1]=PB0 光敏   [2]=PB1 电位器   */

void AD_Init_DMA(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    ADC_InitTypeDef  ADC_InitStructure;
    DMA_InitTypeDef  DMA_InitStructure;

    /* ---------- ① 开时钟：注意 DMA1 挂的是 AHB，不是 APB ---------- */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 | RCC_APB2Periph_GPIOA |
                           RCC_APB2Periph_GPIOB, ENABLE);
    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);      // ★ AHB 版本

    RCC_ADCCLKConfig(RCC_PCLK2_Div6);

    /* ---------- ② 三个引脚都配成模拟输入 ---------- */
    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_7;               // PA7
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_0 | GPIO_Pin_1;  // PB0 / PB1
    GPIO_Init(GPIOB, &GPIO_InitStructure);

    /* ---------- ③ 配置 DMA1 通道 1（这个通道号是硬件定死的） ---------- */
    DMA_DeInit(DMA1_Channel1);

    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&ADC1->DR;   // 从 ADC 数据寄存器搬
    DMA_InitStructure.DMA_MemoryBaseAddr     = (uint32_t)g_adc_buf;   // 搬到我们的数组
    DMA_InitStructure.DMA_DIR                = DMA_DIR_PeripheralSRC; // 方向：外设 → 内存
    DMA_InitStructure.DMA_BufferSize         = ADC_CH_COUNT;          // 一共搬 3 个
    DMA_InitStructure.DMA_PeripheralInc      = DMA_PeripheralInc_Disable;  // 源地址不变
    DMA_InitStructure.DMA_MemoryInc          = DMA_MemoryInc_Enable;       // 目的地址递增
    DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;  // 16 位
    DMA_InitStructure.DMA_MemoryDataSize     = DMA_MemoryDataSize_HalfWord;      // 16 位
    DMA_InitStructure.DMA_Mode               = DMA_Mode_Circular;     // 循环：搬完自动从头来
    DMA_InitStructure.DMA_Priority           = DMA_Priority_Medium;
    DMA_InitStructure.DMA_M2M                = DMA_M2M_Disable;       // 不是内存到内存
    DMA_Init(DMA1_Channel1, &DMA_InitStructure);

    DMA_Cmd(DMA1_Channel1, ENABLE);

    /* ---------- ④ ADC 配成"连续 + 扫描" ---------- */
    ADC_InitStructure.ADC_Mode               = ADC_Mode_Independent;
    ADC_InitStructure.ADC_ScanConvMode       = ENABLE;      // ★ 扫描：转完一个转下一个
    ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;      // ★ 连续：转完一轮接着来
    ADC_InitStructure.ADC_ExternalTrigConv   = ADC_ExternalTrigConv_None;
    ADC_InitStructure.ADC_DataAlign          = ADC_DataAlign_Right;
    ADC_InitStructure.ADC_NbrOfChannel       = ADC_CH_COUNT;
    ADC_Init(ADC1, &ADC_InitStructure);

    /* ---------- ⑤ 规则组的三个位置：通道 7 / 8 / 9 ---------- */
    ADC_RegularChannelConfig(ADC1, ADC_Channel_7, 1, ADC_SampleTime_55Cycles5);
    ADC_RegularChannelConfig(ADC1, ADC_Channel_8, 2, ADC_SampleTime_55Cycles5);
    ADC_RegularChannelConfig(ADC1, ADC_Channel_9, 3, ADC_SampleTime_55Cycles5);

    /* ---------- ⑥ 让 ADC 的每次转换都产生 DMA 请求 ---------- */
    ADC_DMACmd(ADC1, ENABLE);                               // ★ 关键的一行

    /* ---------- ⑦ 使能 + 校准 ---------- */
    ADC_Cmd(ADC1, ENABLE);

    ADC_ResetCalibration(ADC1);
    while (ADC_GetResetCalibrationStatus(ADC1) == SET);
    ADC_StartCalibration(ADC1);
    while (ADC_GetCalibrationStatus(ADC1) == SET);

    /* ---------- ⑧ 连续模式下只需触发这一次，之后永不停机 ---------- */
    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
}
```

**主循环里什么都不用等：**

```c
int main(void)
{
    USART1_Init(9600);
    AD_Init_DMA();

    while (1)
    {
        /* ★ 直接读数组——DMA 已经悄悄把最新的值放进去了 */
        printf("Therm %4d | Light %4d | Pot %4d\r\n",
               g_adc_buf[0], g_adc_buf[1], g_adc_buf[2]);

        Delay_ms(200);
    }
}
```

**★ DMA 版的关键变化，就三处：**

| 变化 | 内容 |
|---|---|
| 扫描 + 连续都开 | `ScanConvMode = ENABLE`、`ContinuousConvMode = ENABLE` |
| 规则组配了 3 个位置 | 通道 7 排第 1、通道 8 排第 2、通道 9 排第 3 |
| **`ADC_DMACmd()`** | 让 ADC 每次转换都发 DMA 请求 |

**读值方式也变了**：不再是 `AD_GetValue()`，而是**直接读 `g_adc_buf[]` 数组**。

> **★ `g_adc_buf` 的下标顺序，就是 `ADC_RegularChannelConfig` 里填的"序列位置"。**
>
> 这就是为什么第 3 个参数要填 1、2、3——**它决定了数据搬进数组的顺序**。填错了，数组下标和通道就对不上（比如你以为 `[0]` 是热敏，实际是电位器）。

> **★ 数组要不要加 `volatile`？** 严格来说加更安全（DMA 会在后台改它，编译器不知道）。Keil 在 `-O0` 下不加也能跑；**开了优化就应该加**。本教程为了代码简洁没加，但你要知道有这回事。

> **★ DMA 通道号是硬件固定的。** F103 上 **ADC1 只能走 DMA1 通道 1**，改不了。查数据手册的"DMA 请求映射"表可以确认。

---

## 5.10 综合实战：简易电压表

把本章学的东西攒成一个有用的东西：**一个能同时显示三路电压和温度的简易电压表。**

用到的全部技术：ADC 扫描、DMA 搬运、电压换算、温度换算、多次平均、串口显示。

```c
#include "stm32f10x.h"
#include <stdio.h>
#include <math.h>

#define ADC_CH_COUNT    3
#define VREF_MV         3300.0f         // ★ 建议改成万用表实测的 VDDA 值（5.8.2）

static uint16_t g_adc_buf[ADC_CH_COUNT];
/* [0]=PA7 热敏   [1]=PB0 光敏   [2]=PB1 电位器 */

/* ========== 延时（同第 4 章 4.5） ========== */
void Delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 12000; j++);
}

/* ========== 串口（同第 4 章 4.9.2） ========== */
void USART1_Init(uint32_t bound)
{
    GPIO_InitTypeDef  GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_USART1, ENABLE);

    GPIO_InitStructure.GPIO_Pin   = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode  = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    USART_InitStructure.USART_BaudRate            = bound;
    USART_InitStructure.USART_WordLength          = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits            = USART_StopBits_1;
    USART_InitStructure.USART_Parity              = USART_Parity_No;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_InitStructure.USART_Mode                = USART_Mode_Tx | USART_Mode_Rx;
    USART_Init(USART1, &USART_InitStructure);

    USART_Cmd(USART1, ENABLE);
}

int fputc(int ch, FILE *f)
{
    USART_SendData(USART1, (uint8_t)ch);
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    return ch;
}

/* ========== ADC + DMA 初始化（同 5.9.2，原样搬过来） ========== */
void AD_Init_DMA(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    ADC_InitTypeDef  ADC_InitStructure;
    DMA_InitTypeDef  DMA_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1 | RCC_APB2Periph_GPIOA |
                           RCC_APB2Periph_GPIOB, ENABLE);
    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);

    RCC_ADCCLKConfig(RCC_PCLK2_Div6);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_7;               // PA7 热敏
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_0 | GPIO_Pin_1;  // PB0 光敏 / PB1 电位器
    GPIO_Init(GPIOB, &GPIO_InitStructure);

    DMA_DeInit(DMA1_Channel1);
    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&ADC1->DR;
    DMA_InitStructure.DMA_MemoryBaseAddr     = (uint32_t)g_adc_buf;
    DMA_InitStructure.DMA_DIR                = DMA_DIR_PeripheralSRC;
    DMA_InitStructure.DMA_BufferSize         = ADC_CH_COUNT;
    DMA_InitStructure.DMA_PeripheralInc      = DMA_PeripheralInc_Disable;
    DMA_InitStructure.DMA_MemoryInc          = DMA_MemoryInc_Enable;
    DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;
    DMA_InitStructure.DMA_MemoryDataSize     = DMA_MemoryDataSize_HalfWord;
    DMA_InitStructure.DMA_Mode               = DMA_Mode_Circular;
    DMA_InitStructure.DMA_Priority           = DMA_Priority_Medium;
    DMA_InitStructure.DMA_M2M                = DMA_M2M_Disable;
    DMA_Init(DMA1_Channel1, &DMA_InitStructure);
    DMA_Cmd(DMA1_Channel1, ENABLE);

    ADC_InitStructure.ADC_Mode               = ADC_Mode_Independent;
    ADC_InitStructure.ADC_ScanConvMode       = ENABLE;      // 扫描
    ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;      // 连续
    ADC_InitStructure.ADC_ExternalTrigConv   = ADC_ExternalTrigConv_None;
    ADC_InitStructure.ADC_DataAlign          = ADC_DataAlign_Right;
    ADC_InitStructure.ADC_NbrOfChannel       = ADC_CH_COUNT;
    ADC_Init(ADC1, &ADC_InitStructure);

    /* 序列位置 1/2/3 决定了数据搬进 g_adc_buf 的下标顺序 */
    ADC_RegularChannelConfig(ADC1, ADC_Channel_7, 1, ADC_SampleTime_55Cycles5);  // → [0]
    ADC_RegularChannelConfig(ADC1, ADC_Channel_8, 2, ADC_SampleTime_55Cycles5);  // → [1]
    ADC_RegularChannelConfig(ADC1, ADC_Channel_9, 3, ADC_SampleTime_55Cycles5);  // → [2]

    ADC_DMACmd(ADC1, ENABLE);

    ADC_Cmd(ADC1, ENABLE);

    ADC_ResetCalibration(ADC1);
    while (ADC_GetResetCalibrationStatus(ADC1) == SET);
    ADC_StartCalibration(ADC1);
    while (ADC_GetCalibrationStatus(ADC1) == SET);

    ADC_SoftwareStartConvCmd(ADC1, ENABLE);     // 连续模式，只需触发一次
}

/* ---------- 阻值换算：传感器在高边，R传感 = R固定 × (VREF/V − 1) ---------- */
static float ADC_ToResistance(uint16_t adc, float r_fixed)
{
    float v_mv = adc * VREF_MV / 4095.0f;

    if (v_mv < 1.0f || v_mv > VREF_MV - 1.0f)
        return -1.0f;                       // 读数超出合理范围

    return r_fixed * (VREF_MV / v_mv - 1.0f);
}

/* ---------- 热敏阻值 → 温度（B 参数方程） ---------- */
static float Resistance_ToTemperature(float r_ntc)
{
    float t_kelvin;

    if (r_ntc < 1.0f)
        return -999.0f;

    t_kelvin = 1.0f / (1.0f / 298.15f + logf(r_ntc / 10000.0f) / 3950.0f);
    return t_kelvin - 273.15f;
}

int main(void)
{
    float v_therm, v_light, v_pot;          // 三路电压（V）
    float r_therm, r_light;                 // 两个传感器的阻值（Ω）
    float temp;                             // 温度（℃）

    USART1_Init(9600);
    AD_Init_DMA();

    printf("\r\n===== Simple Voltmeter =====\r\n");
    printf("VREF setting = %.0f mV\r\n\r\n", VREF_MV);
    printf("  Therm(PA7)   Light(PB0)   Pot(PB1)    Temp\r\n");
    printf("  ----------   ----------   ---------   ------\r\n");

    while (1)
    {
        /* 三路电压换算（DMA 已经填好 g_adc_buf） */
        v_therm = g_adc_buf[0] * VREF_MV / 4095.0f / 1000.0f;
        v_light = g_adc_buf[1] * VREF_MV / 4095.0f / 1000.0f;
        v_pot   = g_adc_buf[2] * VREF_MV / 4095.0f / 1000.0f;

        /* 热敏 → 阻值 → 温度 */
        r_therm = ADC_ToResistance(g_adc_buf[0], 10000.0f);     // R48 = 10K
        temp    = Resistance_ToTemperature(r_therm);

        /* 光敏 → 阻值 */
        r_light = ADC_ToResistance(g_adc_buf[1], 100000.0f);    // R49 = 100K

        printf("  %6.3f V     %6.3f V     %6.3f V    ",
               v_therm, v_light, v_pot);

        if (temp > -900.0f)
            printf("%5.1f C\r\n", temp);
        else
            printf("  ERR\r\n");

        /* 顺带把阻值也打出来——传感器状态一目了然 */
        printf("  (R_therm = %7.0f ohm, R_light = %8.0f ohm)\r\n",
               r_therm, r_light);

        Delay_ms(500);
    }
}
```

**✅ 串口输出：**

```
===== Simple Voltmeter =====
VREF setting = 3300 mV

  Therm(PA7)   Light(PB0)   Pot(PB1)    Temp
  ----------   ----------   ---------   ------
   1.650 V      2.742 V      1.234 V     25.0 C
  (R_therm =   10012 ohm, R_light =    20328 ohm)
   1.648 V      2.740 V      1.236 V     25.1 C
  (R_therm =   10061 ohm, R_light =    20414 ohm)
```

**动手验证清单**（把三路都试一遍，这个综合实验才算做完整）：

| 操作 | 应该看到 |
|---|---|
| 拧电位器 | 电位器那列电压变化，**其余两列不动** |
| 手遮光敏 | 光敏电压**下降**、阻值**上升** |
| 手指捏热敏 | 温度**上升**、热敏阻值**下降**、热敏电压**上升** |
| 用手电筒照光敏 | 光敏电压**上升** |

> **★ 最后一条是本章的核心**：**三路互不干扰，同时更新，CPU 全程没等过一次。** 这就是 DMA 的价值。

**可以继续做的扩展**（如果你想再练手）：

| 扩展 | 用到的知识 |
|---|---|
| 加个按键切换显示模式 | 第 4 章 4.6 的按键锁写法 |
| 数值异常时用蜂鸣器报警 | 第 4 章 4.7（注意**低电平响**） |
| 改成 OLED 显示，脱机运行 | 第 4 章 4.9.3（注意 **SCL=PB9 / SDA=PB8**） |
| 加上中值滤波 | 5.8.5 |

---

## 5.11 本章小结

1. **ADC 把连续的电压变成 0~4095 的整数。** 12 位的意思是最多"猜 12 次"——逐次逼近，每次都把范围砍一半。
2. **换算公式**：`电压 = ADC值 × 3.3 / 4095`，分辨率约 0.8 mV。
3. **★ 板载 ADC1/2/3 是导线名，外设 ADC1/ADC2 是芯片模块**，两者无关。**代码里出现的 `ADC1` 永远是外设。**
4. **本板三路**：PA7（通道 7，热敏）、PB0（通道 8，光敏）、PB1（通道 9，电位器）。
5. **传感器都接在分压高边**，所以**物理量越强 → 电压越高**。阻值反解公式：`R传感 = R固定 × (VREF / V节点 − 1)`。
6. **★ ADC 引脚只能配 `GPIO_Mode_AIN`**。上拉/下拉会并联 40 kΩ（读数偏高/偏低），浮空虽无电阻但数字通路未关、读数漂移；**只有 `AIN` 会把数字通路整个关掉**。
7. **ADC 也在 APB2**，和 GPIO 用同一个开时钟函数（**第 6 章的 DAC 在 APB1，是唯一的例外**）。
8. **ADCCLK 不能超 14 MHz**，用 `RCC_PCLK2_Div6` 得 12 MHz。
9. **上电校准不能省，两个 `while` 等待不能省。**
10. **采样时间要匹配信号源内阻**，本板统一用 55.5 周期。
11. **提高精度三招**：校准 VDDA 实际值、多次平均、必要时加中值滤波。
12. **中断**解放 CPU 的等待，**DMA** 让 CPU 完全不参与。中断服务函数的名字是固定的（`ADC1_2_IRQHandler`），中断里改的变量要加 `volatile`。

**本章勘误**（参考资料中的问题，教程中已修正）：

| # | 出处 | 问题 |
|---|---|---|
| 1 | 学习笔记 ADC 引脚表 | 表头写「外设」，内容填 `ADC1/ADC2/ADC3`——实为**板载网络名**。已按 5.2.2 的约定区分 |
| 2 | 学习笔记 ADC 代码注释 | `ADC_RegularChannelConfig(ADC1, ADC_Channel_8, ...)` 的注释写「配置为通道 0」，与实际参数不符，属笔误 |

---

**下一章**：[第 6 章 DAC](06-DAC.md) —— 反过来，让数字变成电压。

---

> **图示来源**：`ADC_逐次逼近原理.jpg`、`ADC_框图.png`、`ADC_触发控制.png`、`ADC_数据右对齐.png`、`ADC_数据左对齐.png` 取自江协科技《STM32入门教程》课件第 87、88、95、96 页，仅用于教学；`板载三路模拟源原理图.png` 取自本学习板配套原理图。引脚、通道号、分压参数一律以原理图和 [附录C](附录C-板载引脚总表.md) 为准。

---

[返回目录](README.md) · [上一章：GPIO](04-GPIO.md)
