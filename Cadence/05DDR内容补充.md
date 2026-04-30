# LPDDR4 / DDR 初学者学习笔记

# 1. DDR / LPDDR4 是什么？

## 1.1 DDR 是什么？

DDR 全称：

```text
Double Data Rate
双倍数据率
```

意思是：

> 在时钟的上升沿和下降沿都可以传输数据。

普通 SDR 一般一个时钟周期传一次数据：

```text
一个周期传 1 次
```

DDR 一个时钟周期可以传两次：

```text
上升沿传一次
下降沿传一次
```

所以叫 **双倍数据率**。

---

## 1.2 LPDDR4 是什么？

LPDDR4 全称：

```text
Low Power DDR4
低功耗 DDR4
```

---

# 2. LPDDR4 系统整体结构

一个 LPDDR4 系统通常由三部分组成：

```text
SoC 内部 DDR Controller
SoC 内部 DDR PHY
外部 LPDDR4 内存颗粒
```

可以理解成：

```text
┌──────────────────────────┐
│           SoC            │
│                          │
│  CPU / GPU / NPU / ISP   │
│            │             │
│            ▼             │
│     DDR Controller       │
│            │             │
│            ▼             │
│         DDR PHY          │
└────────────┬─────────────┘
             │
             │ CK / CA / DQ / DQS / DM
             ▼
┌──────────────────────────┐
│       外部 LPDDR4 芯片    │
└──────────────────────────┘
```

---

# 3. DDR Controller 是什么？

## 3.1 DDR Controller 在哪里？

**DDR Controller 通常在 SoC 里面。**

它不是外部 LPDDR4 芯片里的模块，而是 SoC 内部负责管理内存访问的控制器。

---

## 3.2 DDR Controller 负责什么？

CPU、GPU、NPU、ISP 等模块都可能访问内存。

例如 CPU 想读一个地址：

```text
CPU 读地址 0x8000_1234
```

CPU 看到的是一个线性地址，但 LPDDR4 内部并不是简单的一维数组，而是 Bank / Row / Column 结构。

所以 DDR Controller 要做地址转换和命令调度：

```text
线性地址
   ↓
Channel / Rank / Bank / Row / Column
   ↓
DDR 命令序列
```

DDR Controller 负责：

```text
地址映射
Bank / Row / Column 管理
读写命令调度
ACT / READ / WRITE / PRE 控制
刷新 Refresh 管理
低功耗管理
多主设备访问仲裁
QoS 调度
```

简单理解：

> **DDR Controller 是内存访问的调度员。**

## 3.3 名词解释
### 3.3.1 Channel（通道）
含义：独立的数据通道，有自己的 DQ/DQS/CA 信号组。

LPDDR4 常见双通道设计：


- Channel A：DQ0~DQ15，独立 DQS，独立 CA
- Channel B：DQ0~DQ15，独立 DQS，独立 CA
- 两个 Channel 可以同时工作
   - Channel A 在做读写时，Channel B 也可以做读写。

如果每个 Channel 是 16bit，合起来就是 32bit 总线宽度。

### 3.3.2 Rank (阵列组)
共享同一组数据总线、但被**不同 CS（片选信号）**选中的一组芯片或内部阵列。


- CS0 = 0 → Rank 0 响应命令
- CS1 = 0 → Rank 1 响应命令

多个 Rank 共享 DQ/DQS 走线，但同一时刻只有一个 Rank 在"听话"。Rank 的好处是增加容量而不增加总线宽度。

### 3.3.3 Bank（存储体）
含义：芯片内部独立的分区，每个 Bank 有自己的 Row Buffer 和存储阵列。

一个 LPDDR4 芯片内部：
```
    Bank0：1024 Row × 64 Column
    Bank1：1024 Row × 64 Column
    Bank2：1024 Row × 64 Column
    Bank3：1024 Row × 64 Column
    ...
```
Bank 的核心意义是并行隐藏延迟：
- Bank0 正在等 tRCD（开行后等待），此时可以操作 Bank1
   - 从发出 ACT 打开一行，到可以发 READ/WRITE 访问这行中的列，中间必须等待的时间。
- Bank2 在执行 PRE（关行），Bank3 可以同时做 READ

如果没有多 Bank，每次 ACT → READ → PRE 只能串行等，效率极低。

类比：仓库内部有多层楼，每层有独立的搬运台。一层在备货时，另一层可以同时出货。

### 3.3.4 Row（行）
含义：Bank 内部的一整行存储单元。

