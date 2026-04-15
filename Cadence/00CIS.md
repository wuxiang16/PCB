# 数据库
教程中是：Trent_CIS.mdb数据库

- Manufacturer Part Number = 厂商定义的唯一物料编号
- Value 在 CIS 中的角色（核心定位），可以这样理解三层结构：

```
CIS Part Number（公司料号） → 唯一KEY
        ↓
MPN（厂商料号） → 采购
        ↓
Value → 人看 / 设计展示
```

👉 一句话：

- Value 是给工程师看的，Manufacturer Part Number 是给采购用的。


# Datasheet / Capture Library / Allegro Library

- Datasheet是说明书
- Capture是电路图中的“图标symbol”
- Allegro是PCB上的“实物形状”

## 三者是怎么连起来的？

- 中间靠一个字段 —— **footprint**
- 最关键的一步：在Capture中有个属性：```PCB Footprint = QFN32_5x5```，意思是：我这个图标，到PCB中要使用QFN_5x5封装。
  
---

# 完整流程（你实际会怎么用）
✅ Step 1：看 Datasheet

- 查引脚 → 画图用
- 查封装尺寸 → 画 PCB 用

✅ Step 2：在 Capture 画电路

- 用 Symbol（图标）连接电路：
- 比如：
电阻连哪里，MCU 接哪里

✅ Step 3：生成网表 → 给 Allegro
- 相当于：“把电路图交给 PCB 工程师”

✅ Step 4：Allegro 放封装
- 根据：PCB Footprint，自动找到封装并放到 PCB 上

## 数据库中的字段：
- Schematic Part
- Allegro PCB Footprint

![alt text](image.png)

---

# 数据库的内容可以将Excel文件内容导入

# 1. Cadence相关实操
## 1.1 添加数据源
![[Pasted image 20260415210838.png]]

![[Pasted image 20260415211029.png]]
- 解释下 ODBC，ODBC 全称 Open Database Connectivity 开放数据库连接。
- ODBC是把 *.mdb 文件和 Cadence CIS 串联起来的关键桥梁。

自定义数据源的名字为 CadenceCIS，找到课程提供的数据库。![[Pasted image 20260415211052.png]]

## 1.2 Capture CIS 原理图 CIS 数据库的配置
先任意打开一个工程文件。

![[Pasted image 20260415211241.png]]
Option 中选择 CIS Configuration。

![[Pasted image 20260415211330.png]]

![[Pasted image 20260415211349.png]]

![[Pasted image 20260415211410.png]]

![[Pasted image 20260415211455.png]]

![[Pasted image 20260415211515.png]]

![[Pasted image 20260415211552.png]]

![[Pasted image 20260415211632.png]]

![[Pasted image 20260415211818.png]]

![[Pasted image 20260415211842.png]]

![[Pasted image 20260415211956.png]]

![[Pasted image 20260415212025.png]]

Finish之后会弹出如下页面就配置完了，直接确认。
![[Pasted image 20260415212059.png]]

确认后弹出页面，找个位置命名保存，这里命名为 wuxiang_cis.dbc。
![[Pasted image 20260415212257.png]]

任意一个PAGE中，如下，按键```z```打开 CIS Explorer。



## 1.3 配置 Capture.ini
WUXIANG_CIS.DBC是我们前面配置的关联 `*.mdb`数据库的配置文件。

![[Pasted image 20260415214738.png]]

# 2. 库中加器件
以电容为例，随便在 *.mdb 数据库中新增一个。保证SCH和PCB Footprint要有。

![[Pasted image 20260415222613.png]]
![[Pasted image 20260415222702.png]]





