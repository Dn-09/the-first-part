# 第 6 章 DAC（归档）

> **归档文件**：STM32F103C8T6 **没有内置 DAC**。本文件基于错误前提编写，仅作为素材保留，不进入新教材。
> 可用内容已改写进《[STM32入门培训教材](STM32入门培训教材.md)》第 5 章“DAC：用 PWM + RC 实现模拟输出”。

第 5 章把**模拟电压变成了数字**。这一章反过来：**把数字变成模拟电压**。

**DAC（Digital-to-Analog Converter，数字-模拟转换器）** 让单片机第一次拥有了"*说出*一个具体电压"的能力——不是只有高和低，而是**任意一个 0 ~ 3.3 V 之间的值**。

本章结构：

| 小节 | 内容 |
|---|---|
| 6.1 | DAC 是什么：与 ADC 互为逆过程 |
| 6.2 | F103C8T6 的 DAC 资源 |
| 6.3 | ⚠️ **本板硬件限制专节**：PA4/PA5 挂了 LED ★ 全书核心冲突 |
| 6.4 | 标准库 DAC 编程流程 ★ 含全书唯一一个 APB1 |
| 6.5 | 实验 9：DAC 输出直流电压（LED 亮度渐变，零改动） |
| 6.6 | 实验 10：万用表定量测量 |
| 6.7 | 实验 11：定时器 + DMA 输出正弦波 |
| 6.8 | 实验 12（可选）：DAC → ADC 回环验证 |
| 6.9 | ADC 与 DAC 对比总结 |

> **本章的特殊性**：全部参考资料里，DAC **只作为名词出现过两次**——一次是外设总表，一次是"TIM6/7 主模式触发 DAC"这么一句。**没有任何讲解、没有任何示例代码、没有一张图。**
>
> **本章 100% 原创。** 而它要处理的那个硬件冲突（6.3），也是全书唯一一个"电路本身就该讲一节课"的地方。

---

## 6.1 DAC 是什么

### 6.1.1 从第 5 章接过来

第 5 章 5.1.2 讲逐次逼近 ADC 时，出现过这样一张图：

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
```

**当时那个"内部 DAC"，就是这一章要讲的东西。**

ADC 和 DAC 是一对**互逆**的转换：

```
        模拟量                              数字量
      (连续电压)                          (离散整数)

        0 ~ 3.3 V   ──────ADC──────►   0 ~ 4095
        0 ~ 3.3 V   ◄─────DAC──────   0 ~ 4095
```

| | ADC | DAC |
|---|---|---|
| 方向 | 模拟 → 数字 | **数字 → 模拟** |
| 输入 | 引脚上的电压 | 你写进寄存器的整数 |
| 输出 | 0~4095 的整数 | 引脚上的电压 |
| 用在哪 | 采集温度、光照、电压 | 输出波形、控制电压、音频 |

**一句话**：**ADC 是"感知世界"，DAC 是"表达意图"。**

### 6.1.2 DAC 怎么把数字变成电压

最常见的方法是**电阻网络**：用一组按二进制权重排列的电阻（或 R-2R 梯形网络），把 12 个二进制位各自变成一个电流，再求和。

```
   D11 D10 D9  …  D1  D0        ← 12 个二进制位
    │   │   │      │   │
   ┌┴───┴───┴──────┴───┴┐
   │   R-2R 电阻网络      │       ← 每一位贡献一个成比例的电流
   └────────┬───────────┘
            │  总电流 ∝ 数字值
            ▼
      ┌───────────┐
      │  输出缓冲  │  ← 把电流变成稳定的电压，并降低输出阻抗
      └─────┬─────┘
            ▼
         引脚输出
```

**关键在最后那个"输出缓冲"**——它决定了 DAC 能不能带得动负载。6.2.3 会细说，它是本章很多坑的根源。

> **为什么 DAC 比 ADC 快得多？**
>
> ADC 要"猜" 12 次（逐次逼近），而 DAC 是**组合逻辑**——12 个位同时作用在电阻网络上，**稳定时间只有几百纳秒**。这就是为什么高精度 DAC 能做到几 MSPS，而 ADC 很难。
>
> 换算到本章：ADC 最快约 1 µs 转一次（5.4.4 算过 5.67 µs）；DAC 换个值只要约 0.4 µs。**DAC 天生就适合输出波形。**

### 6.1.3 12 位、0~4095、输出 0~VREF

和 ADC 完全对称：

| 项目 | 数值 |
|---|---|
| 位数 | **12 位** |
| 输入数字范围 | **0 ~ 4095** |
| 输出电压范围 | **0 ~ VREF**（本板 VREF = VDDA = 3.3 V） |
| 分辨率 | 3.3 V ÷ 4095 ≈ **0.806 mV** |

**换算公式**（和 ADC 方向相反，形式一样）：

```
        输出电压 = 数字值 × 3.3 / 4095
        数字值   = 目标电压 × 4095 / 3.3
```

| 数字值 | 理论输出 |
|---|---|
| 0 | 0 V |
| 1024 | 0.825 V |
| 2048 | 1.650 V |
| 3072 | 2.475 V |
| 4095 | 3.3 V |

> ⚠️ **0 和 4095 这两个端点，实际输出到不了。** 因为输出缓冲是运放，运放的输出**天然到不了电源轨**（rail-to-rail 运放除外）。F103 的 DAC 缓冲启用后，**实际输出范围约为 0.2 V ~ 3.1 V**。
>
> 这个"端点上不去"的现象，6.6 定量测量时会亲眼看到。**不是你的代码错了，是模拟电路的天然限制。**

### 6.1.4 DAC 和 PWM 有什么区别

你可能会想：**用 PWM 加个 RC 滤波，不也能"输出"任意电压吗？** 很多教程确实这么做。三者的区别：

| | PWM + RC 滤波 | 真正的 DAC |
|---|---|---|
| 原理 | 用占空比表示平均值，靠滤波平滑 | 直接产生目标电压 |
| 输出纹波 | **有**（滤波不干净就一直在抖） | 极小 |
| 建立速度 | 慢（受 RC 时间常数限制） | **快**（几百纳秒） |
| 带载能力 | 弱 | 由缓冲器决定 |
| 需要的外设 | 定时器（人人都有） | **DAC（很多芯片没有）** |

| 场合 | 选谁 |
|---|---|
| 控制 LED 亮度、调电机转速 | **PWM 更合适**（本来就是给功率器件用的开关信号） |
| 输出精确直流基准、生成波形、音频 | **DAC** |

> **★ 本章 6.5 会看到一句话：**「用 DAC 让 LED 亮度连续变化」。这句话用 PWM 也能做到，而且更常见。
>
> **我们这里用 DAC 做，是为了让你亲眼看见"模拟量"是什么样子**——它的教学价值大于实用价值。真正要做呼吸灯，PWM 是更好的选择。

---

## 6.2 F103C8T6 的 DAC 资源

### 6.2.1 资源一览

| 项目 | STM32F103C8T6 |
|---|---|
| DAC 外设个数 | **只有 DAC1**（没有 DAC2） |
| 通道数 | **2 路**：通道 1、通道 2 |
| 分辨率 | 12 位 |
| 参考电压 | VREF+ = VDDA = **3.3 V** |
| 触发源 | 软件、定时器 TRGO 等 |
| 附加功能 | **噪声波发生器、三角波发生器**（硬件生成，不用 CPU） |

> **"没有 DAC2"这件事很重要**：F103 系列里，只有**大容量型号**（Flash ≥ 256 KB，型号后缀 `xC`/`xD`/`xE`）才有双 DAC。C8T6 是**中容量**（64 KB），**只有 DAC1**。
>
> 也就是说——**全芯片能输出模拟电压的引脚，只有两根。**

### 6.2.2 引脚、通道、DMA 映射

```
   DAC1
    ├── 通道 1  →  DAC_OUT1  →  PA4
    └── 通道 2  →  DAC_OUT2  →  PA5