DRAM 的读/写不是直接访问单个 bit，而是先打开一整行，整行数据被拉到 Row Buffer（也叫 Sense Amplifier）。

Bank1 内部：

```
        C0  C1  C2  C3  ...  C63
R0     [  ][  ][  ][  ]     [  ]
R1     [  ][  ][  ][  ]     [  ]
...
R1023  [  ][  ][  ][  ]     [  ]
```

- 要访问 Row512 → ACT Bank1, Row512 → 整行进入 Row Buffer
- 一个 Bank 同时只能打开一行。

类比：一排货架。你要拿货架上的某个东西，必须先把整排货架搬到前台（Row Buffer）。搬一次很慢，但搬完之后在这排货架上拿东西就很快了。

### 3.3.5 Column（列）
含义：Row Buffer 中的具体位置，是最终读写数据的偏移。

Row 已经打开进入 Row Buffer 后，READ/WRITE 命令指定 Column 地址，从这个位置开始 Burst 传输。

```
Row Buffer（当前打开的 Row512）：

    C0   C1   C2   C3   ... C63
    [A]  [B]  [C]  [D]       [X]

READ Column2, Burst Length 4 → 输出 C, D, E, F
```

同一 Row 内访问不同 Column 是很快的（Row Hit），不需要重新 ACT。

类比：前台已经摆好一整排货了，你要的是这排货架上的第 N 个位置的东西。

### 3.3.6 完整访问路径示例
假设 CPU 要读 0x8000_1234，Controller 拆解后：
```
Channel 1  →  用 CH_B 的数据总线
Rank 0     →  CS0 选中
Bank 3     →  操作 Bank3
Row 512    →  ACT Bank3, Row512（整行调入 Row Buffer）
Column 32  →  READ Column32（从 Row Buffer 的这个位置开始读）
```

时间线：
```
ACT Bank3, Row512    ← 打开第 512 排货架
    ↓ 等 tRCD        ← 货架搬运到前台需要时间
READ Column32        ← 从前台第 32 个位置开始取货
    ↓ 等 CL          ← 取货指令到货物出来有延迟
DQ 上出现数据         ← 货物连续输出（Burst）
```

---

# 4. DDR PHY 是什么？

DDR PHY 也通常在 SoC 里面。

Controller 更偏逻辑调度，PHY 更偏高速电气接口。

---

## 4.1 DDR PHY 负责什么？

DDR PHY 负责真正把高速信号发出去、收回来。

它负责：

```text
驱动 CK / CA / DQ / DQS
接收 DQ / DQS
调整输入输出延迟
执行读写训练
VREF 调整
阻抗校准
驱动强度控制
采样窗口调整
```

简单理解：

> **DDR Controller 是脑子，DDR PHY 是手脚。**

---

## 4.2 Controller、PHY、LPDDR4 的关系

| 模块            | 位置     | 作用     |
| -------------- | ------   | -------- |
| DDR Controller | SoC 内部 | 决定怎么访问内存 |
| DDR PHY        | SoC 内部 | 负责高速信号收发 |
| LPDDR4 颗粒     | SoC 外部 | 真正存储数据   |

一句话：

> **Controller 决定做什么，PHY 负责怎么发信号，LPDDR4 负责存数据。**

---

# 5. 外部 LPDDR4 芯片里面有什么？

外部 LPDDR4 芯片一般没有完整意义上的 DDR Controller。

它内部主要有：

```text
命令解码器
Bank 阵列
Row / Column 存储阵列
Sense Amplifier
Row Buffer
模式寄存器
刷新电路
IO 接收 / 发送电路
```

它是被 SoC 控制器控制的一方。

可以理解为：

```text
SoC DDR Controller：主动调度者
LPDDR4 芯片：被访问的存储仓库
```

---

# 6. LPDDR4 引脚分类

LPDDR4 管脚很多，但初学者不要一个个死记，应该按功能分组理解。

主要分为：

```text
CK      主时钟
CA      命令 / 地址
DQ      数据
DQS     数据选通
DM/DMI  数据掩码 / 数据反转
CS      片选
CKE     时钟使能
RESET_n 复位
ZQ/RZQ  阻抗校准
VDD/VDDQ/VREF/VSS 电源和参考
```

---

# 7. CK：主时钟

常见命名：

```text
CK_t / CK_c
CKP / CKN
CLKP / CLKN
LPDDR4_CLKP / LPDDR4_CLKN
```

CK 是差分信号。

