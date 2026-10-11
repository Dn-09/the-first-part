# STM32 入门培训教材

## 提高篇

### 提 7 IIC 时序详解：看懂 OLED 背后的通信

> 前置：基础篇第11章软件IIC与OLED；串口观察沿用第6章和提高篇提4。先完成“检查线路、扫描地址”两个实验，再结合代码理解一次传输。OLED写入和接收字节放在后面选读。

本章中的IIC、I2C和I²C指同一种通信方式。环境沿用STM32F103C8T6、SPL V3.5.0、Keil5、Arm Compiler 5 + MicroLIB和ST-Link。

基础篇用OLED显示内容，这一章向下看一层：单片机怎样叫到这块屏幕，怎样把一个字节交给它，又怎样知道对方有没有回应。先掌握接线、调用和排错需要的知识，不展开复杂的多主机机制。

#### 7.1 本章要完成什么

可以把IIC想成“有节拍的点名传话”：STM32负责打节拍，先报设备地址，被叫到的设备回应，再开始传数据。这里的“回应”就是后面要检查的ACK。

| 学习任务 | 能看到什么 | 对应代码 |
|---|---|---|
| 检查空闲线路 | 串口显示SCL、SDA当前电平 | 7.10.3完整入口 |
| 扫描设备地址 | 串口列出产生应答的7位地址 | 7.10.4完整入口 |
| 拆开一次地址探测 | START、地址字节、ACK、STOP对应具体函数 | 7.10.5完整入口，选做 |
| 理解OLED写入 | 地址、控制字节、命令或显示数据分别在什么位置 | 7.9，结合配套OLED驱动 |

先得到“地址有应答”的现象，再看时序与函数。看到ACK，只能说明这一阶段有设备回应；屏幕是否显示正确，还需要正确的屏幕初始化、命令和显示数据。

#### 7.2 准备工程与连接硬件

##### 7.2.1 工程准备

| 项目 | 本章使用 |
|---|---|
| 开发板 | STM32F103C8T6培训开发板，独立供电 |
| OLED通信引脚 | PB8接SCL，PB9接SDA |
| 电脑观察 | CH340 USB转TTL模块与串口助手 |
| 已有模块 | 提4的完整 `Serial.c/.h`；模板中的 `Delay.c/.h`，提供 `Delay_ms` |
| 新增模块 | `Hardware/IICBus.c`、`Hardware/IICBus.h` |
| 入口 | `User/main.c`，每次只使用一个完整入口 |
| 标准库源文件 | 沿用RCC、GPIO、USART、`misc.c`等已有文件 |

本章的软件IIC通过GPIO产生时序，**不需要加入 `stm32f10x_i2c.c`，也不需要使能I2C1外设**。新增文件要加入Keil分组，并检查 `Hardware` 的头文件搜索路径。

`IICBus` 是本章提供的独立实验模块，接口名称与基础篇的 `MyI2C` 分开。做7.10的独立实验时，入口只调用 `IICBus`；原有OLED模块可以保留文件，但不要同时调用两套总线驱动操作PB8、PB9。整合OLED工程时，再根据已有驱动接入7.9的检查方式。

本章没有依赖尚未核对的 `Delay_us` 接口。微秒等待放在 `IICBus.c` 内部，使用STM32F103 Cortex-M3的DWT周期计数器；已有 `Delay_ms` 继续负责两次实验之间的间隔。暂时把内部计时看作工具函数即可，不需要先学习调试单元。

##### 7.2.2 OLED怎样接开发板

按排针上的**信号名称**接线，不按不同商家模块的排针顺序猜测。

| 开发板端 | OLED端 | 作用 |
|---|---|---|
| PB8 | SCL／SCK，须确认该脚在IIC模式下是时钟脚 | 时钟线，控制传输节拍 |
| PB9 | SDA | 数据线，传输地址、数据与应答 |
| GND | GND | 共地 |
| 与模块要求相符的电源 | VCC／VDD | 给OLED模块供电，电压按实物说明核对 |

配套引脚约定是 **PB8=SCL、PB9=SDA**。复制其他教程代码时，先检查这两个定义，不能直接照搬相反的接法。

OLED的供电电压、控制器型号和板载上拉电阻需要分别确认。模块可以用某个电压供电，并不自动说明它的SCL、SDA也适合该电压；还要检查两根线被上拉到哪里。本章实验的前提是总线使用已核对、与各设备匹配的3.3V逻辑电平。

##### 7.2.3 上拉与开漏，只记住这件事

把共享的线想成一根带弹簧的绳子：设备可以把它拉低，松手以后，由上拉电阻把它带回高电平。

本章把PB8、PB9配置为 `GPIO_Mode_Out_OD`，即开漏输出：

| GPIO操作 | 开漏引脚实际做什么 |
|---|---|
| `GPIO_ResetBits` | 主动把线拉低 |
| `GPIO_SetBits` | 松开这根线，让上拉电阻把它带高 |
| `GPIO_ReadInputDataBit` | 读取线上的实际电平，包括其他设备拉低的结果 |

因此，程序“松手”以后，线不一定立即变高；其他设备也可能正在拉低。应答检测和时钟等待都要读取实际引脚。

SCL、SDA各需要上拉。先查OLED模块或开发板是否已经安装；确认没有时，再按3.3V逻辑与实际线路选择外部上拉。短距离、低速实验可从每根线4.7kΩ作为候选值核对，最终以线路上升速度和器件要求为准。多块模块自带上拉时，电阻会并联，不能只看其中一块的标注。

##### 7.2.4 开发板怎样连接电脑

IIC连接开发板与OLED；电脑通过串口查看实验结果。CH340负责USB与TTL串口之间的转换。

| 开发板端 | CH340端 | 本章用途 |
|---|---|---|
| PA9，USART1_TX | RXD／RX | 把检测结果发送给电脑 |
| GND | GND | 共地 |
| PA10，USART1_RX | TXD／TX | 本章三个入口均无需接收电脑数据，可以不接；保留提4接线时沿用已核对的逻辑电平 |
| 开发板电源引脚 | VCC、5V、3.3V等 | 本章不连接，开发板独立供电 |

CH340用USB线接电脑，ST-Link按已有工程接法负责下载。OLED通过开发板侧核对过的电源供电，不接到CH340的TXD或RXD上。