```

**F103 的 DAC 没有重映射、没有替代引脚。PA4 和 PA5，就是全部。**

**DMA 映射**（6.7 会用到）：

| 外设 | DMA 通道 |
|---|---|
| DAC 通道 1 | **DMA1 通道 3** |
| DAC 通道 2 | **DMA1 通道 4** |

> 对照第 5 章 5.9.2：ADC1 用的是 **DMA1 通道 1**。这几个通道号都是**硬件定死的**，改不了，只能查数据手册。

### 6.2.3 输出缓冲：一个必须知道的开关

DAC 的 `DAC_InitTypeDef` 里有一个成员叫 **`DAC_OutputBuffer`**，它控制片内那个输出缓冲器开不开：

| | `DAC_OutputBuffer_Enable` | `DAC_OutputBuffer_Disable` |
|---|---|---|
| 输出阻抗 | **低**（能带负载） | 高（只能接高阻负载） |
| 输出范围 | **约 0.2 V ~ VDDA−0.2 V** | 可以更接近 0 V 和 VDDA |
| 带载能力 | 较强 | **弱**，接个电阻就被拉偏 |
| 用在哪 | 一般场合 | 要输出接近 0 V 的小电压 |

**本教程默认用 `Enable`**，理由：PA4/PA5 上挂着 LED（6.3），我们需要缓冲器去"顶住"这个负载。

> **★ 记住这条因果关系**：**要带负载 → 开缓冲 → 端点到不了。**
>
> 6.6 定量测量时你会看到"设定 0 却量到 0.2 V 左右"，原因就在这里。

### 6.2.4 硬件波形发生器：DAC 自带的彩蛋

F103 的 DAC 有个别处不常见的能力——**硬件直接产生噪声波和三角波**，不占 CPU：

```c
ADC_InitStructure.DAC_WaveGeneration = DAC_WaveGeneration_Triangle;   // 三角波
```

配合 `DAC_LFSRUnmask_TriangleAmplitude` 还能设幅度（从 1 LSB 到 4095）。

**本教程不用它**——因为**用自己的 DMA 输出波形更能讲清原理**（6.7），而这个硬件发生器只支持三角波和噪声，用处有限。

> 知道有这回事即可。**面试时提一句能加分。**

---

## 6.3 ⚠️ 本板硬件限制专节：PA4/PA5 挂了 LED

**这一节是全书唯一一个"硬件设计本身构成一节课"的地方，请完整读完。**

### 6.3.1 冲突是怎么来的

第 4 章 4.5 节的避坑提示里，留下过这么一句话：

> **★ 关于 PA4 / PA5——记住这两脚不简单**
>
> PA4 = `DAC_OUT1`，PA5 = `DAC_OUT2`，是第 6 章 DAC 的两个输出通道；F103 的 DAC 只有这两根引脚，没有替代。
>
> 也就是说，**这两颗 LED 和 DAC 抢同一对引脚。**

**现在到了兑现的时候。**

回顾本板 LED 电路（第 4 章 4.4 ①）：

```
   3V3 ──▶|── LED9  ── R30 1k ── PA3
   3V3 ──▶|── LED10 ── R31 1k ── PA4     ← DAC_OUT1  ⚠️
   3V3 ──▶|── LED11 ── R32 1k ── PA5     ← DAC_OUT2  ⚠️
```

**设计电路的人大概没打算用 DAC**——把 LED 直接挂在了仅有的两个模拟输出脚上。

于是形成了一个尴尬的局面：

```
   F103C8T6 能输出模拟电压的引脚：只有 PA4、PA5
   本板上 PA4、PA5 上挂了什么：  LED10、LED11

   → 想用 DAC，就得和 LED 共存
```

### 6.3.2 影响：LED 什么时候会"捣乱"

**关键是要看清：LED 不是一直导通的。**

回顾第 4 章 4.4 ③ 的那张表：

| PA4 输出 | 电流方向 | LED |
|---|---|---|
| **低（0 V）** | 3.3 V → LED → R31 → PA4（被拉到 0 V） | **亮** |
| 高（3.3 V） | 两端都是 3.3 V，没有压差 | 灭 |

**现在把 PA4 当成 DAC 输出**，情况就复杂了：

```
   3V3 ──▶|── LED10 ── R31 1k ──┬── PA4 (DAC_OUT1)
                                 │
                              DAC 输出缓冲
```

LED 是**二极管**，只有**正向电压超过它的导通压降**才会通电。普通红/绿 LED 的压降约 **1.8 ~ 2.1 V**。

**所以 LED 的导通条件是：**

```
   3.3 V − V_PA4  >  Vf(LED)

   即：V_PA4  <  3.3 − Vf  ≈  1.3 ~ 1.6 V
```

**画成图就是这样：**

```
   V_PA4 (V)
     3.3 ┤━━━━━━━━━━━━━━━━━━━━━━━━┓
         │   LED 截止区（不导通）    ┃
     1.6 ┤━━━━━━━━━━━━━━━━━━━━━━━━┫ ← 临界点（约值，看 LED 颜色）
         │   LED 导通区（灌电流）    ┃
       0 ┤━━━━━━━━━━━━━━━━━━━━━━━━┛
```

**两个区域的行为完全不同：**

| 区间 | LED 状态 | 对 DAC 输出的影响 |
|---|---|---|
| **1.6 V ~ 3.3 V** | **截止**，不导通 | **几乎没有影响** ✅ |
| 0 ~ 1.6 V | **导通**，向引脚灌入电流（最大约 1.4 mA） | **输出电压被拉偏，且非线性** ⚠️ |

**为什么会"拉偏"？** DAC 的输出缓冲器有**输出阻抗**。当 LED 支路灌入电流时，这个电流流过输出阻抗产生压降，把实际输出电压**抬高**（因为电流是灌进引脚的，方向是"往外推"）。

**而且是非线性的**——因为 LED 是非线性器件：电压越低，正向压差越大，电流增长得越快。

> **★ 一个直观的比喻**：DAC 像一个"定压电源"，但它的输出阻抗不是 0。你把它接到一个会"顶回来"的负载上，它就得花力气去顶，一顶就偏。

### 6.3.3 对策：把实验分成两类

既然 LED 只在**低电压区**捣乱，那策略就很清楚了：

| 实验类型 | 电压范围 | 是否受影响 | 处理方式 |
|---|---|---|---|
| **定性观察**（看 LED 亮度变化） | 全范围 | —— | **零改动**，而且**故意利用 LED 看效果** |
| **定量测量 · 上半段** | **1.6 V ~ 3.3 V** | 不影响 | **零改动**，直接测 ✅ |
| **定量测量 · 下半段** | 0 ~ 1.6 V | **有影响** | 需要断开，或接受偏差 |

**★ 这是个好消息：本章的定量实验大部分不需要动硬件。**

**怎么知道自己板上的临界点在哪？** 很简单：

> **实测方法**：让 DAC 从 4095 慢慢往下扫，同时观察 LED10。
>
> **LED 开始发光的那一刻，对应的电压就是临界点。** 记下这个值，比推算靠谱——因为 LED 的实际压降取决于颜色和批次。

**顺便，这个观察本身就是个很好的电路课**：你会看到"LED 从完全不亮，到某一刻突然微亮，然后迅速变亮"——这正是二极管指数型伏安特性的直观体现。

### 6.3.4 三种处理方式

| 方式 | 做法 | 代价 | 什么时候用 |
|---|---|---|---|
| **A. 不变** | 接受 1.6 V 以下的偏差 | 低电压区读数偏大 | **本章 6.5、6.6 上半段、6.7** |
| **B. 断开 LED 支路** | 把 R30/R31/R32 拆掉（或挑开一端） | **需要动烙铁** | 6.6 下半段、6.8 |
| **C. 改用外部负载** | 不接负载，只测空载输出 | 无法验证带载能力 | 任何需要精确读数的场合 |

> **★ 本教程的处理原则：默认走方式 A，把方式 B 标成"可选"。**
>
> 因为绝大多数教学目标（理解模拟量、理解 DAC 流程、看波形）**在 1.6 V 以上就能完整达成**。**不为了让读者动烙铁而设计实验。**

> **⚠️ 关于 R30/R31/R32 的具体封装**：本教程未拿到这几颗电阻的封装信息（0805？0603？）。**断开方法以你板子上的实物为准**：
>
> - 如果是**贴片电阻**：用烙铁两端加热取下，或**用刀片从中间划断**（更省事，但不可逆）
> - 如果板上**留了跳线/焊盘**：直接断开跳线即可
>
> **无论用哪种方式，断开前请先确认你确实需要这个精度。** 6.6 会告诉你什么情况下需要。

---

## 6.4 标准库 DAC 编程流程

### 6.4.1 ★ 开时钟：全书唯一一个 APB1

**这里是第 1 章 1.2 埋下、第 4 章两次预告、第 5 章再次预告的那个坑。现在正式兑现。**

```c
RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);
```

**注意函数名里的 `APB1`，不是 `APB2`。**

把全书出现过的外设总起来看：

| 总线 | 频率 | 本教程涉及的外设 | 开时钟函数 |
|---|---|---|---|
| **APB2** | 72 MHz | **GPIO**、**ADC**、USART1 | `RCC_APB2PeriphClockCmd()` |
| **APB1** | 36 MHz | **DAC**、TIM2~7 | `RCC_APB1PeriphClockCmd()` |

**你看过的每一个开时钟语句，用的都是 `APB2`。** 习惯成自然之后，写 DAC 时会**顺手**打成：

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_DAC, ENABLE);   // ❌ 编译不通过
```