```
单端信号（普通信号）：
    一根线 + 地
    判断：线电压 > VREF → 1，< VREF → 0
    缺点：地上的噪声会直接影响判断

差分信号（CK/DQS）：
    两根线 P 和 N
    判断：P - N > 0 → 1，P - N < 0 → 0
```

---

## 7.1 CK 的作用

CK 用来同步：

```text
命令
地址
控制信号
内部状态切换
```

例如：

```text
ACT
READ
WRITE
PRECHARGE
REFRESH
MRW
MRR
```

这些命令都是根据 CK 的节奏被 LPDDR4 识别。

简单理解：

> **CK 是 DDR 系统的主节拍。**

但是要注意：

> **CK 主要管命令和地址，不是主要用来采样 DQ 数据。**

采样数据主要靠 DQS。

---

# 8. CA：命令 / 地址总线

LPDDR4 中常见：

```text
CA0 ~ CA5
```

有些原理图可能写成：

```text
A0 ~ A5
LPDDR4_A0 ~ LPDDR4_A5
```

---

## 8.1 CA 传什么？

CA 不是单纯地址线，而是命令和地址复用线。

它传输：

```text
命令 Command
Row 行地址
Column 列地址
Bank 相关地址
模式寄存器信息
```

常见命令：

```text
ACT     打开某一行
READ    读数据
WRITE   写数据
PRE     关闭当前行
REFRESH 刷新
MRW     写模式寄存器
MRR     读模式寄存器
```

初学阶段最重要的命令是：

```text
ACT  → 打开 Row
READ → 读 Column
WRITE → 写 Column
PRE → 关闭 Row
```

一句话：

> **CA 告诉 LPDDR4：我要做什么、访问哪里。命令和地址**

---

# 9. DQ：数据线

DQ 是真正传输数据的信号。

常见命名：

```text
DQ0
DQ1
...
DQ15
```

如果是双通道，可能有：

```text
CHA_DQ0 ~ CHA_DQ15
CHB_DQ0 ~ CHB_DQ15
```

---

## 9.1 DQ 的方向

DQ 是双向的。

写入时：

```text
控制器 → LPDDR4
```

读取时：

```text
LPDDR4 → 控制器
```

简单理解：

> **DQ 是真正的数据通道。**

---

# 10. DQS：数据选通信号

DQS 全称：

```text
Data Strobe
数据选通
```

常见命名：

```text
DQS0P / DQS0N
DQS1P / DQS1N
DQS_t / DQS_c
```

DQS 是差分信号。

---

## 10.1 DQS 的作用

DQS 用来采样 DQ。

可以这样记：

```text
DQ  = 数据是什么
DQS = 什么时候采数据
```

DQS 是 DQ 的采样节拍。

---

## 10.2 写时 DQS 谁发？

写操作方向是：

```text
控制器 → LPDDR4
```

写入时：

```text
控制器发送 DQ
控制器发送 DQS
LPDDR4 根据 DQS 采样 DQ
```

所以：

> **写时，DQ 和 DQS 都由控制器发出。**

---

## 10.3 读时 DQS 谁发？

读操作方向是：

```text
LPDDR4 → 控制器
```

读取时：

```text
LPDDR4 返回 DQ
LPDDR4 返回 DQS
控制器根据 DQS 采样 DQ
```

所以：

> **读时，DQ 和 DQS 都由 LPDDR4 返回。**

---

## 10.4 为什么要 DQS？

因为 DDR 很快，读数据时数据从 LPDDR4 返回控制器，会经过很多延迟：

```text
内存内部延迟
封装延迟
PCB 走线延迟
温度变化
电压变化
SoC 输入路径延迟
```

如果只用 CK 采 DQ，可能出现：

```text
CK 到了，但 DQ 还没稳定
```

或者：

```text
DQ 已经快切到下一个数据，CK 才采样
```

所以 DDR 让数据和采样节拍一起走：

```text
DQ 和 DQS 一起从发送端出来
接收端根据 DQS 采样 DQ
```

一句话：

> **DQS 是跟着数据一起走的采样时钟。**

---

# 11. CK 和 DQS 的关系

CK 和 DQS 都像“时钟”，但作用不同。

| 对比项   | CK                | DQS                |
| ------- | ----------------- | ------------------ |
| 中文     | 主时钟            | 数据选通            |
| 作用     | 同步命令 / 地址   | 采样 DQ 数据         |
| 作用对象 | CA、CS、CKE       | DQ、DM/DMI          |
| 方向     | 控制器 → LPDDR4   | 写时控制器发，读时 LPDDR4 发 |
| 是否持续  | 通常持续         | 数据 Burst 时活动     |
| 是否按 Byte Lane 分组 | 否   | 是                  |