电脑首次使用时，安装厂商驱动：[WCH CH340／CH341 Windows驱动下载](https://www.wch.cn/downloads/CH341SER_EXE.html)。安装后在设备管理器确认COM端口，串口助手设置 **9600、8位数据、无校验、1位停止、无流控、文本接收**。

##### 7.2.5 上电前核对

1. 开发板、OLED、CH340共地。
2. PB8接SCL，PB9接SDA。
3. OLED供电与总线上拉电压已经核对。
4. 本次入口只使用一套PB8、PB9总线驱动。
5. 先断电接线，接好后上电；改动接线或拆接OLED时先断电。

#### 7.3 先跑通两个核心实验

##### 7.3.1 实验一：检查空闲线路

1. 复制已经能通过串口输出的工程。
2. 加入7.10.1、7.10.2中的 `IICBus.h/.c`。
3. 用7.10.3作为完整 `main.c`，暂时不调用OLED初始化。
4. 打开串口，下载后复位。

正常空闲时，预期看到：

~~~text
IIC idle-line check
init=OK
SCL=1 SDA=1
SCL=1 SDA=1
~~~

这段程序只初始化并观察线路，没有给OLED发送显示命令。**两根线为高是正常空闲的必要条件，不能据此认定已经找到了OLED。**

如果一直有一根线为0，先检查供电、接线、上拉与模块状态，处理好再做扫描。

##### 7.3.2 实验二：扫描设备地址

1. 沿用实验一的模块和接线。
2. 把7.10.4替换为 `main.c`。
3. 复位后观察 `SCAN BEGIN` 与 `SCAN END` 之间的结果。
4. 记录出现应答的7位地址，和配套驱动中的地址写法核对。

扫描像“逐个点名”：程序只发送各地址的写方向地址字节，检查第9个时钟上的回应，然后停止。本例不追加命令或显示数据。

扫描范围是普通7位地址 `0x08～0x77`，跳过保留地址。只在本章已经确认的OLED实验线路上执行；接入其他器件后，应先核对该器件允许的探测方式。

两轮扫描之间间隔2秒。没有设备应答时会显示：

~~~text
SCAN BEGIN
No ACK address found.
SCAN END count=0
~~~

有应答时输出格式为 `ACK addr7=0x..`，其中数值由实际检测得到。本章不预先固定OLED地址。出现 `SCL_TIMEOUT` 或 `BUS_BUSY` 时停止本轮扫描，先检查线路，再复位重试。

#### 7.4 看懂一笔传输：只记住五件事

##### 7.4.1 START和STOP

像讲话前宣布“开始”、讲完宣布“结束”，IIC也有明确的边界。

| 信号 | 必须看到的变化 | 本章函数 |
|---|---|---|
| START，起始 | SCL为高时，SDA从高变低 | `IICBus_Start()` |
| STOP，停止 | SCL为高时，SDA从低变高 | `IICBus_Stop()` |

正常数据阶段，SDA在SCL低时准备数据，在SCL高时保持稳定。START、STOP就是这个规则中的特殊信号。

##### 7.4.2 一个字节是8位，高位先发

发送 `0xA5`，即二进制 `1010 0101` 时，对应关系如下：

| 时钟序号 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 |
|---|---|---|---|---|---|---|---|---|---|
| 这一拍的内容 | bit7 | bit6 | bit5 | bit4 | bit3 | bit2 | bit1 | bit0 | ACK／NACK |
| 示例SDA值 | 1 | 0 | 1 | 0 | 0 | 1 | 0 | 1 | 由接收方决定 |

前8拍传数据，第9拍检查应答。发送代码中的 `0x80` 用来查看当前最高位，左移一位后，下一位成为新的最高位。

##### 7.4.3 ACK和NACK由谁给出

ACK可以理解为“这一字节我收到了”。在第9拍，**发送方先释放SDA，再由接收方决定是否拉低**。

| 当前传输 | 谁发前8位 | 谁在第9拍应答 |
|---|---|---|
| STM32向OLED写地址或数据 | STM32 | OLED：低为ACK，高为NACK |
| STM32从支持读取的设备接收数据 | 从设备 | STM32：还要继续就ACK，最后一字节后NACK |

写入时收到NACK，应结束本次传输并查原因。读取最后一个字节后，STM32主动给NACK，表示“读到这里就够了”，这属于正常结束。

本章的ACK函数只在第9拍采样；**不会一直等SDA变低**。另一个“等待超时”针对SCL释放后仍不能升高，两者用途不同。

##### 7.4.4 地址和读写位

7位地址可以理解成设备的“门牌号”，读写位表示本次要向它发送，还是从它接收。

| 计算项目 | 形式 |
|---|---|
| 写方向地址字节 | `(address7 << 1) \| 0` |
| 读方向地址字节 | `(address7 << 1) \| 1` |

例如，**只用来演示计算**的7位地址 `0x3C`，对应写地址字节 `0x78`、读地址字节 `0x79`。这不是对配套OLED地址的确认。

本章 `IICBus_Probe`、`IICBus_Write`、`IICBus_Read` 均接收**未左移的7位地址**。低层 `IICBus_SendByte` 接收完整的8位字节，因此手动发送地址时要先完成上述计算。

##### 7.4.5 一次最小探测怎样走完

| 顺序 | 做什么 | 对应调用 |
|---|---|---|
| 1 | 宣布开始 | `IICBus_Start()` |
| 2 | 发送写方向地址字节的8位 | `IICBus_SendByte(address7 << 1)` |
| 3 | 检查第9拍的应答 | `IICBus_ReadAck()` |
| 4 | 宣布结束 | `IICBus_Stop()` |

`IICBus_Probe(address7)` 已经把这四步包好。初学时用它即可，7.10.5再把四步拆开观察。

#### 7.5 初始化函数：到底准备了什么

##### 7.5.1 `IICBus_Init`的调用与结果

调用形式：`IICBus_Status IICBus_Init(void)`，无参数，返回本次准备结果。

| 返回值 | 含义 | 处理方式 |
|---|---|---|
| `IICBUS_OK` | 计时准备完成，SCL、SDA均处于高电平 | 继续探测 |
| `IICBUS_SCL_TIMEOUT` | 释放SCL后，2ms内仍没有读到高电平 | 查时钟线、上拉与从设备状态 |
| `IICBUS_BUS_BUSY` | SCL已高，但SDA仍低 | 查数据线与模块状态 |
| `IICBUS_NOT_READY` | 内部计时条件未准备好 | 核对模板时钟、CMSIS与DWT配置 |

初始化依次完成：

1. 打开GPIOB时钟。
2. 预先把PB8、PB9的输出锁存设为释放状态，再设为开漏输出。
3. 更新 `SystemCoreClock`，准备内部周期计数与等待时间。
4. 释放两根线，检查实际空闲电平。

它**不发送OLED初始化命令**，也不自动寻找设备地址。总线准备好后，才由设备驱动安排自己的初始化。

启动时调用一次。排除供电或线路故障后可以重新初始化；本章不包含自动总线恢复。传输正在进行时不要重新调用它，也不要在传输期间修改系统时钟或停止DWT周期计数。

##### 7.5.2 初始化与引脚操作中的函数

| 函数／配置 | 参数与返回 | 本章作用 | 其他常见用途与注意 |
|---|---|---|---|
| `RCC_APB2PeriphClockCmd` | 时钟标识、`ENABLE／DISABLE`；无返回值 | 开GPIOB时钟 | 也能控制GPIOA、AFIO等APB2外设时钟，标识要选对 |
| `GPIO_StructInit` | 配置结构体地址；无返回值 | 先填入默认字段，再改本章的引脚、速度与模式 | 只准备变量，不把设置写入引脚，应用配置还要调用 `GPIO_Init` |
| `GPIO_Init` | 端口、配置结构体地址；无返回值 | 配置PB8、PB9开漏输出 | 也用于输入、推挽、复用等模式；软件IIC用普通开漏 |
| `GPIO_SetBits` | 端口、引脚掩码；无返回值 | 释放SCL或SDA | 推挽模式下会主动驱动高；本章是开漏，含义要结合模式 |
| `GPIO_ResetBits` | 端口、引脚掩码；无返回值 | 拉低SCL或SDA | 可同时拉低多个指定引脚 |
| `GPIO_ReadInputDataBit` | 端口、单个引脚；返回0或1 | 查实际线电平 | 也用于读取按键；本章读取的是输入状态，不是输出锁存值 |
| `SystemCoreClockUpdate` | 无参数、无返回值 | 更新核心时钟频率变量 | 根据已经配置的时钟树更新变量，本身不切换系统时钟 |

结构体中选择 `GPIO_Speed_2MHz` 是引脚输出速度档位，**不是把IIC设成2MHz**。通信节拍由软件操作与内部等待共同形成。

内部等待取10μs作为一个阶段的最短等待，实际总线频率还受函数执行和中断影响，不承诺固定50kHz。初学实验先用这个设置，确认线路与ACK正常后再考虑提速。

#### 7.6 传输函数怎样用

##### 7.6.1 先看统一返回值

本章所有有状态的接口都返回 `IICBus_Status`。判断成功用 `status == IICBUS_OK`；**OK的数值为0，不用 `if (status)` 表示成功**。

| 状态 | 表示什么 |
|---|---|
| `IICBUS_OK` | 当前操作完成；读取应答时表示ACK |
| `IICBUS_NACK` | 第9拍读到高电平，对方没有应答 |
| `IICBUS_SCL_TIMEOUT` | SCL不能在设定等待时间内升高 |
| `IICBUS_BUS_BUSY` | 本应释放的数据线仍低 |
| `IICBUS_BAD_ARG` | 地址、指针、数量或应答参数无效 |
| `IICBUS_NOT_READY` | 尚未成功准备计时与引脚 |

`IICBus_StatusText(status)` 把状态转成英文短文本，供已有 `Serial_Printf` 打印。高层函数会尝试STOP收尾；如果STOP也失败，优先返回收尾错误，提示本次没有正常释放总线。

##### 7.6.2 底层函数与时序一一对应

以下函数先说明“这一小段做什么”。完整传输通常用7.6.3的高层接口。

| 调用形式 | 参数与返回 | 本次作用 | 其他用途／前提 |
|---|---|---|---|
| `IICBus_Start()` | 无参数；返回状态 | 在SCL高时让SDA下降，随后拉低SCL | 也可产生重复起始；只用于本章单主机总线 |
| `IICBus_Stop()` | 无参数；返回状态 | 在SCL高时释放SDA，使其上升 | 写完、读完或NACK时结束；失败时不能认定总线已恢复 |
| `IICBus_SendByte(data)` | 一个8位字节；返回状态 | 高位先发，产生8个数据时钟 | 地址、控制、命令、数据都能发送；它不包含ACK拍 |
| `IICBus_ReadAck()` | 无参数；返回OK、NACK或错误 | 释放SDA，产生第9拍并读取应答 | 每个成功发送的字节后调用一次 |
| `IICBus_ReceiveByte(&data)` | 有效的字节变量地址；返回状态 | 释放SDA并接收8位，结果写入变量 | 用于支持读取的设备；接收后还要发ACK或NACK |
| `IICBus_SendAck(level)` | 0表示ACK，1表示NACK；返回状态 | STM32在第9拍拉低或释放SDA | 接收中间字节用0，最后一字节用1 |
| `IICBus_GetLines(&scl, &sda)` | 两个字节变量地址，允许空指针；无返回值 | 读取当前两线实际电平 | 用于空闲检查，不负责判断器件身份 |

`ReceiveByte(&data)` 中的 `&` 是把变量的地址交给函数，让函数把收到的字节写进去。它的返回值是操作状态，收到的数据在 `data` 中，两者要分开看。

##### 7.6.3 整笔传输的接口

| 调用形式 | 参数与返回 | 本章作用 | 其他用途／前提 |
|---|---|---|---|
| `IICBus_Probe(address7)` | 普通7位地址；返回状态 | 发送写地址并查应答，最后STOP | 查目标是否回应；没有数据字节，也不识别控制器 |
| `IICBus_Write(address7, data, length)` | 7位地址、有效数组、字节数；返回状态 | 写地址后逐字节发送，每字节检查ACK | 写OLED或其他设备的实际协议字节，数量必须与数组一致 |
| `IICBus_Read(address7, data, length)` | 7位地址、可写数组、接收数量；返回状态 | 直接读数据，中间ACK、最后NACK，然后STOP | 只适合允许直接读取的器件；不自动发送寄存器地址 |

例如，已知数组有3个有效字节时，`length` 就传3；不能把数组容量当成每次有效长度。写字节的具体含义由目标器件决定，`IICBus_Write` 只负责传输。读接口失败时，数组可能只更新了一部分，不应把它当作一组成功接收的数据使用。

这些接口都是**同步调用**：函数执行期间CPU仍在逐拍操作GPIO，成功返回后这笔传输才结束。它们没有队列，也不自动使用DMA。按本章示例统一由主循环调用，中断中只记录待处理事件。

#### 7.7 拆开代码，看四个关键位置

##### 7.7.1 设置数据，再升高时钟

~~~c
IICBus_SetSDA((uint8_t)((data & 0x80u) != 0u));
IICBus_Delay();
status = IICBus_RaiseSCL();
~~~

第一行选择当前最高位：位为0就拉低SDA，位为1就释放SDA。等数据稳定后，再释放SCL并等待线上实际变高。

函数随后保持高电平一段时间，再把SCL拉低、左移数据。循环8次，完成一个字节的数据部分。

##### 7.7.2 读ACK前一定先释放SDA

~~~c
IICBus_SetSDA(1u);
IICBus_Delay();
status = IICBus_RaiseSCL();
~~~

前8位由STM32发送，第9位由对方回应，所以STM32必须松开SDA。若主机自己一直拉低，会把没有设备的情况也误读成ACK。

SCL真正变高并等待稳定后，代码读取SDA：低返回 `IICBUS_OK`，高返回 `IICBUS_NACK`。随后拉低SCL，准备下一字节或停止。

##### 7.7.3 等待SCL要有出口

内部 `IICBus_RaiseSCL` 先释放SCL，再读取它是否升高。2ms内仍为低就返回 `IICBUS_SCL_TIMEOUT`，避免线路故障时程序一直卡在等待中。

从设备短暂保持SCL为低叫时钟延展，可以理解为“这一拍先等一下”。本例允许有限等待；如果目标器件需要更长时间，应按它的手册调整，不能把2ms当成所有IIC设备的统一限制。

##### 7.7.4 为什么出现NACK后还要STOP

NACK说明这一阶段对方没有接收。程序需要结束本笔传输，释放线路，再由主循环报告结果。因此 `Probe`、`Write`、`Read` 都统一走收尾函数。

不要在每个字节中间打印串口文字。本例先完成传输和STOP，再打印结果，读代码与抓取波形都更清楚。

#### 7.8 选做：把地址探测拆成四步

用7.10.5替换入口，模块与接线沿用。

程序先扫描；只有找到一个应答地址时，才把它作为本次跟踪对象。接着每秒完成一次手动探测，并打印：

- START的结果；
- 地址的8位发送结果；
- ACK检查结果；
- STOP的结果。

若找到零个或多个地址，先检查接线或设备组成，再复位。单个扫描结果也不能证明不存在同地址的两个设备；这一步用于本章已确认的单OLED线路。

**完整四步结束后才输出文字。**串口看到的四行是分步结果记录，不是把传输暂停四次。

有逻辑分析仪时，以START作为触发点，核对地址字节与第9拍。没有分析仪时，也能通过 `ADDR8=OK`、`ACK=NACK` 区分“8位已送出”和“对方已经应答”。

#### 7.9 把应答检查接回OLED写入

##### 7.9.1 OLED写入比探测多了什么

地址探测只到地址与ACK。实际OLED写入还要继续发送器件规定的协议字节。

| 传输位置 | 由谁发送 | 要从哪里确认 |
|---|---|---|
| START | STM32 | IIC规则 |
| 地址字节，写方向 | STM32 | 实际7位地址及地址接口约定 |
| 地址ACK | OLED | 第9拍读SDA |
| 控制字节，如该控制器需要 | STM32 | OLED控制器手册与已有驱动 |
| 控制字节ACK | OLED | 第9拍读SDA |
| 命令或显示数据 | STM32 | 对应控制器的命令表或显存格式 |
| 每字节ACK | OLED | 每个字节后检查 |
| STOP | STM32 | IIC规则 |

可以把地址看成“送到哪一家”，控制字节看成“这一包属于哪类内容”，命令或数据看成“包里的内容”。这些值要由实际OLED驱动决定，通信函数不能替设备猜。

本章没有把某个常见OLED的地址、控制字节或显示命令写成配套硬件的固定值。定稿这一节前，需要核对：**控制器型号与分辨率、模块供电和上拉、实际地址，以及基础篇 `OLED.c/.h`、`MyI2C.c/.h` 的接口**。

##### 7.9.2 一个可以复用的写入包装

下面是本章新增的演示函数，传入值均由实际OLED驱动提供。仅适用于已确认使用“一个控制字节＋一个内容字节”格式的控制器，不负责屏幕初始化，也不替代完整显示驱动。

~~~c
#include "IICBus.h"

/**
 * @brief  发送已核对的OLED控制字节和一个内容字节
 * @param  address7：未左移的实际7位设备地址
 * @param  control：控制器规定的控制字节
 * @param  value：实际命令或显示数据
 * @retval IICBus_Status：传输与收尾的结果
 */
IICBus_Status IICDemo_WriteOLEDByte(uint8_t address7,
                                   uint8_t control,
                                   uint8_t value)
{
    uint8_t packet[2];

    packet[0] = control;
    packet[1] = value;
    return IICBus_Write(address7, packet, 2u);
}
~~~

这段函数的作用是把两个**已经核对的字节**组成数组，再交给通用写接口。它返回NACK时，由上层停止本轮显示操作并记录错误；不会在错误后继续假装显示成功。

整合时先在已有OLED驱动中找到实际发送命令或数据的位置，再改造传输层。保留它已经跑通的初始化顺序、字库和显示格式；不要为了使用这个包装函数另猜一套OLED接口。

##### 7.9.3 函数调用与屏幕刷新之间的关系

| 层次 | 负责什么 |
|---|---|
| 应用层显示函数 | 决定显示哪些文字或数字 |
| OLED驱动 | 把内容转成该控制器需要的命令与显示数据 |
| IIC传输层 | 发送字节、检查应答、结束传输 |
| GPIO操作 | 控制SCL、SDA的实际电平 |

如果已有驱动先改内存显存、再集中刷新，更新文字以后还需要按它的实际接口刷新。地址有ACK但屏幕没有变化时，继续检查屏幕初始化、命令格式和刷新流程。

##### 7.9.4 选读：读取时为什么要NACK

对允许直接读取的设备，`IICBus_Read` 已经处理了下面的顺序：

| 要接收的字节 | 接收后STM32怎么回应 |
|---|---|
| 后面还要继续接收 | 发ACK，即 `IICBus_SendAck(0u)` |
| 最后一个字节 | 发NACK，即 `IICBus_SendAck(1u)`，然后STOP |

需要“先写寄存器地址，再重复起始读取”的设备，应按它的手册组织对应流程，不能直接把普通 `Read` 当成寄存器读取函数。OLED是否支持通过IIC读取，也必须看实际控制器；本章不要求用OLED做读取实验。

#### 7.10 完整代码

前三个入口共用下面两份总线文件，以及已有 `Serial.c/.h`、`Delay.c/.h`。`main.c` 每次只选一份。

##### 7.10.1 `Hardware/IICBus.h`

~~~c
#ifndef __IICBUS_H
#define __IICBUS_H

#include "stm32f10x.h"

/** @brief 本章总线操作结果，成功为0。 */
typedef enum
{
    IICBUS_OK = 0,
    IICBUS_NACK,
    IICBUS_SCL_TIMEOUT,
    IICBUS_BUS_BUSY,
    IICBUS_BAD_ARG,
    IICBUS_NOT_READY
} IICBus_Status;

IICBus_Status IICBus_Init(void);
void IICBus_GetLines(uint8_t *scl, uint8_t *sda);
const char *IICBus_StatusText(IICBus_Status status);

IICBus_Status IICBus_Start(void);
IICBus_Status IICBus_Stop(void);
IICBus_Status IICBus_SendByte(uint8_t data);
IICBus_Status IICBus_ReadAck(void);
IICBus_Status IICBus_ReceiveByte(uint8_t *data);
IICBus_Status IICBus_SendAck(uint8_t level);

IICBus_Status IICBus_Probe(uint8_t address7);
IICBus_Status IICBus_Write(uint8_t address7,
                          const uint8_t *data,
                          uint16_t length);
IICBus_Status IICBus_Read(uint8_t address7,
                         uint8_t *data,
                         uint16_t length);

#endif
~~~

##### 7.10.2 `Hardware/IICBus.c`

~~~c
#include "IICBus.h"

/* 引脚固定沿用培训开发板约定。 */
#define IICBUS_PORT           GPIOB
#define IICBUS_SCL_PIN        GPIO_Pin_8
#define IICBUS_SDA_PIN        GPIO_Pin_9

#define IICBUS_PHASE_US       10u
#define IICBUS_SCL_TIMEOUT_US 2000u

static uint32_t iic_phase_cycles = 0u;
static uint32_t iic_timeout_cycles = 0u;
static uint8_t iic_ready = 0u;

/**
 * @brief  控制开漏SCL：0拉低，1释放
 * @param  release：0或1
 * @retval 无
 */
static void IICBus_SetSCL(uint8_t release)
{
    if (release != 0u)
    {
        GPIO_SetBits(IICBUS_PORT, IICBUS_SCL_PIN);
    }
    else
    {
        GPIO_ResetBits(IICBUS_PORT, IICBUS_SCL_PIN);
    }
}

/**
 * @brief  控制开漏SDA：0拉低，1释放
 * @param  release：0或1
 * @retval 无
 */
static void IICBus_SetSDA(uint8_t release)
{
    if (release != 0u)
    {
        GPIO_SetBits(IICBUS_PORT, IICBUS_SDA_PIN);
    }
    else
    {
        GPIO_ResetBits(IICBUS_PORT, IICBUS_SDA_PIN);
    }
}

/**
 * @brief  读取SCL实际电平
 * @param  无
 * @retval 0为低，1为高
 */
static uint8_t IICBus_ReadSCL(void)
{
    return GPIO_ReadInputDataBit(IICBUS_PORT, IICBUS_SCL_PIN);
}

/**
 * @brief  读取SDA实际电平
 * @param  无
 * @retval 0为低，1为高
 */
static uint8_t IICBus_ReadSDA(void)
{
    return GPIO_ReadInputDataBit(IICBUS_PORT, IICBUS_SDA_PIN);
}

/**
 * @brief  等待一个时序阶段；由已经启动的DWT周期计数计时
 * @param  无
 * @retval 无；只能在计时准备成功后使用
 */
static void IICBus_Delay(void)
{
    uint32_t start;

    start = DWT->CYCCNT;
    while ((uint32_t)(DWT->CYCCNT - start) < iic_phase_cycles)
    {
        /* 无符号相减允许计数器自然回绕。 */
    }
}

/**
 * @brief  释放SCL并等到线上真正变高
 * @param  无
 * @retval OK或SCL_TIMEOUT
 */
static IICBus_Status IICBus_RaiseSCL(void)
{
    uint32_t start;

    IICBus_SetSCL(1u);
    start = DWT->CYCCNT;

    while (IICBus_ReadSCL() == 0u)
    {
        if ((uint32_t)(DWT->CYCCNT - start) >= iic_timeout_cycles)
        {
            return IICBUS_SCL_TIMEOUT;
        }
    }

    return IICBUS_OK;
}

/**
 * @brief  判断是否为本章支持的普通7位地址
 * @param  address7：未左移的地址
 * @retval 1有效，0无效；本章跳过保留地址
 */
static uint8_t IICBus_AddressValid(uint8_t address7)
{
    return (uint8_t)((address7 >= 0x08u) && (address7 <= 0x77u));
}

/**
 * @brief  初始化GPIO与内部计时，检查总线空闲
 * @param  无
 * @retval IICBus_Status
 */
IICBus_Status IICBus_Init(void)
{
    GPIO_InitTypeDef gpio;
    IICBus_Status status;
    uint32_t cycles_per_us;
    uint32_t first_count;
    uint8_t i;

    iic_ready = 0u;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);

    /* 先设置释放状态，再切入普通开漏输出。 */
    GPIO_SetBits(IICBUS_PORT, IICBUS_SCL_PIN | IICBUS_SDA_PIN);

    GPIO_StructInit(&gpio);
    gpio.GPIO_Pin = IICBUS_SCL_PIN | IICBUS_SDA_PIN;
    gpio.GPIO_Speed = GPIO_Speed_2MHz;
    gpio.GPIO_Mode = GPIO_Mode_Out_OD;
    GPIO_Init(IICBUS_PORT, &gpio);

    SystemCoreClockUpdate();
    if (SystemCoreClock == 0u)
    {
        return IICBUS_NOT_READY;
    }

    /* 本模块使用核心周期计数，不占用SysTick或TIM外设。 */
    CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;

    if ((DWT->CTRL & DWT_CTRL_NOCYCCNT_Msk) != 0u)
    {
        return IICBUS_NOT_READY;
    }

    /* 保留原计数，不重置其他模块可能正在读取的时间基准。 */
    DWT->CTRL |= DWT_CTRL_CYCCNTENA_Msk;
    __DSB();
    __ISB();

    first_count = DWT->CYCCNT;
    for (i = 0u; i < 32u; i++)
    {
        __NOP();
    }
    if (DWT->CYCCNT == first_count)
    {
        return IICBUS_NOT_READY;
    }

    cycles_per_us = (SystemCoreClock + 999999u) / 1000000u;
    iic_phase_cycles = cycles_per_us * IICBUS_PHASE_US;
    iic_timeout_cycles = cycles_per_us * IICBUS_SCL_TIMEOUT_US;
    iic_ready = 1u;

    IICBus_SetSDA(1u);
    status = IICBus_RaiseSCL();
    if (status != IICBUS_OK)
    {
        return status;
    }

    IICBus_Delay();
    if (IICBus_ReadSDA() == 0u)
    {
        return IICBUS_BUS_BUSY;
    }

    return IICBUS_OK;
}