或者更糟——**编译器没报错**，但 DAC 完全没反应：

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);   // ✅ 正确
```

> **★ 这个坑为什么值得反复预告三次？**
>
> 因为它**不报错**。`RCC_APB2Periph_DAC` 这个宏**根本不存在**，所以写成 APB2 会编译失败——这反而是好事。
>
> **真正危险的是"时钟压根没开"**：程序编译下载都正常，`DAC_SetChannel1Data()` 也执行了，但**引脚上一点电压都没有**。你会去查 `DAC_Init` 的每个参数、查引脚模式、查缓冲开关……**唯独想不起来最前面那一行。**
>
> **排查口诀（第 4 章 4.1.2 就给你的）：凡是操作某个外设，先想它挂在哪条总线上，第一行先开时钟。**

### 6.4.2 GPIO 模式：又是 `GPIO_Mode_AIN`

**DAC 引脚要配成模拟输入 `GPIO_Mode_AIN`**，不是推挽输出。

```c
GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_4;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;      // ★ 不是 Out_PP！
GPIO_Init(GPIOA, &GPIO_InitStructure);
```

**为什么一个"输出"引脚要配成"输入"模式？** 这是最容易困惑的地方。

因为**输出引脚此刻已经不是 GPIO 在控制了**——它交给了 DAC 外设。

| 配置 | 后果 |
|---|---|
| `GPIO_Mode_AIN` | **关闭整个数字通路**（第 4 章 4.2.3 讲过），引脚只听 DAC 的 ✅ |
| `GPIO_Mode_Out_PP` | GPIO 的数字输出电路**仍然挂在引脚上**，会和 DAC 争控制权，输出值不对 |

**`AIN` 在这里的作用是"把 GPIO 的数字部分彻底关掉、让开位置"**——这和第 5 章配 ADC 引脚是同一个动作，只是目的不同：

| | 为什么用 `AIN` |
|---|---|
| 第 5 章 ADC 引脚 | 让外部电压**原封不动进来** |
| 第 6 章 DAC 引脚 | 让 DAC 电压**原封不动出去** |

> **★ 和第 4 章 4.5 直接对照**：流水灯实验里，PA4/PA5 配的是 `GPIO_Mode_Out_PP`，因为那时候它们是**普通的 GPIO 输出**。
>
> **同一对引脚，两种用途，两种模式**——这正是 4.2 那张八模式表存在的意义。

### 6.4.3 `DAC_InitTypeDef` 的四个成员

```c
DAC_InitTypeDef DAC_InitStructure;
```

| 成员 | 填什么 | 说明 |
|---|---|---|
| `DAC_Trigger` | `DAC_Trigger_None` | **触发方式**。`None` = 写完数据直接输出 |
| | `DAC_Trigger_T6_TRGO` | 由 **TIM6** 触发（6.7 输出波形用） |
| | `DAC_Trigger_Software` | 软件触发，需额外调 `DAC_SoftwareTriggerCmd()` |
| `DAC_WaveGeneration` | `DAC_WaveGeneration_None` | **波形发生器**。`None` = 不用片内发生 |
| | `DAC_WaveGeneration_Triangle` | 硬件三角波（6.2.4 提过，本教程不用） |
| `DAC_LFSRUnmask_TriangleAmplitude` | `DAC_LFSRUnmask_Bit0` | 只在用硬件波形时才有效，填什么无所谓 |
| `DAC_OutputBuffer` | `DAC_OutputBuffer_Enable` | **输出缓冲**（6.2.3）。默认开 |

**本教程最常用的组合：**

```c
DAC_InitStructure.DAC_Trigger                        = DAC_Trigger_None;
DAC_InitStructure.DAC_WaveGeneration                 = DAC_WaveGeneration_None;
DAC_InitStructure.DAC_LFSRUnmask_TriangleAmplitude   = DAC_LFSRUnmask_Bit0;
DAC_InitStructure.DAC_OutputBuffer                   = DAC_OutputBuffer_Enable;
```

> **`DAC_LFSRUnmask_Bit0` 这一行看着莫名其妙，但它必须填。** 结构体成员不赋值就是随机值，虽然本例中用不到它，**但养成"每个成员都显式赋值"的习惯**（第 4 章 4.4⑥ 提醒过）。

### 6.4.4 输出一个电压

```c
DAC_Cmd(DAC_Channel_1, ENABLE);                     // 使能通道 1

DAC_SetChannel1Data(DAC_Align_12b_R, 2048);         // 写入 12 位数据（右对齐）
```

**两个函数，两步：**

| 函数 | 作用 |
|---|---|
| `DAC_Cmd(DAC_Channel_x, ENABLE)` | 打开通道（**不开的话写数据也没用**） |
| `DAC_SetChannel1Data(DAC_Align_12b_R, 值)` | 把 0~4095 写进数据寄存器 |

**`DAC_Align_12b_R` 里的 `R` 是 Right（右对齐）**，和第 5 章 5.3.2 的 ADC 数据对齐是同一个概念的镜像。DAC 还有 `DAC_Align_12b_L`（左对齐）和 `DAC_Align_8b_R`（8 位）。

**为什么用 `DAC_Trigger_None` 时不需要额外触发？**

因为数据一写进寄存器，硬件就会**自动在几个 APB1 周期后**把它送到输出。这比 `DAC_Trigger_Software` 少一行代码，本章 6.5、6.6 都用它。

> **★ 通道 2 的函数名不一样，别写错：**
>
> | 通道 | 设置数据 | 使能 |
> |---|---|---|
> | 通道 1（PA4） | `DAC_SetChannel1Data()` | `DAC_Cmd(DAC_Channel_1, ENABLE)` |
> | 通道 2（PA5） | `DAC_SetChannel2Data()` | `DAC_Cmd(DAC_Channel_2, ENABLE)` |
>
> **中间那个数字是通道号，不是 ADC 的 `ADC1` 那种外设编号。** 写成 `DAC_SetChannel2Data` 去控制 PA4，是最容易犯的笔误。

### 6.4.5 完整初始化代码

```c
#include "stm32f10x.h"

/* ---------- DAC 初始化：单通道（PA4） ---------- */
void DA_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    DAC_InitTypeDef  DAC_InitStructure;

    /* ① ★ 开时钟：DAC 在 APB1！GPIO 在 APB2 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);

    /* ② PA4 配成模拟输入（不是推挽输出！） */
    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_4;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    /* ③ 配置 DAC 通道 1 */
    DAC_InitStructure.DAC_Trigger                      = DAC_Trigger_None;
    DAC_InitStructure.DAC_WaveGeneration               = DAC_WaveGeneration_None;
    DAC_InitStructure.DAC_LFSRUnmask_TriangleAmplitude = DAC_LFSRUnmask_Bit0;
    DAC_InitStructure.DAC_OutputBuffer                 = DAC_OutputBuffer_Enable;
    DAC_Init(DAC_Channel_1, &DAC_InitStructure);

    /* ④ 使能通道 1 */
    DAC_Cmd(DAC_Channel_1, ENABLE);
}