核心记忆：

```text
CK  ：告诉内存做什么、什么时候开始
DQS ：告诉接收方什么时候采 DQ
```

一句话：

> **CK 是系统总指挥，DQS 是数据采样节拍器。**

---

# 12. DM / DMI 是什么？

DM 全称：

```text
Data Mask
数据掩码
```

LPDDR4 中常见的是 DMI，它可能承担两种功能：

```text
DM  = Data Mask，数据掩码
DBI = Data Bus Inversion，数据总线反转
```

---

## 12.1 DM 的作用

DM 主要用于写操作。

它告诉内存：

```text
这个字节要不要真正写进去
```

常见理解：

```text
DM = 0：写入
DM = 1：屏蔽，不写入
```

---

## 12.2 DM 举例

假设内存中有 4 个字节：

```text
Byte0 = AA
Byte1 = BB
Byte2 = CC
Byte3 = DD
```

现在只想修改 Byte1：

```text
Byte1 = 11
```

其他字节保持不变。

可以这样：

```text
DQ : AA  11  CC  DD
DM : 1   0   1   1
```

结果：

```text
Byte0 = AA   保持
Byte1 = 11   更新
Byte2 = CC   保持
Byte3 = DD   保持
```

所以：

> **DM 是写入时的“别写这个字节”开关。**

---

## 12.3 DMI 的 DBI 功能

DBI 的目的是减少数据线翻转，从而降低功耗、改善信号质量。

如果某个 Byte Lane 上很多位都要翻转，DBI 可以选择把数据整体反相传输，并用 DMI 告诉接收端：

```text
这一组数据被反转过
```

接收端再恢复回来。

简单理解：

> **DBI 是为了少翻转几根线。**

---

# 13. CS / CKE / RESET_n

## 13.1 CS：片选

CS 全称：

```text
Chip Select
```

常见：

```text
CS_n
```

低有效。

作用：

```text
选择哪个内存芯片或哪个 Rank 响应命令
```

---

## 13.2 CKE：时钟使能

CKE 全称：

```text
Clock Enable
```

作用：

```text
控制 LPDDR4 进入或退出低功耗状态
控制时钟相关状态
```

例如：

```text
正常工作
Power Down
Self Refresh
```

---

## 13.3 RESET_n：复位

RESET_n 是低有效复位。

```text
RESET_n = 0：复位
RESET_n = 1：释放复位
```

作用：

```text
上电初始化 LPDDR4
让内存进入已知状态
```

---

# 14. ZQ / RZQ：阻抗校准

ZQ 或 RZQ 是校准引脚，通常外接精密电阻到地。

---

## 14.1 为什么需要 ZQ？

DDR 是高速信号，必须考虑：

```text
驱动强度
输出阻抗
终端匹配
过冲
下冲
反射
振铃
```

如果驱动太强：

```text
过冲明显
反射严重
EMI 变差
```

如果驱动太弱：

```text
边沿太慢
眼图变小
采样困难
```

ZQ/RZQ 的作用是给芯片一个参考，让芯片校准 IO 驱动强度和阻抗。

简单理解：

> **ZQ/RZQ 是 DDR 信号强度的校准尺。**

---

# 15. VDD / VDDQ / VREF / VSS

## 15.1 VDD

VDD 是电源。

LPDDR4 通常有多组电源，例如：

```text
VDD1
VDD2
VDDQ
```

不同电源给不同内部模块或 IO 供电。

---

## 15.2 VDDQ

VDDQ 是 IO 相关电源。

它影响：

```text
DQ
DQS
DMI
部分 CA / 控制信号
```

---

## 15.3 VREF

VREF 是参考电压。

它用来判断输入信号是 0 还是 1：

```text
输入电压 > VREF → 判断为 1
输入电压 < VREF → 判断为 0
```

如果 VREF 不稳定，可能导致 0/1 误判。

所以 VREF 要求：

```text
稳定
低噪声
远离强干扰
滤波良好
```

---

## 15.4 VSS

VSS 是地。

它提供：

```text
电流回流路径
信号参考地
电源参考地
```

高速 DDR 信号非常依赖良好的回流路径。

---

# 16. Channel / Rank / Die / Byte Lane

这些概念经常出现，初学者容易混。

---

## 16.1 Channel

Channel 是独立的数据通道。

LPDDR4 常见双通道：

```text
Channel A
Channel B
```

例如每个 Channel 是 16bit：

```text
Channel A：16bit
Channel B：16bit
```

合起来就是：