/**
 * @brief  查看当前线路实际电平
 * @param  scl：保存SCL电平的变量地址，允许为空
 * @param  sda：保存SDA电平的变量地址，允许为空
 * @retval 无；在本模块初始化配置引脚后调用
 */
void IICBus_GetLines(uint8_t *scl, uint8_t *sda)
{
    if (scl != 0)
    {
        *scl = IICBus_ReadSCL();
    }
    if (sda != 0)
    {
        *sda = IICBus_ReadSDA();
    }
}

/**
 * @brief  将状态转换为串口输出文本
 * @param  status：本模块返回的状态
 * @retval 静态文本指针，不需要释放
 */
const char *IICBus_StatusText(IICBus_Status status)
{
    switch (status)
    {
        case IICBUS_OK:          return "OK";
        case IICBUS_NACK:        return "NACK";
        case IICBUS_SCL_TIMEOUT: return "SCL_TIMEOUT";
        case IICBUS_BUS_BUSY:    return "BUS_BUSY";
        case IICBUS_BAD_ARG:     return "BAD_ARG";
        case IICBUS_NOT_READY:   return "NOT_READY";
        default:                return "UNKNOWN";
    }
}

/**
 * @brief  产生起始；在合法字节边界调用也可产生重复起始
 * @param  无
 * @retval IICBus_Status
 */
