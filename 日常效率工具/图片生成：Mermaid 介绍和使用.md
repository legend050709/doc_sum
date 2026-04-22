```table-of-contents
```
# 介绍
Mermaid `[ 'mə:meid ]`（字面翻译：美人鱼；女子游泳健将） 是一个基于 JavaScript 的图表生成工具，它使用简单的文本语法来创建丰富的图表和可视化内容。主要特点包括：
- **纯文本描述**：使用类似 Markdown 的简洁语法
- **多种图表支持**：流程图、序列图、甘特图、类图等
- **无缝集成**：支持 GitHub、GitLab、VS Code 等平台
- **开源免费**：MIT 许可证，可自由使用

#  核心图表类型

## 基本结构
### 关键字与方向
关键字和方向，是最基本的。你得告诉 **Mermaid**，你要画的是个什么图，图的走向是什么？
- 使用 `关键字`，声明图的**类型**
- 使用 `方向`，声明图的**方向**(走向)

#### 关键字
所有 Mermaid 图表都以图表类型声明开始：
```bash
graph TD  // 流程图
sequenceDiagram  // 序列图
gantt  // 甘特图
```

> 注意 ：关键字的所有字母，必须**全部小写**！任何一个字母大写都会导致报错！

#### 方向
**方向有四种 (五种写法)：**
```text
TB
    top to bottom 自上而下
TD
    top-down 自上而下
BT
    bottom to top 自下而上
LR
    left to right 从左往右
RL
    right to left 从右往左
```

> 注意 方向的字母必须全部**大写**！否则会报错！

#### 语法
- **关键字**与**方向** 写在**首行**；先写**关键字**，用一个**空格**隔开，再写**方向**
- 书写后续代码时，**推荐**用一个 制表符 `tab` 缩进后书写；只是推荐，不缩进也不会报错

#### 语句的结束
**Mermaid** 语句的结束，有**两种**方式:
1. 在结尾加 分号 `;`
2. 换行
3. 也可以两个都用，**换行 + 分号**

范例1：
```mermaid
graph LR
	a-->b
	c-->d
```

范例2：
```mermaid
graph LR
	a-->b; c-->d
```



## 流程图 (Flowchart)
```mermaid
graph TD
    A[开始] --> B(处理步骤)
    B --> C{条件判断}
    C -->|是| D[执行操作]
    C -->|否| E[结束]
    D --> E
```
### 流程图语法
```text
graph [方向]
    A[方形节点] --> B(圆角节点)
    B --> C{菱形条件}
    C -->|条件1| D[结果1]
    C -->|条件2| E[结果2]
```

说明：
```bash
方向标识：
    TB/TD: 从上到下
    BT: 从下到上
    RL: 从右到左
    LR: 从左到右
```


## 序列图 (Sequence Diagram)
```mermaid
sequenceDiagram
    participant 用户
    participant 系统
    用户->>系统: 请求数据
    系统-->>用户: 返回结果
    loop 重试机制
        用户->>系统: 重试请求
        系统-->>用户: 返回结果
    end
```

## 甘特图 (Gantt Chart)
```mermaid
gantt
    title 项目计划
    dateFormat  YYYY-MM-DD
    section 开发阶段
    需求分析       :a1, 2023-01-01, 30d
    系统设计       :after a1, 20d
    编码实现       :2023-02-20, 25d
    section 测试阶段
    单元测试       :2023-03-15, 15d
    集成测试       :2023-04-01, 20d
```

## 状态图 (State Diagram)
```mermaid
stateDiagram-v2
    [*] --> 待机
    待机 --> 运行: 启动命令
    运行 --> 待机: 停止命令
    运行 --> 故障: 错误发生
    故障 --> 待机: 复位操作
```

## 饼图 (Pie Chart)
```mermaid
pie
    title 编程语言使用比例
    "Python" : 40
    "Java" : 25
    "C++" : 20
    "其他" : 15
```

## 类图 (Class Diagram)

```mermaid
classDiagram
    Animal <|-- Duck
    Animal <|-- Fish
    Animal <|-- Zebra
    Animal : +int age
    Animal : +String gender
    Animal: +isMammal()
    Animal: +mate()
    class Duck{
      +String beakColor
      +swim()
      +quack()
    }
    class Fish{
      -int sizeInFeet
      -canEat()
    }
    class Zebra{
      +bool is_wild
      +run()
    }
```


## 数据包图（packet）
如下所示，TCP头的结构：

```mermaid
---
title: "TCP Packet"
---
packet-beta
  0-15: "Source Port"
  16-31: "Destination Port"
  32-63: "Sequence Number"
  64-95: "Acknowledgment Number"
  96-99: "Data Offset"
  100-105: "Reserved"
  106: "URG"
  107: "ACK"
  108: "PSH"
  109: "RST"
  110: "SYN"
  111: "FIN"
  112-127: "Window"
  128-143: "Checksum"
  144-159: "Urgent Pointer"
  160-191: "(Options and Padding)"
  192-255: "Data (variable length)"

```

## 柱状图和折线图(xy-chart)
```mermaid

    xychart-beta
    title "Sales Revenue"
    x-axis [jan, feb, mar, apr, may, jun, jul, aug, sep, oct, nov, dec]
    y-axis "Revenue (in $)" 4000 --> 11000
    bar [5000, 6000, 7500, 8200, 9500, 10500, 11000, 10200, 9200, 8500, 7000, 6000]
    line [5000, 6000, 7500, 8200, 9500, 10500, 11000, 10200, 9200, 8500, 7000, 6000]
```