```text
32bit 总线宽度
```

---

## 16.2 Byte Lane

Byte Lane 是按 8bit 分组的数据通道。

例如一个 16bit Channel：

```text
Byte Lane 0：DQ0  ~ DQ7
Byte Lane 1：DQ8  ~ DQ15
```

每个 Byte Lane 通常对应：

```text
一组 DQS
一根 DMI/DM
```

对应关系：

```text
DQ0  ~ DQ7   ↔ DQS0P/N ↔ DMI0
DQ8  ~ DQ15  ↔ DQS1P/N ↔ DMI1
```

这对原理图和 PCB 很重要。

---

## 16.3 Rank

Rank 可以理解为一组被同一个 CS 选择的内存阵列。

如果系统有：

```text
CS0
CS1
```

通常可能对应不同 Rank。

---

## 16.4 Die

Die 是芯片内部的裸片。

一个 LPDDR4 封装里面可能叠了多个 Die。容量越大，内部 Die 可能越多。

---

# 17. DRAM 内部为什么有行和列？

DRAM 内部不是简单的一维数组，而是二维矩阵。

可以想象成：

```text
             Column
          C0   C1   C2   C3
Row R0   [ ]  [ ]  [ ]  [ ]
    R1   [ ]  [ ]  [ ]  [ ]
    R2   [ ]  [ ]  [ ]  [ ]
```

每个交叉点就是一个存储单元。

---

## 17.1 DRAM 存储单元

一个 DRAM bit cell 通常由：

```text
一个电容
一个晶体管
```

组成。

电容有电荷表示 1，没有电荷表示 0。

但电容很小，电荷会泄漏，所以 DRAM 需要刷新。

这就是：

```text
Refresh 刷新
```

---

# 18. Bank 是什么？

Bank 可以理解成一个独立的小存储阵列。

例如：

```text
Bank0：Row × Column
Bank1：Row × Column
Bank2：Row × Column
Bank3：Row × Column
```

多个 Bank 的好处：

```text
一个 Bank 准备数据时
另一个 Bank 可以做其他操作
提高并行度
```

完整地址通常会被拆成：

```text
Channel / Rank / Bank / Row / Column / Byte Offset
```

---

# 19. Row Buffer 是什么？

Row Buffer 也叫行缓冲。

DRAM 读写不是直接访问某个单独格子，而是先打开一整行。

当打开某一行时，这一整行会被读到 Row Buffer / Sense Amplifier 中。

示意：

```text
DRAM Array:

        C0  C1  C2  C3  C4
R99    [ ][ ][ ][ ][ ]
R100   [A][B][C][D][E]   ← 打开这一行
R101   [ ][ ][ ][ ][ ]

            ↓

Row Buffer:
        A  B  C  D  E
```

重点：

> **ACT 打开的不是一个 bit，而是一整行。**

---

# 20. DDR 读取数据流程

一次典型读取流程：

```text
ACTIVATE → READ → 数据 Burst 输出 → PRECHARGE
```

可以记成：

```text
开行 → 选列 → 连续读 → 关行
```

---

## 20.1 ACT：打开行

控制器先发 ACT 命令，同时带上：

```text
Bank 地址
Row 地址
```

例如：

```text
ACT Bank1, Row100
```

意思是：

```text
打开 Bank1 里的 Row100
```

LPDDR4 内部会把 Row100 整行放到 Row Buffer。

---

## 20.2 tRCD：开行后等待

ACT 后不能立刻 READ。

因为整行数据需要被感放器读出并稳定。

这段等待时间叫：

```text
tRCD
```

简单理解：

> **tRCD = 打开行之后，到可以读/写列之前的等待时间。**

---

## 20.3 READ：读列

Row 已经打开后，控制器发 READ 命令，同时带上：

```text
Column 地址
```

例如：

```text
READ Column20
```

意思是：

```text
从当前打开的 Row Buffer 的 Column20 开始读
```

---

## 20.4 CL：读延迟

READ 发出后，数据不会立刻出现在 DQ 上。

从 READ 命令到有效数据出现之间的延迟叫：

```text
CL
CAS Latency
```

简单理解：

> **CL = 发出读命令后，等多久数据才出来。**

---

## 20.5 Burst：突发输出

DDR 一次 READ 不会只输出一个 bit，而是连续输出一串数据。

例如简化成：

```text
D0 → D1 → D2 → D3
```

对应列访问：

```text
Column20 → Column21 → Column22 → Column23
```

这叫：

```text
Burst
突发传输
```

---

## 20.6 PRE：关闭行