IICBus_Status IICBus_Start(void)
{
    IICBus_Status status;

    if (iic_ready == 0u)
    {
        return IICBUS_NOT_READY;
    }

    IICBus_SetSDA(1u);
    IICBus_Delay();

    status = IICBus_RaiseSCL();
    if (status != IICBUS_OK)
    {
        return status;
    }

    IICBus_Delay();
    if (IICBus_ReadSDA() == 0u)
    {
        return IICBUS_BUS_BUSY;
    }

    /* SCL保持高时，让SDA下降形成START。 */
    IICBus_SetSDA(0u);
    IICBus_Delay();
    IICBus_SetSCL(0u);

    return IICBUS_OK;
}

/**
 * @brief  尝试产生停止并检查SDA是否已经释放
 * @param  无
 * @retval IICBus_Status
 */
IICBus_Status IICBus_Stop(void)
{
    IICBus_Status status;

    if (iic_ready == 0u)
    {
        return IICBUS_NOT_READY;
    }

    IICBus_SetSCL(0u);
    IICBus_SetSDA(0u);
    IICBus_Delay();

    status = IICBus_RaiseSCL();
    if (status != IICBUS_OK)
    {
        /* SCL故障时仍释放主机输出，但不能保证形成有效STOP。 */
        IICBus_SetSDA(1u);
        return status;
    }

    IICBus_Delay();
    IICBus_SetSDA(1u);
    IICBus_Delay();

    if (IICBus_ReadSDA() == 0u)
    {
        return IICBUS_BUS_BUSY;
    }

    return IICBUS_OK;
}