/* ---------- 设置输出：0~4095 ---------- */
void DA_SetValue(uint16_t value)
{
    if (value > 4095) value = 4095;                     // 越界保护
    DAC_SetChannel1Data(DAC_Align_12b_R, value);
}
```

**注意 `DA_Init()` 里的两行开时钟写在一起，但用的是两个不同函数**——这个对比值得多看一眼。

---

## 6.5 实验 9：DAC 输出直流电压

**目标**：让 PA4 上的 LED10 亮度连续变化，**亲眼看到"模拟量"和"数字量"的区别**。

> **这是本章唯一一个"零硬件改动 + 现象立刻可见"的实验。** 先做它，一秒钟就能确认 DAC 是否工作。

### ① 原理图

```
                    3V3
                     │
        ┌────────────┼────────────┐
        │            ▼            │
        │          LED10          │
        │            │            │
        │         R31 1kΩ         │
        │            │            │
        │       ┌────┴────┐       │
        │       │  DAC 输出 │       │
        │       │  缓冲器   │       │
        │       └────┬────┘       │
        │            │            │
        │        DA_Init()        │
        └─────────────────────────┘
                  PA4
                (DAC_OUT1)
```

**LED10 的亮度，就是 PA4 输出电压的"可视化仪表"。** PA4 电压越低，LED 越亮。

### ② 硬件连接

**不需要接线，也不需要断开任何东西。** LED10 已经在板上了。

### ③ 原理讲解

**为什么这个实验用 LED 观察反而更好？**

因为**第 4 章的 GPIO 和这一章的 DAC，输出的是同一对引脚上的电压**。用同一颗 LED 去看，区别最直观：

| | GPIO（第 4 章 4.5 流水灯） | DAC（本实验） |
|---|---|---|
| 能输出的值 | **只有 2 个**：0 V 或 3.3 V | **4096 个**：0 ~ 3.3 V 任意值 |
| LED 表现 | **只有亮和灭**，非此即彼 | **从全灭到全亮，中间有 4000 多级过渡** |
| 本质 | 数字量 | **模拟量** |

**这就是"数字"和"模拟"的区别，用眼睛就能看见。**

**实验设计**：让 DAC 的值从 0 慢慢升到 4095，再从 4095 降回 0——LED 会**从暗到亮、再从亮到暗**平滑变化，像呼吸一样。

> ⚠️ **注意方向**：DAC 值**越大**，PA4 电压**越高**，LED **越暗**（因为 LED 是低电平点亮，第 4 章 4.4 讲过）。
>
> 所以：**DAC 值 0 → LED 最亮；DAC 值 4095 → LED 完全熄灭。**
>
> **这个方向感是反的**，一开始容易搞混。记住：**DAC 输出的是电压，LED 看的是"电压有多低"。**

### ④ 完整代码

```c
#include "stm32f10x.h"

/* ---------- 延时（同第 4 章 4.5） ---------- */
void Delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 12000; j++);
}

/* ---------- DAC 初始化 ---------- */
void DA_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    DAC_InitTypeDef  DAC_InitStructure;

    /* ★ 两个外设，两条总线，两个函数 */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);   // GPIO 在 APB2
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);     // DAC  在 APB1

    /* PA4 配成模拟输入 */
    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_4;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    DAC_InitStructure.DAC_Trigger                      = DAC_Trigger_None;
    DAC_InitStructure.DAC_WaveGeneration               = DAC_WaveGeneration_None;
    DAC_InitStructure.DAC_LFSRUnmask_TriangleAmplitude = DAC_LFSRUnmask_Bit0;
    DAC_InitStructure.DAC_OutputBuffer                 = DAC_OutputBuffer_Enable;
    DAC_Init(DAC_Channel_1, &DAC_InitStructure);

    DAC_Cmd(DAC_Channel_1, ENABLE);
}

/* ========== 主函数 ========== */
int main(void)
{
    uint16_t value;

    DA_Init();

    while (1)
    {
        /* ───── 渐亮：DAC 值从 4095 降到 0，PA4 电压降低，LED 变亮 ───── */
        for (value = 4095; value > 0; value -= 8)
        {
            DAC_SetChannel1Data(DAC_Align_12b_R, value);
            Delay_ms(2);
        }

        /* ───── 渐暗：DAC 值从 0 升到 4095，PA4 电压升高，LED 变暗 ───── */
        for (value = 0; value < 4095; value += 8)
        {
            DAC_SetChannel1Data(DAC_Align_12b_R, value);
            Delay_ms(2);
        }
    }
}
```

**关于步长 8 和延时 2 ms：**

| 参数 | 计算 | 说明 |
|---|---|---|
| 步长 8 | 4095 ÷ 8 ≈ 512 步 | 走完全程 512 步 |
| 延时 2 ms | 512 × 2 ms ≈ **1 秒** | 一个完整的呼吸周期约 **2 秒** |

**想调快调慢就改这两个数。** 步长改小（如 1）则过渡更细腻，但周期变长。

### ⑤ 现象验证

**✅ LED10 像呼吸一样平滑地由暗到亮、再由亮到暗，循环往复。**

**★ 关键观察点——请仔细看：**

| 观察 | 说明 |
|---|---|
| 亮度是**连续**变化的吗？ | **是**。中间看不出"台阶"，虽然有 512 步，但人眼分辨不出来 |
| 和 GPIO 的流水灯比，差别在哪？ | 流水灯只有"亮/灭"两个状态，**跳变**；这里**连续过渡** |
| 最暗时是完全熄灭吗？ | **是**。DAC 值到 4095 时 PA4 ≈ 3.1 V，LED 完全截止 |

**这个"平滑"就是模拟量最直观的样子。**

> **★ 一个可以顺手做的小验证**：把 `Delay_ms(2)` 改成 `Delay_ms(0)`（或删掉延时）。因为 `Delay_ms(0)` 也会执行外层循环判断，程序仍然跑得很快——**LED 会变成一种"频闪"的观感，而不是平滑呼吸**。
>
> 因为 DAC 更新得太快，你看到的不再是每个中间值的亮度，而是闪烁。**这反过来说明：平滑的观感来自于"有序地、慢慢地经过每一个值"。**

### ⑥ 避坑提示

> ⚠️ **★ LED 完全不亮或一直全亮？** 按这个顺序查：
>
> | 顺序 | 检查项 |
> |---|---|
> | 1 | **`RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE)` 写了吗？用的是 APB1 吗？** ← 本实验第一坑 |
> | 2 | PA4 配的是 `GPIO_Mode_AIN` 吗？（配成 `Out_PP` 会和 DAC 抢引脚） |
> | 3 | `DAC_Cmd(DAC_Channel_1, ENABLE)` 写了吗？ |
> | 4 | `DAC_SetChannel1Data()` 的第一个参数是 `DAC_Align_12b_R` 吗？ |
> | 5 | 用的是 `DAC_SetChannel1Data`（通道**1**）吗？PA4 是通道 1 |

> ⚠️ **★ 方向搞反了？** DAC 值大 = 电压高 = LED **暗**。如果发现"值越大越亮"，说明你心里把 LED 当成了高电平点亮——回到 6.5 ③ 再看一遍。

> ⚠️ **LED 亮了但看不出"渐变"？** 有可能是延时太短。把 `Delay_ms(2)` 改成 `Delay_ms(10)` 试试。

---

## 6.6 实验 10：万用表定量测量

**目标**：不再用眼睛看亮度，而是**用万用表量出 PA4 的实际电压**，和理论值对照。

**这是本章最有"工程味"的实验**——你会亲眼看到理想和现实的差距，并且知道**每一处差距是从哪来的**。

### ① 原理图

同 6.5。但这次多了一个"测量点"：

```
   3V3 ──▶|── LED10 ── R31 1k ──┬── PA4  ◄── 万用表红表笔
                                 │
                              DAC 输出     万用表黑表笔 ── GND