```mermaid
    xychart-beta
    title "Sales Revenue"
    x-axis [jan, feb, mar, apr, may, jun, jul, aug, sep, oct, nov, dec]
    y-axis "Revenue (in $)" 4000 --> 11000
    bar [5000, 6000, 7500, 8200, 9500, 10500, 11000, 10200, 9200, 8500, 7000, 6000]
```

```mermaid

    xychart-beta
    title "Sales Revenue"
    x-axis [jan, feb, mar, apr, may, jun, jul, aug, sep, oct, nov, dec]
    y-axis "Revenue (in $)" 4000 --> 11000
    line [5000, 6000, 7500, 8200, 9500, 10500, 11000, 10200, 9200, 8500, 7000, 6000]
```

## 象限图(quadrant-chart)
象限图是一种将坐标平面分成四个象限（区域）的图表，常用于：
- 把数据点根据两个变量的不同取值范围，划分到四个象限里；
- 便于分析数据在不同象限的位置和分布；
- 例如，用于绩效分析、风险评估、市场细分等。
简单来说，**象限图就是在XY坐标系基础上，通过两条中轴线将图分成四个部分，展示数据的分类情况**。

## 状态图(state)

```mermaid
stateDiagram-v2
    [*] --> Still
    Still --> [*]
    Still --> Moving
    Moving --> Still
    Moving --> Crash
    Crash --> [*]
```


## 时间线(timeline)
```mermaid
timeline
    title Timeline of Industrial Revolution
    section 17th-20th century
        Industry 1.0 : Machinery, Water power, Steam <br>power
        Industry 2.0 : Electricity, Internal combustion engine, Mass production
        Industry 3.0 : Electronics, Computers, Automation
    section 21st century
        Industry 4.0 : Internet, Robotics, Internet of Things
        Industry 5.0 : Artificial intelligence, Big data,3D printing
```

## ER图(Entity Relationship图)

```mermaid
erDiagram
  CUSTOMER ||--o{ ORDER : places
  ORDER ||--|{ LINE_ITEM : contains

```

## 看板图(kanban)
看板图（Kanban 图）是一种可视化工作流程的图表，起源于日本丰田的生产管理体系，现在广泛用于**敏捷开发、项目管理、任务流程控制**等领域。

**Kanban 图** 是一种将工作项按照状态分栏展示的图表，常见的栏有：
- 📝 To Do（待办）
- 🚧 In Progress（进行中）
- ✅ Done（已完成）

### Kanban 的关键理念
**可视化工作**：让每个人都能看到项目当前状态。
 **限制在制品数量（WIP）**：避免多任务导致效率低。
**持续流动**：任务从左到右流动，清晰反映进度。
**持续改进**：通过观察瓶颈优化流程。

```mermaid
---
config:
  kanban:
    ticketBaseUrl: 'https://org.atlassian.net/browse/#TICKET#'
---
kanban
  Todo
    [Create Documentation]
    docs[Create Blog about the new diagram]
  [In progress]
    id6[Create renderer so that it works in all cases. We also add som extra text here for testing purposes. And some more just for the extra flare.]
  id9[Ready for deploy]
    id8[Design grammar]@{ assigned: 'knsv' }
  id10[Ready for test]
    id4[Create parsing tests]@{ ticket: MC-2038, assigned: 'K.Sveidqvist', priority: 'High' }
    id66[last item]@{ priority: 'Very Low', assigned: 'knsv' }
  id11[Done]
    id5[define getData]
    id2[Title of diagram is more than 100 chars when user duplicates diagram with 100 char]@{ ticket: MC-2036, priority: 'Very High'}
    id3[Update DB function]@{ ticket: MC-2037, assigned: knsv, priority: 'High' }

  id12[Can't reproduce]
    id3[Weird flickering in Firefox]
```

![](attachments/Pasted%20image%2020250731205647.png)

## 需求图（requirement）

```mermaid
requirementDiagram
requirement test_req {
id: 1
text: the test text.
risk: high
verifyMethod: test
}
element test_entity {
type: simulation
}
test_entity - satisfies -> test_req
```

# 如何使用 Mermaid
## mermaid在线编辑器

[mermaid在线编辑](https://mermaid.live/)

### 支持的图表
![](attachments/Pasted%20image%2020250731205126.png)

![](attachments/Pasted%20image%2020250731203410.png)


## VS Code 集成
5. 安装 Mermaid 插件
6. 创建 `.mmd` 文件
7. 编写 Mermaid 代码
8. 右键选择 "Preview Mermaid"

## 在 Markdown 中使用
```mermaid
graph LR
    A --> B
    B --> C
```



# 参考
```bash

# Mermaid 系列
https://pkmer.cn/tags/mermaid/

# Mermaid 流程图
https://pkmer.cn/Pkmer-Docs/02-%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86%E5%9F%BA%E7%A1%80/mermaid/mermaid%E8%AF%AD%E6%B3%95-%E6%B5%81%E7%A8%8B%E5%9B%BE/

# Mermaid 语法
https://pkmer.cn/Pkmer-Docs/02-%E7%9F%A5%E8%AF%86%E7%AE%A1%E7%90%86%E5%9F%BA%E7%A1%80/mermaid/mermaid%E8%AF%AD%E6%B3%95/


```