/**
 * @brief  发送一个字节的8个数据位，不包含第9拍
 * @param  data：完整8位字节
 * @retval IICBus_Status；成功后仍需调用ReadAck
 */
IICBus_Status IICBus_SendByte(uint8_t data)
{
    IICBus_Status status;
    uint8_t i;

    if (iic_ready == 0u)
    {
        return IICBUS_NOT_READY;
    }

    IICBus_SetSCL(0u);

    for (i = 0u; i < 8u; i++)
    {
        IICBus_SetSDA((uint8_t)((data & 0x80u) != 0u));
        IICBus_Delay();

        status = IICBus_RaiseSCL();
        if (status != IICBUS_OK)
        {
            IICBus_SetSDA(1u);
            return status;
        }

        IICBus_Delay();
        IICBus_SetSCL(0u);
        data = (uint8_t)(data << 1);
    }

    return IICBUS_OK;
}

/**
 * @brief  产生第9拍并接收ACK；不会无限等待SDA变低
 * @param  无
 * @retval OK表示ACK，NACK表示无应答，或返回线路错误
 */
IICBus_Status IICBus_ReadAck(void)
{
    IICBus_Status status;
    uint8_t level;

    if (iic_ready == 0u)
    {
        return IICBUS_NOT_READY;
    }

    IICBus_SetSCL(0u);
    IICBus_SetSDA(1u);
    IICBus_Delay();

    status = IICBus_RaiseSCL();
    if (status != IICBUS_OK)
    {
        return status;
    }

    IICBus_Delay();
    level = IICBus_ReadSDA();
    IICBus_SetSCL(0u);

    return (level == 0u) ? IICBUS_OK : IICBUS_NACK;
}