如果后面还访问同一个 Row，可以继续 READ，不需要关闭。

如果要访问另一个 Row，就要先 PRE：

```text
PRECHARGE
```

作用：

```text
关闭当前打开的 Row
让 Bank 准备打开下一行
```

---

# 21. Row Hit / Row Miss / Row Conflict

## 21.1 Row Hit：行命中

如果当前 Bank 已经打开 Row100，下一次还访问 Row100 的其他列：

```text
当前打开：Bank1 Row100
目标访问：Bank1 Row100 Column80
```

只需要：

```text
READ Column80
```

不用重新 ACT。

这叫 Row Hit。

特点：

```text
速度快
延迟低
效率高
```

---

## 21.2 Row Miss：行未命中

如果当前 Bank 没有打开任何 Row，而目标是 Row100：

```text
ACT Row100
READ Column
```

这叫 Row Miss。

---

## 21.3 Row Conflict：行冲突

如果当前打开 Row100，但目标是 Row200：

```text
当前打开：Bank1 Row100
目标访问：Bank1 Row200
```

需要：

```text
PRE Row100
ACT Row200
READ Column
```

这比 Row Hit 慢很多。

---

# 22. DDR 写入数据流程

写入流程和读取类似：

```text
ACT → WRITE → Burst 写入 → 可选 PRE
```

例如要写：

```text
Bank1 Row100 Column20
```

如果 Row100 没打开：

```text
1. ACT Bank1 Row100
2. 等待 tRCD
3. WRITE Column20
4. 控制器输出 DQ + DQS + DM/DMI
5. LPDDR4 根据 DQS 采样 DQ
6. 根据 DM/DMI 决定哪些字节真正写入
```

---

# 23. LPDDR4 写时序

写操作方向：

```text
控制器 → LPDDR4
```

写时序可以理解为：

```text
控制器发 WRITE 命令
控制器输出 DQ
控制器输出 DQS
控制器输出 DM/DMI
LPDDR4 根据 DQS 采样 DQ
```

简化波形：

```text
CK   : ┌─┐ ┌─┐ ┌─┐ ┌─┐
       └─┘ └─┘ └─┘ └─┘

CA   : [ WRITE ]

DQS  :       ↑   ↓   ↑   ↓

DQ   :       D0  D1  D2  D3

DM   :       M0  M1  M2  M3
```

写操作核心：

> **写：控制器发数据，也发采样节拍。**

---

# 24. LPDDR4 读时序

读操作方向：

```text
LPDDR4 → 控制器
```

读时序可以理解为：

```text
控制器发 READ 命令
等待 CL
LPDDR4 输出 DQ
LPDDR4 输出 DQS
控制器根据 DQS 采样 DQ
```

简化波形：

```text
CK   : ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
       └─┘ └─┘ └─┘ └─┘ └─┘

CA   : [ READ ]

等待 :       <---- CL ---->

DQS  :                    ↑   ↓   ↑   ↓

DQ   :                    D0  D1  D2  D3
```

读操作核心：

> **读：内存返回数据，也返回采样节拍。**

---

# 25. 为什么 DQS 采样点要在数据中间？

DQ 数据不是瞬间稳定的。

从 D0 切换到 D1 的过程中，会有不稳定区。

理想情况：

```text
DQ :  ---- D0 稳定窗口 ----
DQS:          ↑
            采样点
```

DQS 的采样边沿放在 DQ 稳定窗口中间，最安全。

如果太早：

```text
DQS 到了，但 DQ 还没稳定
```

如果太晚：

```text
DQ 快切换到下一个数据了
```

都会导致采错。

这个数据稳定窗口也常被叫做：

```text
Data Eye
眼图
```

DDR 设计的目标就是：

> **让采样点尽量落在眼图中心。**

---

# 26. 为什么 PCB 要做等长？

高速信号在 PCB 上传播需要时间。

线越长，到达越晚。

如果 DQ 和 DQS 长度差太大：

```text
DQS 到了，DQ 还没稳定
```

或者：

```text
DQ 已经快变成下一个数据，DQS 才到
```

就会采错。

所以 DDR 设计要求：

```text
DQ 与对应 DQS 等长
DQS P/N 差分对等长
CK P/N 差分对等长
Byte Lane 内 DQ 匹配
CA/命令地址组满足时序要求
```

本质不是为了“走线长度好看”，而是：

> **让信号到达时间对齐。**

---

# 27. 为什么 CK / DQS 要差分？

差分信号有两个互补信号：

```text
P：正端
N：负端
```