```

### ② 硬件连接

**只需要万用表**（本教程第 2 章硬件清单里就有）：

| 表笔 | 接哪 |
|---|---|
| **红表笔** | **PA4** |
| 黑表笔 | 板上任意 **GND** |

**万用表拨到直流电压档（DCV），量程选 20 V 或自动量程。**

**★ 关于是否要断开 R31：见 6.3.3 的分区**。本实验分两部分做：

- **上半段（1.6 V ~ 3.3 V）零改动**
- 下半段（0 ~ 1.6 V）可选，需要断开

### ③ 原理讲解

**先明确要验证什么**：`DAC_SetChannel1Data(v)` 之后，PA4 上真的能得到 `v × 3.3 / 4095` 伏吗？

**预期的偏差来源**（提前告诉你，免得你以为是 bug）：

| 偏差来源 | 典型影响 | 见 |
|---|---|---|
| **输出缓冲到不了轨** | 0 V 和 3.3 V 两端量不到 | 6.1.3 / 6.2.3 |
| **VDDA 实际不是 3.300 V** | 全程 ±1% 左右的系统偏差 | 5.8.2（同一个道理） |
| **LED 灌电流**（低于临界点时） | 低电压区读数**偏高** | 6.3.2 |
| DAC 本身的偏移/增益误差 | 几个 LSB | 数据手册 |
| 万用表自身精度 | 通常 ±0.5% | —— |

**明白这些，你就不会看到 0.01 V 的误差就慌。**

### ④ 完整代码

**让 DAC 停在几个固定值上，每个值停留 3 秒**，方便你从容测量：

```c
#include "stm32f10x.h"

void Delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 12000; j++);
}

void DA_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    DAC_InitTypeDef  DAC_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);     // ★ APB1

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_4;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    DAC_InitStructure.DAC_Trigger                      = DAC_Trigger_None;
    DAC_InitStructure.DAC_WaveGeneration               = DAC_WaveGeneration_None;
    DAC_InitStructure.DAC_LFSRUnmask_TriangleAmplitude = DAC_LFSRUnmask_Bit0;
    DAC_InitStructure.DAC_OutputBuffer                 = DAC_OutputBuffer_Enable;
    DAC_Init(DAC_Channel_1, &DAC_InitStructure);

    DAC_Cmd(DAC_Channel_1, ENABLE);
}

int main(void)
{
    /* 要测量的测试点：从高到低，每个停留 3 秒 */
    const uint16_t test_points[] = { 4095, 3720, 3100, 2480, 2048, 1860, 1240, 620, 0 };
    const uint8_t  point_count   = 9;
    uint8_t i;

    DA_Init();

    while (1)
    {
        for (i = 0; i < point_count; i++)
        {
            DAC_SetChannel1Data(DAC_Align_12b_R, test_points[i]);
            Delay_ms(3000);                     // 停留 3 秒，够你读数
        }
    }
}
```

**测试点的选择**：覆盖了 3.3 V 附近、1.65 V 中点、1.6 V 临界线上下、以及 0。

### ⑤ 现象验证

**把每个点的读数填进下表**（理论值 = 值 × 3.3 ÷ 4095）：

| 设定值 | 理论电压 | 万用表读数 | 偏差 | LED10 状态 |
|---|---|---|---|---|
| 4095 | 3.300 V | ______ | ______ | 灭 |
| 3720 | 2.998 V | ______ | ______ | 灭 |
| 3100 | 2.498 V | ______ | ______ | 灭 |
| 2480 | 1.999 V | ______ | ______ | 灭 |
| 2048 | 1.650 V | ______ | ______ | 灭 |
| 1860 | 1.499 V | ______ | ______ | **刚亮？** |
| 1240 | 0.999 V | ______ | ______ | 亮 |
| 620 | 0.500 V | ______ | ______ | 更亮 |
| 0 | 0.000 V | ______ | ______ | 最亮 |

**★ 三个必须观察到的现象：**

**① 1.6 V 以上的点，读数应该和理论值很接近（偏差在 1~2% 以内）。**

这说明 **DAC 在上半段工作是准的**，而且 **LED 没有干扰**。**这就是 6.3.3 那张分区图的实际验证。**

**② LED 开始发光的那一行，就是临界点。**

找到它，写下你板子的实际临界值：**______ V**。

这一行的读数已经开始有偏差了——**因为这一刻 LED 刚开始导通**。

**③ 最低点（设定 0）量不到 0 V，大约会停在 0.1 ~ 0.2 V。**

这是 **6.1.3 / 6.2.3 讲过的输出缓冲限制**，**不是 bug**。而且此时 LED 正大量灌电流，两个因素叠加。

> **★ 把观察结果和理论对上号**，这个实验才算做完：
>
> | 现象 | 对应原理 |
> |---|---|
> | 上半段准 | DAC 正常工作 |
> | 低电压区偏高 | LED 灌电流（6.3.2） |
> | 量不到 0 V | 输出缓冲到不了轨（6.2.3） |
> | 全程略有系统偏差 | VDDA ≠ 3.300 V（5.8.2 的同一个坑） |

**如果想让下半段也准**：按 6.3.4 的方式 B，断开 R31（PA4 对应的那颗），重新测一遍低电压区。

### ⑥ 避坑提示

> ⚠️ **★ 红表笔别打滑。** PA4 和 PA3/PA5 挨得很近。**先接好 GND 黑表笔，再拿红表笔去点 PA4。** 打滑短接到 3.3 V 一般不会烧（有保护），但会让你读数突然跳变，以为程序有问题。

> ⚠️ **万用表要用 DCV 档，不是 ACV。** DAC 输出的是**直流**。用 ACV 档量出来接近 0，会让你以为 DAC 没工作。

> ⚠️ **读数一直在小范围跳动？** 正常。DAC 输出有噪声、万用表本身也有最后一位的抖动。**取一个稳定的中间值记录即可。**

> ⚠️ **别指望 0.1% 的精度。** 本板的 VDDA 由 LDO 提供，VDDA 本身就有 ±1% 误差，**这决定了你的测量精度上限**。要更准，就得用外部精密基准——超出本教程范围。

> ⚠️ **★ 停在某个点不动了？** 那是 3 秒的延时到了、程序切到下一个点了，不是卡死。看 LED 亮度是不是跟着变了。

---

## 6.7 实验 11：定时器 + DMA 输出正弦波

**目标**：让 PA4 自动输出一个**连续变化的正弦波**，CPU 全程不参与。

这是全书的**技术顶点**——它把前面所有章节的东西串起来了：

| 用到的 | 来自 |
|---|---|
| GPIO + 模拟模式 | 第 4 章 4.2.3 |
| DAC 基础 | 本章 6.4 |
| **DMA 配置思路** | 第 5 章 5.9.2 |
| 定时器 | 本章首次出现（只讲够用的部分） |
| 数学换算 | 第 5 章 5.7（B 参数方程同一类） |

### ① 原理图

```
   ┌──────────┐   更新事件    ┌──────────┐
   │   TIM6   │──── TRGO ───►│   DAC    │
   │ (定时器) │               │  触发输入 │
   └──────────┘               └────┬─────┘
                                   │ 每次触发，取一个值输出
                                   ▼
   ┌──────────┐   DMA1 通道 3  ┌──────────┐
   │ 正弦表    │◄────自动搬运───│ DAC 数据 │
   │ uint16_t │                │ 寄存器    │
   │ [128]    │                └────┬─────┘
   └──────────┘                     ▼
                                   PA4 ──► LED10
```

**三者的分工**：

| 部件 | 干什么 | 频率 |
|---|---|---|
| **DMA** | 把正弦表里的值**一个个搬进** DAC 数据寄存器 | 由 DAC 请求驱动 |
| **TIM6** | 每隔固定时间**触发一次** DAC 转换 | **决定波形频率** |
| **DAC** | 把搬进来的值**变成电压**输出 | 跟随 TIM6 |

**CPU 完全空闲**——这就是 DMA 的意义（第 5 章 5.9.2 已经体会过一次）。

### ② 硬件连接

**不需要接线。** 输出在 PA4（LED10 上能看到效果）。

> **想看真正的波形？** 需要示波器。**没有示波器也能做这个实验**——LED10 的亮度会以波形频率变化，低频时能看到"呼吸/频闪"。**现象验证一节会讲怎么用 LED 判断。**

### ③ 原理讲解

**① 为什么要用 DMA？**

如果不用 DMA，就只能这样写：

```c
while (1)
{
    for (i = 0; i < 128; i++)
    {
        DAC_SetChannel1Data(DAC_Align_12b_R, sine_table[i]);
        Delay_us(100);      // 靠延时卡时间
    }
}
```

**问题**：`Delay_us` 的精度取决于循环次数，而且**主循环一被打断（中断、别的代码），波形马上就抖**。

**用 TIM6 + DMA 之后**，时序由**硬件定时器**保证，CPU 在干什么都不影响波形。这是"**用硬件做硬件的活**"的典型例子。

**② 波形频率怎么算？**

```
   正弦波频率  =  触发频率  ÷  表里的点数

   触发频率由 TIM6 决定：
       触发频率 = 72 MHz ÷ (预分频+1) ÷ (周期+1)