/**
 * @brief  接收一个字节，不包含随后由主机发送的应答拍
 * @param  data：保存结果的有效变量地址
 * @retval IICBus_Status；成功后再调用SendAck
 */
IICBus_Status IICBus_ReceiveByte(uint8_t *data)
{
    IICBus_Status status;
    uint8_t value = 0u;
    uint8_t i;

    if (data == 0)
    {
        return IICBUS_BAD_ARG;
    }
    if (iic_ready == 0u)
    {
        return IICBUS_NOT_READY;
    }

    IICBus_SetSCL(0u);
    IICBus_SetSDA(1u);

    for (i = 0u; i < 8u; i++)
    {
        IICBus_Delay();

        status = IICBus_RaiseSCL();
        if (status != IICBUS_OK)
        {
            return status;
        }

        IICBus_Delay();
        value = (uint8_t)((value << 1) | IICBus_ReadSDA());
        IICBus_SetSCL(0u);
    }

    *data = value;
    return IICBUS_OK;
}

/**
 * @brief  主机接收数据后发送第9拍的ACK或NACK
 * @param  level：0为ACK，1为NACK
 * @retval IICBus_Status
 */
IICBus_Status IICBus_SendAck(uint8_t level)
{
    IICBus_Status status;

    if (level > 1u)
    {
        return IICBUS_BAD_ARG;
    }
    if (iic_ready == 0u)
    {
        return IICBUS_NOT_READY;
    }

    IICBus_SetSCL(0u);
    IICBus_SetSDA(level);
    IICBus_Delay();

    status = IICBus_RaiseSCL();
    if (status != IICBUS_OK)
    {
        IICBus_SetSDA(1u);
        return status;
    }

    IICBus_Delay();
    IICBus_SetSCL(0u);
    IICBus_SetSDA(1u);

    return IICBUS_OK;
}

/**
 * @brief  内部组合操作：发8位，再检查第9拍
 * @param  data：要发送的字节
 * @retval IICBus_Status
 */
static IICBus_Status IICBus_WriteByteChecked(uint8_t data)
{
    IICBus_Status status;

    status = IICBus_SendByte(data);
    if (status != IICBUS_OK)
    {
        return status;
    }

    return IICBus_ReadAck();
}

/**
 * @brief  内部统一收尾，优先报告STOP失败
 * @param  transfer_status：前面传输的结果
 * @retval STOP失败则返回收尾错误，否则返回原传输结果
 */
static IICBus_Status IICBus_Finish(IICBus_Status transfer_status)
{
    IICBus_Status stop_status;

    stop_status = IICBus_Stop();
    if (stop_status != IICBUS_OK)
    {
        return stop_status;
    }

    return transfer_status;
}

/**
 * @brief  用写方向地址探测设备是否应答
 * @param  address7：未左移的普通7位地址
 * @retval IICBus_Status；不会追加命令或数据字节
 */
IICBus_Status IICBus_Probe(uint8_t address7)
{
    IICBus_Status status;

    if (IICBus_AddressValid(address7) == 0u)
    {
        return IICBUS_BAD_ARG;
    }

    status = IICBus_Start();
    if (status != IICBUS_OK)
    {
        return status;
    }

    status = IICBus_WriteByteChecked((uint8_t)(address7 << 1));
    return IICBus_Finish(status);
}

/**
 * @brief  写一组字节，每个字节都检查应答
 * @param  address7：未左移的普通7位地址
 * @param  data：有效数组地址
 * @param  length：要写的有效字节数，必须大于0
 * @retval IICBus_Status；返回前已经尝试STOP
 */
IICBus_Status IICBus_Write(uint8_t address7,
                          const uint8_t *data,
                          uint16_t length)
{
    IICBus_Status status;
    uint16_t i;

    if ((IICBus_AddressValid(address7) == 0u) ||
        (data == 0) || (length == 0u))
    {
        return IICBUS_BAD_ARG;
    }

    status = IICBus_Start();
    if (status != IICBUS_OK)
    {
        return status;
    }

    status = IICBus_WriteByteChecked((uint8_t)(address7 << 1));

    for (i = 0u; (i < length) && (status == IICBUS_OK); i++)
    {
        status = IICBus_WriteByteChecked(data[i]);
    }

    return IICBus_Finish(status);
}

/**
 * @brief  直接读一组字节，最后一个字节后发NACK
 * @param  address7：未左移的普通7位地址
 * @param  data：容量不少于length的可写数组
 * @param  length：要接收的字节数，必须大于0
 * @retval IICBus_Status；失败时数组可能只写入部分结果
 */
IICBus_Status IICBus_Read(uint8_t address7,
                         uint8_t *data,
                         uint16_t length)
{
    IICBus_Status status;
    uint16_t i;

    if ((IICBus_AddressValid(address7) == 0u) ||
        (data == 0) || (length == 0u))
    {
        return IICBUS_BAD_ARG;
    }

    status = IICBus_Start();
    if (status != IICBUS_OK)
    {
        return status;
    }

    status = IICBus_WriteByteChecked((uint8_t)((address7 << 1) | 1u));

    for (i = 0u; (i < length) && (status == IICBUS_OK); i++)
    {
        status = IICBus_ReceiveByte(&data[i]);
        if (status == IICBUS_OK)
        {
            /* 后面还要继续读则ACK，最后一个字节则NACK。 */
            status = IICBus_SendAck((uint8_t)((i + 1u == length) ? 1u : 0u));
        }
    }

    return IICBus_Finish(status);
}
~~~

##### 7.10.3 完整入口一：空闲线路检查

~~~c
#include "stm32f10x.h"
#include "misc.h"
#include "Serial.h"
#include "Delay.h"
#include "IICBus.h"