接收端看的是：

```text
P - N
```

优点：

```text
抗干扰能力强
共模噪声抵消
边沿更清晰
适合高速
```

所以 DDR 中：

```text
CKP / CKN 是差分
DQS_P / DQS_N 是差分
```

---

# 28. DDR Training 是什么？

DDR Training 是启动时控制器和 PHY 对 DDR 接口进行校准。

因为实际硬件中存在：

```text
PCB 走线长度差异
芯片工艺差异
封装差异
温度变化
电压变化
负载差异
```

所以控制器需要训练：

```text
读 DQS 延迟
写 DQS 延迟
DQ 采样位置
VREF
驱动强度
阻抗
```

目标：

```text
找到最可靠的采样窗口中心
```

如果 Training 失败，常见表现：

```text
不开机
卡在 BootROM / SPL / U-Boot
随机重启
内存测试失败
温度变化后异常
偶发死机
```

---

# 29. 常见时序参数

初学者不用一开始背 JEDEC 表，但要理解含义。

| 参数     | 含义                   | 简单理解              |
| ------- | ---------------------- | --------------------- |
| tRCD    | ACT 到 READ/WRITE 的等待时间 | 开行后等一会儿才能读写  |
| CL      | READ 到数据出来的延迟    | 读命令后等多久有数据  |
| tRP     | PRE 到下一次 ACT 的时间  | 关行后等多久能开新行  |
| tRAS    | Row 打开后至少保持时间   | 一行不能刚打开就关 |
| tWR     | 写完成到可以 PRE 的时间  | 写完后要等真正写入完成 |
| Burst Length | 一次连续传输长度    | 一次 READ/WRITE 连续传多少数据 |

---

# 30. CPU 读内存的完整过程

假设 CPU 执行：

```text
load 0x8000_1234
```

大概过程：

```text
1. CPU 发起读请求
2. Cache Miss，请求进入 SoC 内部总线
3. DDR Controller 收到请求
4. Controller 解析地址，得到 Channel / Rank / Bank / Row / Column
5. 判断目标 Row 是否已经打开
6. 如果没有打开，发 ACT
7. 等待 tRCD
8. 发 READ Column
9. 等待 CL
10. LPDDR4 返回 DQ + DQS
11. DDR PHY 根据 DQS 采样 DQ
12. 数据进入 Controller
13. 数据返回 CPU / 填入 Cache
```

从 CPU 看只是读一个地址。

从 DDR 看，其实是一串复杂时序。

---

# 31. CPU 写内存的完整过程

假设 CPU 执行：

```text
store 0x8000_1234 = 0x55
```

大概过程：

```text
1. CPU 发出写请求
2. 数据进入 Cache / 写缓冲
3. DDR Controller 解析地址
4. 得到 Channel / Rank / Bank / Row / Column
5. 如果目标 Row 没打开，发 ACT
6. 等待 tRCD
7. 发 WRITE Column
8. DDR PHY 输出 DQ + DQS + DM/DMI
9. LPDDR4 根据 DQS 采样 DQ
10. 根据 DM/DMI 决定哪些字节写入
11. 写完成后等待 tWR
12. 需要换行时 PRE
```

---

# 32. LPDDR4 原理图检查重点

做原理图时，可以按下面顺序检查。

## 32.1 电源

检查：

```text
VDD1 是否正确
VDD2 是否正确
VDDQ 是否正确
电源上电时序是否满足要求
去耦电容是否足够
去耦电容是否靠近芯片
```

---

## 32.2 VREF

检查：

```text
VREF 电压是否正确
来源是否可靠
是否有滤波
是否远离高速干扰
```

VREF 不稳会直接导致 0/1 判断错误。

---

## 32.3 ZQ/RZQ

检查：

```text
电阻阻值是否正确
精度是否满足要求
是否靠近芯片
是否接到干净 GND
```

---

## 32.4 CK

检查：

```text
CKP/CKN 是否接对
P/N 是否接反
差分对是否完整
是否接到正确 Channel
```

---

## 32.5 CA / 控制信号

检查：

```text
CA0~CA5 是否对应正确
CS 是否正确
CKE 是否正确
RESET_n 是否正确
ODT/相关控制脚是否按设计连接
```

---

## 32.6 DQ / DQS / DMI

重点检查 Byte Lane：

```text
DQ0~DQ7   是否对应 DQS0 和 DMI0
DQ8~DQ15  是否对应 DQS1 和 DMI1
```

还要检查：