```

**本实验的设定**：

| 参数 | 值 | 计算 |
|---|---|---|
| TIM6 预分频 | 72 − 1 | 72 MHz ÷ 72 = **1 MHz** 计数频率 |
| TIM6 周期 | 7812 − 1 | 1 MHz ÷ 7812 ≈ **128 Hz** 触发 |
| 正弦表点数 | **128** | —— |
| **正弦波频率** | | 128 ÷ 128 = **1 Hz** |

**1 Hz 的正弦——每秒一个完整周期。** 这个频率**低到能用眼睛看**：LED10 的亮度会在一秒内完成一次"最亮 → 最暗 → 最亮"的变化。

> **想看真正的正弦波（比如 1 kHz）？** 把 TIM6 周期改成 `78 - 1`（1 MHz ÷ 78 ≈ 12.8 kHz，÷128 = 100 Hz），或 `8 - 1`（1 MHz ÷ 8 = 125 kHz，÷128 ≈ 977 Hz）。**但那时肉眼只能看到 LED 稳定在中等亮度**（因为变化太快，人眼做了平均）——这正是 6.1.4 里 PWM 那条"用眼睛看不见的高频"的道理。
>
> **本教程默认 1 Hz**，为的是**用 LED 就能验证**。

**③ 正弦表怎么来的？**

正弦波的 12 位 DAC 值：

```
   value(i) = 2048 + 2047 × sin( 2π × i / N )

   2048 = 中心值（对应 1.65 V，正弦的中轴）
   2047 = 幅度（留 1 个 LSB 余量，保证 0 ≤ value ≤ 4095）
```

**本实验在程序启动时用 `sinf()` 算一遍**（只算一次，128 次乘加，耗时微不足道）。

> **实际工程里通常把正弦表做成 `const` 数组放在 Flash 里**，省 RAM 也省启动时间。本教程用"启动时计算"，是因为**一张 128 行的数字表印在书里对读者没有价值**——算出来更清楚，也顺便复习了 5.7 的三角函数用法。

### ④ 完整代码

```c
#include "stm32f10x.h"
#include <math.h>

#define PI          3.14159265f
#define SINE_POINTS 128                 /* 一个正弦周期的采样点数 */

/* ★ DMA 的搬运源：正弦波数据表 */
static uint16_t g_sine_table[SINE_POINTS];

/* ---------- 生成正弦表（只在启动时跑一次） ---------- */
static void SineTable_Generate(void)
{
    uint16_t i;

    for (i = 0; i < SINE_POINTS; i++)
    {
        /* 2048 = 中轴（1.65V），2047 = 幅度 */
        g_sine_table[i] = (uint16_t)(2048.0f
                          + 2047.0f * sinf(2.0f * PI * i / SINE_POINTS));
    }
}

/* ---------- TIM6 初始化：负责"什么时候换下一个值" ---------- */
void TIM6_Init(void)
{
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;

    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM6, ENABLE);    // TIM6 也在 APB1

    /* 72 MHz ÷ 72 = 1 MHz；1 MHz ÷ 7812 ≈ 128 Hz 触发 */
    TIM_TimeBaseStructure.TIM_Prescaler         = 72 - 1;
    TIM_TimeBaseStructure.TIM_Period            = 7812 - 1;
    TIM_TimeBaseStructure.TIM_ClockDivision     = TIM_CKD_DIV1;
    TIM_TimeBaseStructure.TIM_CounterMode       = TIM_CounterMode_Up;
    TIM_TimeBaseStructure.TIM_RepetitionCounter = 0;
    TIM_TimeBaseInit(TIM6, &TIM_TimeBaseStructure);

    /* ★ 让 TIM6 每次计数溢出时，向 TRGO 输出一个触发脉冲 */
    TIM_SelectOutputTrigger(TIM6, TIM_TRGOSource_Update);

    TIM_Cmd(TIM6, ENABLE);
}

/* ---------- DMA 初始化：负责"把值搬过去" ---------- */
void DMA_Init_DAC(void)
{
    DMA_InitTypeDef DMA_InitStructure;

    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);      // DMA 在 AHB

    DMA_DeInit(DMA1_Channel3);                              // ★ DAC 通道1 → DMA1 通道3

    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&DAC->DHR12R1;  // 目标：DAC 数据寄存器
    DMA_InitStructure.DMA_MemoryBaseAddr     = (uint32_t)g_sine_table;   // 源：正弦表
    DMA_InitStructure.DMA_DIR                = DMA_DIR_PeripheralDST;    // 方向：内存 → 外设
    DMA_InitStructure.DMA_BufferSize         = SINE_POINTS;
    DMA_InitStructure.DMA_PeripheralInc      = DMA_PeripheralInc_Disable;  // 目标地址不变
    DMA_InitStructure.DMA_MemoryInc          = DMA_MemoryInc_Enable;       // 源地址递增
    DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;  // 16 位
    DMA_InitStructure.DMA_MemoryDataSize     = DMA_MemoryDataSize_HalfWord;
    DMA_InitStructure.DMA_Mode               = DMA_Mode_Circular;     // 循环：搬完自动从头
    DMA_InitStructure.DMA_Priority           = DMA_Priority_High;
    DMA_InitStructure.DMA_M2M                = DMA_M2M_Disable;
    DMA_Init(DMA1_Channel3, &DMA_InitStructure);

    DMA_Cmd(DMA1_Channel3, ENABLE);
}

/* ---------- DAC 初始化：这次要用 TIM6 触发 ---------- */
void DA_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    DAC_InitTypeDef  DAC_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_4;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    DAC_InitStructure.DAC_Trigger                      = DAC_Trigger_T6_TRGO;  // ★ 换触发源
    DAC_InitStructure.DAC_WaveGeneration               = DAC_WaveGeneration_None;
    DAC_InitStructure.DAC_LFSRUnmask_TriangleAmplitude = DAC_LFSRUnmask_Bit0;
    DAC_InitStructure.DAC_OutputBuffer                 = DAC_OutputBuffer_Enable;
    DAC_Init(DAC_Channel_1, &DAC_InitStructure);

    DAC_Cmd(DAC_Channel_1, ENABLE);

    /* ★ 让 DAC 每次收到触发就发一个 DMA 请求 */
    DAC_DMACmd(DAC_Channel_1, ENABLE);
}