/**
 * @brief  初始化后反复显示两根线的实际电平
 * @param  无
 * @retval 正常运行时不返回
 */
int main(void)
{
    IICBus_Status status;
    uint8_t scl;
    uint8_t sda;

    /* 独立示例沿用提4的分组；整合工程由全局统一设置。 */
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    Serial_Init();

    status = IICBus_Init();
    Serial_Printf("IIC idle-line check\r\n");
    Serial_Printf("init=%s\r\n", IICBus_StatusText(status));

    while (1)
    {
        IICBus_GetLines(&scl, &sda);
        Serial_Printf("SCL=%u SDA=%u\r\n",
                      (unsigned int)scl,
                      (unsigned int)sda);
        Delay_ms(1000);
    }
}
~~~

##### 7.10.4 完整入口二：地址扫描

~~~c
#include "stm32f10x.h"
#include "misc.h"
#include "Serial.h"
#include "Delay.h"
#include "IICBus.h"

/**
 * @brief  扫描本章普通7位地址并打印应答结果
 * @param  无
 * @retval 正常完成为OK，线路异常时返回错误并停止本轮
 */
static IICBus_Status ScanAddresses(void)
{
    IICBus_Status status;
    uint8_t address7;
    uint16_t count = 0u;

    Serial_Printf("SCAN BEGIN\r\n");

    for (address7 = 0x08u; address7 <= 0x77u; address7++)
    {
        status = IICBus_Probe(address7);

        if (status == IICBUS_OK)
        {
            count++;
            Serial_Printf("ACK addr7=0x%02X\r\n",
                          (unsigned int)address7);
        }
        else if (status != IICBUS_NACK)
        {
            Serial_Printf("SCAN STOP addr7=0x%02X status=%s\r\n",
                          (unsigned int)address7,
                          IICBus_StatusText(status));
            return status;
        }
    }

    if (count == 0u)
    {
        Serial_Printf("No ACK address found.\r\n");
    }

    Serial_Printf("SCAN END count=%u\r\n", (unsigned int)count);
    return IICBUS_OK;
}

/**
 * @brief  重复扫描OLED实验线路，出现线路错误后停止
 * @param  无
 * @retval 正常运行时不返回
 */
int main(void)
{
    IICBus_Status status;

    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    Serial_Init();

    status = IICBus_Init();
    Serial_Printf("IIC address scan\r\n");
    Serial_Printf("init=%s\r\n", IICBus_StatusText(status));

    if (status != IICBUS_OK)
    {
        while (1)
        {
            /* 先处理硬件或计时问题，再复位。 */
        }
    }

    while (1)
    {
        status = ScanAddresses();
        if (status != IICBUS_OK)
        {
            while (1)
            {
                /* 不在异常线路上持续重试。 */
            }
        }

        Delay_ms(2000);
    }
}
~~~

##### 7.10.5 完整入口三：分步探测一个地址，选做

~~~c
#include "stm32f10x.h"
#include "misc.h"
#include "Serial.h"
#include "Delay.h"
#include "IICBus.h"

/**
 * @brief  扫描并记录唯一应答地址
 * @param  address7：保存扫描结果的有效变量地址
 * @retval 1表示唯一地址，0表示没有、多个或发生线路错误
 */
static uint8_t FindSingleAddress(uint8_t *address7)
{
    IICBus_Status status;
    uint8_t address;
    uint16_t count = 0u;

    for (address = 0x08u; address <= 0x77u; address++)
    {
        status = IICBus_Probe(address);
        if (status == IICBUS_OK)
        {
            *address7 = address;
            count++;
        }
        else if (status != IICBUS_NACK)
        {
            Serial_Printf("scan error=%s\r\n",
                          IICBus_StatusText(status));
            return 0u;
        }
    }

    Serial_Printf("ACK address count=%u\r\n", (unsigned int)count);
    return (uint8_t)(count == 1u);
}

/**
 * @brief  拆开完成一次探测，结束后再打印四步结果
 * @param  address7：本轮扫描得到的7位地址
 * @retval 1表示没有线路错误，0表示发生错误
 */
static uint8_t ProbeBySteps(uint8_t address7)
{
    IICBus_Status start_status;
    IICBus_Status byte_status;
    IICBus_Status ack_status;
    IICBus_Status stop_status;

    start_status = IICBus_Start();
    if (start_status != IICBUS_OK)
    {
        Serial_Printf("START=%s\r\n",
                      IICBus_StatusText(start_status));
        return 0u;
    }

    byte_status = IICBus_SendByte((uint8_t)(address7 << 1));
    if (byte_status == IICBUS_OK)
    {
        ack_status = IICBus_ReadAck();
    }
    else
    {
        ack_status = IICBUS_NOT_READY;
    }

    stop_status = IICBus_Stop();

    Serial_Printf("addr7=0x%02X addr8_write=0x%02X\r\n",
                  (unsigned int)address7,
                  (unsigned int)(address7 << 1));
    Serial_Printf("START=%s\r\n", IICBus_StatusText(start_status));
    Serial_Printf("ADDR8=%s\r\n", IICBus_StatusText(byte_status));
    if (byte_status == IICBUS_OK)
    {
        Serial_Printf("ACK=%s\r\n", IICBus_StatusText(ack_status));
    }
    else
    {
        Serial_Printf("ACK=SKIPPED\r\n");
    }
    Serial_Printf("STOP=%s\r\n", IICBus_StatusText(stop_status));

    if ((byte_status != IICBUS_OK) || (stop_status != IICBUS_OK))
    {
        return 0u;
    }

    return (uint8_t)((ack_status == IICBUS_OK) ||
                     (ack_status == IICBUS_NACK));
}

/**
 * @brief  找到唯一应答地址后，重复观察四步探测结果
 * @param  无
 * @retval 正常运行时不返回
 */
int main(void)
{
    IICBus_Status status;
    uint8_t address7 = 0u;

    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    Serial_Init();

    status = IICBus_Init();
    Serial_Printf("IIC probe by steps\r\n");
    Serial_Printf("init=%s\r\n", IICBus_StatusText(status));

    if (status != IICBUS_OK)
    {
        while (1)
        {
        }
    }

    if (FindSingleAddress(&address7) == 0u)
    {
        Serial_Printf("Check device count, then reset.\r\n");
        while (1)
        {
        }
    }

    while (1)
    {
        if (ProbeBySteps(address7) == 0u)
        {
            Serial_Printf("Line error: check wiring, then reset.\r\n");
            while (1)
            {
            }
        }

        Delay_ms(1000);
    }
}
~~~

#### 7.11 软件IIC与硬件IIC，先分清职责

| 比较项 | 本章软件IIC | STM32硬件IIC |
|---|---|---|
| 谁产生时序 | CPU调用GPIO函数 | I2C外设按配置工作 |
| 引脚选择 | 可选择合适GPIO，本章固定PB8、PB9 | 使用芯片规定的引脚或允许的重映射 |
| 引脚模式 | `GPIO_Mode_Out_OD` | `GPIO_Mode_AF_OD` |
| 处理器工作 | CPU逐拍操作，本章接口同步等待 | 外设产生位时序，CPU仍需组织传输并处理状态 |
| 初学用途 | 直观看到线的变化与函数关系 | 掌握设备协议后再学习外设事件与传输管理 |