```text
DQS P/N 是否接反
DMI 是否接到对应 Byte Lane
Channel A / B 是否混接
DQ 分组是否符合 SoC 设计要求
```

---

# 33. PCB 布线重点

## 33.1 DQ / DQS 是最关键组

重点：

```text
DQ 与对应 DQS 等长
DQS P/N 差分对等长
Byte Lane 内 DQ 组内等长
DMI 跟随对应 Byte Lane
```

---

## 33.2 CK 差分对

重点：

```text
CKP/CKN 差分阻抗控制
P/N 等长
参考平面连续
少打孔
远离噪声源
```

---

## 33.3 CA 组

重点：

```text
CA 组内长度匹配
与 CK 满足时序关系
避免跨分割平面
保持参考地连续
```

---

## 33.4 电源完整性

重点：

```text
去耦电容靠近电源球
大中小电容搭配
电源平面低阻抗
VREF 干净
VDDQ 噪声低
GND 回流路径完整
```

---

# 34. 初学者常见误区

## 误区 1：把 CK 当成采样 DQ 的主要时钟

正确理解：

```text
CK 管命令 / 地址
DQS 管数据采样
```

---

## 误区 2：认为 DQS 只由控制器发

正确理解：

```text
写：控制器发 DQS
读：LPDDR4 发 DQS
```

---

## 误区 3：DQ 和 DQS 没按 Byte Lane 匹配

例如：

```text
DQ0~DQ7 接到了 DQS1
DQ8~DQ15 接到了 DQS0
```

这种很容易训练失败。

---

## 误区 4：DMI 接错 Lane

DMI 必须和对应 DQ Byte Lane 匹配。

---

## 误区 5：只关注信号，不关注电源

很多 DDR 问题不是线接错，而是：

```text
电源噪声大
VREF 不稳
ZQ 电阻错误
GND 回流不好
去耦不足
```

---

## 误区 6：跨分割平面布线

高速信号跨参考平面分割，会破坏回流路径，导致信号质量变差。

---

# 35. 用仓库类比理解整套 DDR

| DDR 概念         | 类比          |
| -------------- | ----------- |
| DDR Controller | 仓库调度员       |
| DDR PHY        | 搬运接口 / 手脚   |
| LPDDR4         | 仓库          |
| CK             | 总调度节拍       |
| CA             | 指令单         |
| Bank           | 仓库分区        |
| Row            | 一排货架        |
| Column         | 货架上的位置      |
| Row Buffer     | 搬到前台的一整排货   |
| ACT            | 把一排货架搬到前台   |
| READ           | 从前台某个位置开始取货 |
| WRITE          | 往前台某个位置放货   |
| Burst          | 连续取 / 放多个货  |
| PRE            | 清空前台，准备换一排  |
| DQ             | 货物内容        |
| DQS            | 取 / 放货的节拍   |
| DM             | 这个货不要放进去的标记 |
| ZQ             | 调整搬运力度      |
| VREF           | 判断 0/1 的标准线 |

---

# 36. 最后速记版

```text
LPDDR4：
低功耗高速 DDR 内存。

DDR Controller：
在 SoC 内部，负责内存访问调度和地址映射。

DDR PHY：
在 SoC 内部，负责高速信号收发和训练。

LPDDR4 颗粒：
在 SoC 外部，真正存储数据。

CK：
主时钟，管命令/地址。

CA：
命令地址总线，告诉内存做什么、访问哪里。

DQ：
真正的数据线。

DQS：
数据采样节拍，跟 DQ 成组出现。

DM/DMI：
写掩码 / 数据反转标志。

写操作：
控制器发 WRITE
控制器发 DQ + DQS + DM
LPDDR4 根据 DQS 采样 DQ

读操作：
控制器发 READ
等待 CL
LPDDR4 返回 DQ + DQS
控制器根据 DQS 采样 DQ

行列访问：
ACT 打开 Row
READ/WRITE 选择 Column
Burst 连续传输
PRE 关闭 Row

PCB 核心：
DQ 和 DQS 对齐
Byte Lane 分组正确
CK/DQS 差分对正确
CA 时序满足
VREF 稳定
ZQ 正确
电源完整性好
```

---

# 37. 一句话总总结

> **LPDDR4 工作时，SoC 内部 DDR Controller 先把 CPU 的内存请求转换成 Bank / Row / Column 操作，再通过 DDR PHY 用 CK/CA 发命令和地址；真正传数据时，DQ 和 DQS 成组一起走，接收端根据 DQS 在 DQ 稳定窗口中间采样；PCB 设计的核心就是保证这些高速信号在正确时间到达。**