/* ========== 主函数 ========== */
int main(void)
{
    SineTable_Generate();

    DA_Init();          /* 顺序：先配 DAC */
    DMA_Init_DAC();     /* 再配 DMA */
    TIM6_Init();        /* 最后开定时器——一开就开始输出波形了 */

    while (1)
    {
        /* ★ 这里什么都不用做。
         *   定时器负责节奏，DMA 负责搬数据，DAC 负责输出电压。
         *   CPU 是空闲的。 */
    }
}
```

**★ 三个"互相认识"的地方，缺一不可：**

| 谁认识谁 | 怎么实现的 | 漏了会怎样 |
|---|---|---|
| TIM6 → DAC | DAC 的 `DAC_Trigger_T6_TRGO` | 不触发，输出永远不变 |
| DAC → DMA | `DAC_DMACmd(DAC_Channel_1, ENABLE)` | 没有搬运请求，输出停在**第一个值** |
| DMA → DAC 寄存器 | `DMA_PeripheralBaseAddr = &DAC->DHR12R1` | 搬错地方，输出乱来 |

> **★ 为什么 `DMA_PeripheralBaseAddr` 是 `DHR12R1` 而不是别的？**
>
> `DHR12R1` = **D**ata **H**olding **R**egister, **12**-bit, **R**ight-aligned, channel **1**。
>
> 拆开读：**12 位、右对齐、通道 1 的数据保持寄存器**。DMA 往里写，DAC 就输出对应电压。
>
> 如果用的是通道 2，这里要改成 `DHR12R2`；用左对齐则是 `DHR12L1`。**写错了不会编译报错，只是输出不对。**

> **★ 主循环为什么是空的？**
>
> 因为**所有工作都由硬件完成了**。这看起来像"浪费"，实际上是最优——**CPU 空出来可以去干别的事**（比如同时刷新 OLED、响应按键、跑控制算法）。
>
> **这就是嵌入式系统的核心思想：能用硬件做的，绝不用软件循环去等。**

### ⑤ 现象验证

**没有示波器也能验证：**

| 观察 | 说明 |
|---|---|
| **LED10 的亮度以约 1 秒为周期缓慢变化** | 1 Hz 正弦波。亮到最亮约 0.5 秒，再暗回去 |
| 亮度的变化是**平滑**的吗？ | 是。128 个点足够密，看不出台阶 |
| 把 TIM6 周期改小（如 `78 - 1`） | LED 变化变快；再快就变成"稳定在某亮度"（人眼平均掉了） |

**★ 用 LED 判断波形质量**：

| LED 表现 | 说明 |
|---|---|
| 平滑地亮-暗-亮 | **正常**。这是正弦波 |
| 每次变到最亮时**停顿一下** | 表头表尾衔接处不平滑（本项目 128 点整周期，不会有这问题） |
| 亮度**跳一下**或闪烁 | 触发时序有问题，检查 TIM6 的 TRGO 设置 |

**有示波器的话（理想验证）：**

- 探头接 PA4，地接 GND
- 应该看到一个**峰值约 3.1 V、谷值约 0.2 V、1 Hz** 的正弦波
- **波形上下不到轨**——这是 6.2.3 讲的输出缓冲限制，**正常现象**

> **★ 把 `sinf` 换成锯齿波？** 把 `SineTable_Generate()` 里那行换成：
>
> ```c
> g_sine_table[i] = (uint16_t)(i * 4095 / (SINE_POINTS - 1));   // 锯齿波
> ```
>
> 示波器上就是一条斜线。**自己改一下试试，是理解"表里的值决定波形形状"最快的方式。**

### ⑥ 避坑提示

> ⚠️ **★ 输出停在某个固定电压不动？** 99% 是漏了 `DAC_DMACmd(DAC_Channel_1, ENABLE)`。没有它，DAC 不会向 DMA 发请求，波形表里的值就永远搬不过去。输出会停在**表里的第一个值**上（2048，约 1.65 V）——**看到 PA4 稳定在 1.65 V，先查这一行。**

> ⚠️ **★ `DAC_Trigger` 忘了从 `DAC_Trigger_None` 改成 `DAC_Trigger_T6_TRGO`？** 那样 DAC 会在数据一写进来就立刻输出，**完全不理会 TIM6**，DMA 也会以最快的速度把整个表搬完——输出会乱成一团。

> ⚠️ **DMA 通道号写错。** DAC 通道 1 是 **DMA1 通道 3**，通道 2 是**通道 4**。写成通道 1（那是 ADC1 的，第 5 章 5.9.2 用过）会完全没反应。

> ⚠️ **`DMA_DIR` 反了。** 这里是 **`DMA_DIR_PeripheralDST`**（**外设是目的地**），因为数据是"内存 → 外设"。第 5 章 ADC 是 `DMA_DIR_PeripheralSRC`（外设是源）。**两个字面很像，方向相反。**

> ⚠️ **中断里/主循环里别去写 `DAC_SetChannel1Data()`。** 那会和 DMA 抢数据寄存器，输出会抖。**DMA 模式下，DAC 数据寄存器归 DMA 管。**

---

## 6.8 实验 12（可选）：DAC → ADC 回环验证

> **⚠️ 这是一个可选实验，而且它需要动硬件。**
>
> **本节的第一个价值，其实是"讲清楚为什么它需要动硬件"**——这个分析过程本身就是一堂电路课。**如果你不想动烙铁，直接读 6.8.1 和 6.8.3，然后跳到 6.9，完全不影响后面的内容。**

### 6.8.1 想做什么

**思路很漂亮**：让 DAC 输出一个电压，再用 ADC 把它读回来，**看读到的和设定的对不对得上**。

```
   DAC ──► PA4 ──────[ 飞线 ]──────► PA7 ◄── ADC
   (输出)                            (输入)

   我设 1.65 V  →  ADC 读回来是不是 2048？
```

**这就是"教学闭环"**：第 5 章学的 ADC 和第 6 章学的 DAC，在这里握手。

### 6.8.2 ⚠️ 为什么本板不能直接飞线

**如果你直接把 PA4 和 PA7 用杜邦线连起来，结果会让你困惑。** 原因是 **PA7 上已经挂着一个分压电路**（第 5 章 5.2.3）：

```
                                            PA4 (DAC 输出)
                                             │
                                             │  ← 你想接的飞线
                                             │
   3V3 ── R59 (10K NTC) ──┬── 板载 ADC1 ─────┴──── PA7
                          │
                        R48 (10K)
                          │
                         GND
```

**把这两点接起来之后，PA7 那一点上并联了三样东西：**

| 并联的元件 | 等效 | 对 DAC 输出的影响 |
|---|---|---|
| **R59（10K 到 3V3）** | 往上拉 | 想把节点拉到 3.3 V |
| **R48（10K 到 GND）** | 往下拉 | 想把节点拉到 0 V |
| **R31 + LED10（到 3V3）** | 非线性 | 低压时灌电流（6.3.2） |

**R59 和 R48 串联在 3V3 和 GND 之间，中间抽头就是 PA7。** 对 DAC 来说，这相当于**接了一个 10K∥10K = 5 kΩ 的负载，而且这个负载自己会把电压拽向 1.65 V**。

```
   DAC 想输出 3.0 V
        │
        ▼
   ┌─────────┐
   │ 5 kΩ 的  │  ← 这个负载"倔强地"想把节点拉到 1.65 V
   │ 分压网络  │
   └─────────┘
        │
        ▼
   实际量到 2.3 V 左右 ── 而且温度一变，分压比还跟着变
```

**结论：PA7 上做不了回环。**

**那换 PB1（电位器）呢？** 更糟——电位器是一个**可变**分压器，抽头一动负载就变。**PB0（光敏）同理，而且阻值随光照变。**

> **★ 这不是 DAC 的问题，是"往一个已经有驱动的节点上强加电压"必然的结果。**
>
> 这个道理在电路里反复出现：**要测量一个信号，测量电路的内阻必须远大于信号源内阻。** 反过来，要往一个节点"灌"一个电压，就必须能压过节点上原有的驱动能力。

### 6.8.3 要做这个实验，需要满足的前提

**必须同时断开两处：**

| 要断开的 | 为什么 | 对应元件 |
|---|---|---|
| **PA4 上的 LED 支路** | 否则低压区灌电流（6.3.2） | R31 |
| **PA7 上的热敏分压支路** | 否则分压网络会拽住 DAC 输出（6.8.2） | R48 或 R59 |

**两处都是拆焊。** 这就是为什么本实验被标成"可选"——**收益（一次闭环验证）和代价（动两次烙铁）不太成正比。**

> **★ 我的建议**：如果你已经在做 6.6 的下半段（为了测低压区已经拆了 R31），**那么在 PA7 那边再拆一颗 R48 就能做这个实验，值得一试**。否则**跳过它**——6.9 的对比表已经能达到同样的认知目标。

### 6.8.4 假如你满足前提，代码在这里

**接线**：杜邦线 PA4 ↔ PA7。**两处支路已断开。**

```c
#include "stm32f10x.h"
#include <stdio.h>

void Delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 12000; j++);
}

/* ---------- 串口（同第 4 章 4.9.2） ---------- */
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