在STM32F103C8T6上，I2C1默认使用PB6、PB7；使用本章PB8、PB9时，硬件方案需要AFIO时钟与I2C1重映射。软件方案只操作GPIO，不需要这个重映射。

硬件IIC也不会自动知道OLED的地址、命令或显示内容。DMA能进一步承担适用的字节搬运，但不代替设备协议。硬件IIC配置放在后续扩展，本章先把软件传输跑通。

#### 7.12 一主多从与现象验证

##### 7.12.1 多个设备怎样接同一组线

扩展接入第二个IIC设备时，两根信号线共用：

| 共用线路 | 对应连接 |
|---|---|
| SCL | PB8与各设备的SCL并在同一根时钟线上 |
| SDA | PB9与各设备的SDA并在同一根数据线上 |
| GND | STM32与各设备共地 |
| 电源与上拉 | 逐个核对器件供电与共享信号电平，检查并联上拉 |

STM32通过地址选择设备，所以不需要为每块IIC设备另外增加片选线。两个设备地址相同时，可能同时应答；扫描出一个地址不代表线上只有一块设备。能改地址的设备按手册调整，不能改的需要改变硬件连接方案。

本章只使用单主机，不包含多主机仲裁。软件IIC能搬到合适GPIO，也不意味着任意电压和任意长度的连线都能直接使用。

##### 7.12.2 有逻辑分析仪时看什么

分析仪的电平兼容性先核对，再把两个输入通道分别接PB8、PB9，分析仪GND接开发板GND，不用分析仪电源脚给开发板供电。

选择I2C协议解码，以START触发。采样率应明显高于实际时钟频率；本章低速实验可从1MHz采样起步，再按捕获结果调整。

| 检查位置 | 应看到什么 |
|---|---|
| START | SCL高时，SDA下降 |
| 地址数据 | 8拍，高位先发送 |
| 地址显示 | 确认解码器显示7位地址，还是包含读写位的8位字节 |
| ACK | 第9拍SDA为低；正常数据阶段SDA在SCL高时稳定 |
| STOP | SCL高时，SDA上升 |
| OLED实际写入 | 地址后还出现驱动规定的控制、命令或数据字节，以及各自应答 |

地址扫描会连续访问多个地址。有ACK的地址和没有设备的地址都会出现在捕获记录中，不要把大量NACK直接当成扫描程序错误。

##### 7.12.3 没有逻辑分析仪时怎样检查

1. 实验一确认空闲两线为高。
2. 实验二检查是否出现应答地址。
3. 选做实验三确认四步结果，并核对7位地址与写地址字节的计算。
4. 后续接回已确认的OLED驱动，再观察实际显示。

串口返回OK与屏幕正常显示是有用的检查结果，但不能代替对波形时序、上升时间的测量。文中的串口文字均为预期格式，具体地址与实测截图由配套硬件验证后记录。

#### 7.13 常见问题与练习

##### 7.13.1 排错表

| 现象 | 优先检查 | 处理方向 |
|---|---|---|
| `SCL=0`或 `SDA=0` | 接线、供电、短路、上拉、设备状态 | 断电核对后再上电，先通过空闲检查 |
| 扫描结果始终为0 | PB8/PB9是否接反、模块接口模式、共地与电源 | 按信号名核对，确认确实使用IIC模式 |
| 所有地址都出现ACK | SDA是否被主机或硬件一直拉低 | 检查开漏和ACK前释放；本例通常先报告BUS_BUSY |
| `SCL_TIMEOUT` | SCL上拉、短路、从设备保持低电平 | 排除线路问题，再按器件手册核对延展需求 |
| `BUS_BUSY` | SDA被持续拉低 | 查模块状态和数据线；本章不自动强行恢复总线 |
| `NOT_READY` | 模板时钟、CMSIS、DWT周期计数是否正常 | 检查系统启动与内部计时配置，不绕过检查继续发送 |
| `BAD_ARG` | 是否传入左移地址、空指针、0长度 | 高层传7位地址，传有效数组和有效数量 |
| 地址有ACK，OLED仍不显示 | 控制器、初始化、控制字节、显示数据、刷新方式 | 返回已有OLED驱动逐项核对 |
| `ADDR8=OK`但 `ACK=NACK` | 实际地址、设备供电、接口模式 | 8位发出不等于目标存在或已经接收 |
| 扫描出来的值与驱动不同 | 一处是7位，一处是左移后的地址字节 | 查各自参数约定，只按要求左移一次 |
| 联合工程才出现异常 | 两套驱动、别的模块是否占用PB8/PB9 | 统一总线所有权与初始化 |
| 刷新很慢 | 软件逐拍等待、重复发送、实际刷新字节数 | 先保证正确，再减少冗余传输；硬件IIC放在扩展 |
| `IICBus_*`未定义 | `.c`是否加入Keil工程 | 文件放进目录后还要加入分组 |
| 找不到 `IICBus.h` | 文件位置和头文件搜索路径 | 检查Hardware路径 |
| 提示 `DWT`或相关宏未定义 | 是否使用配套STM32F103 CMSIS核心头文件 | 核对标准工程，不混用其他内核的核心文件 |
| 电脑没有输出或出现乱码 | CH340端口、PA9→RXD、共地、9600参数 | 回到第6章的串口验证步骤 |

##### 7.13.2 三个小练习

1. 在实验二中增加“本轮扫描编号”，确认每一轮都有开始与结束记录。
2. 找到配套OLED驱动中的地址常量，判断它保存的是7位地址还是地址字节，并用本次扫描结果核对。
3. 在已确认的OLED写入记录中，圈出START、地址、ACK、控制或数据、STOP，再找到负责各段的函数。

有逻辑分析仪时，再尝试捕获一笔NACK记录，与ACK记录比较第9拍。改动硬件连接前先断电。

##### 7.13.3 官方资料，按需选读

- [NXP《I²C-bus specification and user manual》UM10204](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)：本章对应3.1.1、3.1.3～3.1.6、3.1.10及3.1.12，查开漏、数据、起止、应答与地址。
- [ST《RM0008参考手册》](https://www.st.com/resource/en/reference_manual/cd00171190-stm32f101-103-105-107-stm32f100-series-armbased-32bit-mcus-stmicroelectronics.pdf)：查GPIO输入／输出、I2C1引脚重映射和I2C外设。
- [TI《I2C Bus Pullup Resistor Calculation》SLVA689](https://www.ti.com/lit/pdf/slva689)：上拉阻值与总线电容、上升时间的关系，阻值计算选读。
- [Arm《Cortex-M3 Technical Reference Manual》](https://www.keil.com/dd/docs/datashts/arm/cortex_m3/r2p0/ddi0337g_cortex_m3_r2p0_trm.pdf)：内部周期计数的依据，DWT细节选读。
- [Arm CMSIS Cortex-M3核心头文件](https://github.com/ARM-software/CMSIS_4/blob/master/CMSIS/Include/core_cm3.h)：查看 `DWT`、`CoreDebug` 与相关位定义；工程仍使用配套标准库自带的CMSIS文件。

先完成两项核心实验，能够接线、找到应答地址、区分地址字节和ACK，就达到了本章的第一阶段目标。

