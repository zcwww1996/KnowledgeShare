[TOC]

来源：https://www.jianshu.com/p/72e395d8cb33

ODS、DW和DM的概念：
- ODS——操作性数据(短实时)
- DW——数据仓库
- DM——数据集市

# 1.数据中心整体架构

![数据中心整体架构](https://imageproxy.pimg.tw/resize?url=https://i.loli.net/2020/11/30/rMo8TUlivQgPbLp.png)


数据仓库的整理架构，各个系统的元数据通过ETL同步到操作性数据仓库ODS中，对ODS数据进行面向主题域建模形成DW（数据仓库），DM是针对某一个业务领域建立模型，具体用户（决策层）查看DM生成的报表。

# 2.数据仓库的ODS、DW和DM概念

数据仓库（Data Warehouse）：是一个面向主题的（Subject Oriented）、集成的（Integrated）、<font style="color: red;">**相对稳定的**</font>（Non-Volatile）、<font style="color: red;">**反映历史变化**</font>（Time Variant）的数据集合，用于<font style="color: red;">**支持管理决策**</font>(Decision Making Support)。

ODS：是一个面向主题的、集成的、<font style="color: red;">**可变的、当前的细节数据**</font>集合，用于支持企业对于即时性的、操作性的、集成的全体信息的需求。

![ods、dw、dm区分](https://imageproxy.pimg.tw/resize?url=https://i.loli.net/2020/11/30/Wcji4yrOkQgSHCo.png)

ODS中的数据具有以下4个基本特征：
- ① 面向主题的：进入ODS的数据是来源于各个操作型数据库以及其他外部数据源，数据进入ODS前必须经过 ETL过程（抽取、清洗、转换、加载等）。
- ② 集成的：ODS的数据来源于各个操作型数据库，同时也会在数据清理加工后进行一定程度的综合。
- ③ 可更新的：可以联机修改。这一点区别于数据仓库。
- ④ 当前或接近当前的：“当前”是指数据在存取时刻是最新的，“接近当前”是指存取的数据是最近一段时间得到的。

# 3.ODS与DW的区别

ODS在DB ~ ODS ~ DW三层体系结构中起到一个承上启下的作用。
ODS中的数据虽然具有DW中的数据的面向主题的、集成的特点，但是也有很多区别。

（1）**存放的数据内容不同**：<br>
ODS中主要存放当前或接近当前的数据、细节数据，可以进行联机更新。<br>
DW中主要存放细节数据和历史数据，以及各种程度的综合数据，不能进行联机更新。<br>
ODS中也可以存放综合数据，但只在需要的时候生成。

（2）**数据规模不同**：<br>
由于存放的数据内容不同，因此DW的数据规模远远超过ODS。

（3）**技术支持不同**：<br>
ODS需要支持面向记录的联机更新，并随时保证其数据与数据源中的数据一致。<br>
DW则需要支持ETL技术和数据快速存取技术等。

（4）**面向的需求不同**：<br>
ODS主要面向两个需求：一是用于满足企业进行全局应用的需要，即企业级的OLTP和即时的OLAP；二是向数据仓库提供一致的数据环境用于数据抽取。<br>
DW主要用于高层战略决策，供挖掘分析使用。

（5）**使用者不同**：<br>
ODS主要使用者是企业中层管理人员，他们使用ODS进行企业日常管理和控制。<br>
DW主要使用者是企业高层和数据分析人员。

# 4.ODS、DW、DM协作层次图

![协作层次](https://imageproxy.pimg.tw/resize?url=https://i.loli.net/2020/11/30/aq1mcrxfl52H4es.png)


# 5.通过一个简单例子看这几层的协作关系

![例子](https://imageproxy.pimg.tw/resize?url=https://i.loli.net/2020/11/30/rJS7sNVLTpKIqoP.png)


# 6.ODS到DW的集成示例

![集成例子](https://imageproxy.pimg.tw/resize?url=https://i.loli.net/2020/11/30/MK7LsoU4H8IhR5i.png)