/* ---------- DAC（PA4 输出） ---------- */
void DA_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    DAC_InitTypeDef  DAC_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);

    GPIO_InitStructure.GPIO_Pin  = GPIO_Pin_4;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    DAC_InitStructure.DAC_Trigger                      = DAC_Trigger_None;
    DAC_InitStructure.DAC_WaveGeneration               = DAC_WaveGeneration_None;
    DAC_InitStructure.DAC_LFSRUnmask_TriangleAmplitude = DAC_LFSRUnmask_Bit0;
    DAC_InitStructure.DAC_OutputBuffer                 = DAC_OutputBuffer_Enable;
    DAC_Init(DAC_Channel_1, &DAC_InitStructure);

    DAC_Cmd(DAC_Channel_1, ENABLE);
}

/* ---------- ADC（PA7 采集） ---------- */
void AD_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    ADC_InitTypeDef  ADC_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_ADC1, ENABLE);
    RCC_ADCCLKConfig(RCC_PCLK2_Div6);

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

/* ========== 主函数：DAC 输出 → ADC 读回 → 比较 ========== */
int main(void)
{
    uint16_t set_value, got_value;
    float    error_mv;

    USART1_Init(9600);
    DA_Init();
    AD_Init();

    printf("SET   GOT   ERR(mV)\r\n");

    while (1)
    {
        /* 从 4095 往 0 扫，每 256 个值取一个测试点 */
        for (set_value = 4095; set_value > 0; set_value -= 256)
        {
            DAC_SetChannel1Data(DAC_Align_12b_R, set_value);
            Delay_ms(5);                    /* 等输出稳定 */

            got_value = AD_GetValue();
            error_mv  = (got_value - set_value) * 3.3f / 4095.0f * 1000.0f;

            printf("%4d  %4d  %6.0f\r\n", set_value, got_value, error_mv);

            Delay_ms(100);
        }
    }
}
```

**串口上会看到：**

```
SET   GOT   ERR(mV)
4095  4060   -28
3839  3812   -22
3583  3558   -20
...
1535  1524    -9
1279  1270    -7
1023  1015    -6
...
 511   505    -5
 255   251    -3
```

**★ 读数会略小于设定值**，原因还是那两个老朋友：

| 原因 | 影响 |
|---|---|
| DAC 输出缓冲到不了轨（6.2.3） | 高值端偏低更明显 |
| ADC 的量化与增益误差（5.1.4） | 全程几个 LSB |

**误差在 ±30 mV 以内，就算这个回环验证通过了。** 全程一致的负偏差说明是系统性误差（正常）；**如果误差忽正忽负、跳得很大**，那就要检查接线和断开的电阻是否确实断开了。

---

## 6.9 ADC 与 DAC 对比总结

**本章和上一章合起来，构成了"数字世界 ↔ 模拟世界"的完整通路。**

| | **ADC**（第 5 章） | **DAC**（第 6 章） |
|---|---|---|
| 全称 | Analog-to-Digital Converter | Digital-to-Analog Converter |
| 方向 | 模拟 → 数字 | **数字 → 模拟** |
| 输入 | 引脚上的电压 | 你写进寄存器的整数 |
| 输出 | 0 ~ 4095 的整数 | 引脚上的电压 |
| 分辨率 | 12 位 | 12 位 |
| **挂在哪条总线** | **APB2** | **APB1** ⚠️ |
| **开时钟函数** | `RCC_APB2PeriphClockCmd()` | `RCC_APB1PeriphClockCmd()` ⚠️ |
| **GPIO 模式** | `GPIO_Mode_AIN` | `GPIO_Mode_AIN` |
| 本板引脚 | PA7 / PB0 / PB1 | **PA4 / PA5** |
| 引脚是否复用其他功能 | 否 | **是** ⚠️ PA4/PA5 上有 LED |
| 转换耗时 | 约 5.7 µs（本教程设置） | 约 0.4 µs（快得多） |
| 数据对齐 | `ADC_DataAlign_Right` | `DAC_Align_12b_R` |
| 触发方式 | 软件 / 外部 | `None` / 软件 / **TIM6-TRGO** |
| DMA 通道 | ADC1 → **DMA1 通道 1** | DAC1 通道1 → **DMA1 通道 3** |
| 本板的主要坑 | 引脚配成上下拉会拉偏电压 | **挂载的 LED 在低压区灌电流** |
| 典型用途 | 采集温度、光照、电池电压 | 输出基准、波形、音频 |

**三条最值得记住的对照：**

1. **总线上，DAC 是全书唯一的例外。** ADC 和 GPIO 都在 APB2，只有 DAC 在 APB1。**写 DAC 代码时，第一行多看一眼。**

2. **引脚模式上，两者都用 `GPIO_Mode_AIN`——但理由不同。** ADC 是"让电压原封不动进来"，DAC 是"让电压原封不动出去"。**同一个模式，两种用途。**

3. **转换时间上，DAC 快得多。** 因为 ADC 要"猜 12 次"（逐次逼近），而 DAC 是 12 个位同时作用在电阻网络上。**所以波形输出天然适合 DAC。**

> **★ 从"感知"到"表达"**：第 5 章你让单片机**感知**了世界（读温度、读光照），第 6 章你让它**表达**了意图（输出指定的电压）。
>
> **到这一步，你已经具备了做闭环控制的最小能力**——测量（ADC）、判断（CPU）、输出（DAC）。恒温箱、自动亮度调节、电源管理，本质都是这三步的组合。

---

## 6.10 本章小结

1. **DAC 和 ADC 互逆**：ADC 把电压变成 0~4095，DAC 把 0~4095 变成电压。换算公式形式相同、方向相反。
2. **F103C8T6 只有 DAC1，两路输出**：PA4 = 通道 1、PA5 = 通道 2。**没有替代引脚。**
3. **★ 本板 PA4/PA5 上挂了 LED**（第 4 章 4.5⑥ 的伏笔在此兑现）：
   - **1.6 V 以上** LED 截止，**DAC 输出干净**
   - **1.6 V 以下** LED 灌入电流，**输出被拉偏且非线性**
   - 临界点实测最简单——**看 LED 什么时候开始亮**
4. **输出缓冲**决定带载能力：开了能带负载，但**输出到不了 0 V 和 3.3 V**（约 0.2 ~ 3.1 V）。
5. **★ `RCC_APB1PeriphClockCmd()`——全书唯一一个 APB1。** 第 1 章起预告了四次，6.4.1 正式兑现。
6. **DAC 引脚配 `GPIO_Mode_AIN`**，不是推挽输出——**把 GPIO 的数字部分彻底让开给 DAC**。
7. **`DAC_SetChannel1Data()` 是通道 1（PA4），`SetChannel2Data()` 是通道 2（PA5）**，别写串。
8. **输出波形三件套**：**TIM6 定节奏**（`TIM_TRGOSource_Update` → TRGO）、**DMA 搬数据**（DMA1 通道 3 → `DAC->DHR12R1`）、**DAC 输出电压**（`DAC_Trigger_T6_TRGO` + `DAC_DMACmd()`）。**CPU 全程空闲。**
9. **回环实验（DAC→ADC）在本板做不了**，因为 ADC 引脚上都常年挂着分压网络，会拽住 DAC 输出。**要做得同时拆掉两处支路。**

**本章勘误**（参考资料中的问题）：

| # | 出处 | 问题 |
|---|---|---|
| 1 | PPT 外设总表 / DOCX | DAC 仅作为名词出现 2 次，**无任何讲解与示例**。**本章 100% 原创**，无法对照原始资料核对，引脚与参数一律以数据手册和板载原理图为准 |
| 2 | PPT 引脚定义图 | PA4/PA5 的复用功能栏**未列 DAC**（DAC_OUT 属附加功能），只看 PPT 会以为这两脚没别的用途 |

---

**上一章**：[第 5 章 ADC](05-ADC.md) —— 让电压变成数字。

---

> **图示来源**：本章**无外部图示**。参考资料（PPT 207 页 / DOCX 694 段）中 DAC 完全空白，所有示意图均为本教程自绘（ASCII 图），电路描述一律以板载原理图和 [附录C](附录C-板载引脚总表.md) 为准。
>
> **本教程的全部章节到此结束。** 附录 A（标准库函数速查）、附录 B（常见问题 FAQ）见目录。

---

[返回目录](README.md)
