---
{"dg-publish":true,"permalink":"/Code/3.ComputerNetwork/CN.0.计算机网络/","title":"计算机网络","noteIcon":""}
---


# 计算机网络

《计算机网络：自顶向下方法》以及相关课程第一章的笔记

## 互联网组成

从*硬件构成*来看，互联网是以**主机(端系统)** 和 **分组交换设备**为节点，**通信链路**为边，层层连接形成的*网络的网络*
- 分组交换设备：路由器(网络层)/交换机(数据链路层)

从*服务*来看，互联网是由**分布式应用**和**为其提供通信服务的基础设施**组成的

## 互联网连接方式

方式一：

$$ \frac{网络核心 - 接入网 - 网络边缘}{物理媒体} $$

依照节点的链路和类型可分为**网络边缘**，**网络核心**和**接入网、物理媒体**

**网络边缘**：主机以及主机之上的应用程序

**网络核心**：分组交换机的网状网络，传输数据实现主机间通信
{ #internetCore}


**接入网**的模式包括**数字用户线**(Digital Subscriber Line, DSL)(拨号上网)、**电缆因特网接入**(cable Internet access)、**光纤到户**(Fiber To The Home, FTTH)、**WiFi**(无线局域网)和**蜂窝网络**(广域无线)等

**物理媒体**为发射器-接收器对间传播比特的介质，根据媒体是否为固体分为**导引型媒体**和**非导引型媒体**

数据传输基本方法：**电路交换**(circuit switching) 和**分组交换**(packet switching)
- 前者为为分片传输，为呼叫预留端-端资源，资源不共享，保证性能但会造成浪费，连接建立时间长，不适用于互联网连接，多用于传统电话网络
- 后者使用**存储转发传输**(store-and-forward transmission) 机制，传输时使用全部带宽，每个交换节点完全存储上一个节点传输的全部分组数据后再进行转发，一次转发为**一跳**，会导致**排队**和**延迟**
	存储转发又根据*有无网络层的连接*分为**数据报网络**和**虚电路网络**
	- 前者由分组的目标地址决定下一跳，路径不固定
	- 后者依靠分组维护的虚电路表决定下一跳，路径在呼叫建立时即决定

方式二：

根据节点的关系划分**ISP网络**(Internet Service Providers)，互联网由接入ISP、区域ISP、第一层ISP、**ICP**(Internet Content Providers）、**PoP**(Point of Presence)和**IXP**(Internet Exchange Point)组成。

![](https://image.jiang849725768.asia/2022/202212021759977.png)

端系统通过接入ISP与互联网相连，接入ISP上是区域ISP，再之上是全局/全球ISP，底层ISP可以同时连接多个更高层级的ISP(多宿)，连接使用PoP，位于同一层级(对等)的ISP可以相互连接，IXP提供多个ISP的一起对等
ICP为Google, Microsoft等自构建的服务网络，目的是更好更便宜地与用户连接，其同时与各级ISP连接

## 互联网限制

受节点和物理媒体的能力以及物理规律的限制，在分组数据传输的过程中会发生**时延**和分组丢失(**丢包**)，端系统每秒传输的数据量(**吞吐量**)也受到限制

### 时延

时延主要包括**节点处理时延**，**排队时延**，**传输时延**，**传播时延**
- 节点处理时延为节点对数据进行检查以及路线规划等花费的时间
- 排队时延为流量过大时等待之前其他数据传播的时间
- 传输时延为分组数据以链路传输速率完整转发的时间
- 传播时延为数据在两个节点之间物理传播的时间

排队延时收到**流量强度**的影响，流量强度 $I = \frac{La}{R}$，L 表示分组数据大小，a 表示每秒到达分组数，R 表示链路带宽(比特传输速度)

设计系统时流量强度不能大于1，而随着流量强度趋近于1，平均排队时延趋于无限大
由于分组到达节点间的间隔具有随机性，因此当流量强度等于1时也会因为链路传输能力浪费的累积而导致延迟持续增长至无限大

### 丢包

**丢包**源于实际情况下链路的队列缓冲区容量有限，不可能无限排队，因此当分组到达一个满队列节点时，该分组将会被丢弃，根据不同协议该分组可能会被前一个节点或源端系统重传，或根本不重传

### 吞吐量

**吞吐量**为源主机和目标主机之间传输的速率(数据量/单位时间)
- **瞬间吞吐量**为某一时间点的即时速率
- **平均吞吐量**为在一个时间段内数据传输速度的平均值
- **瓶颈链路**为端到端路径上限制吞吐量的速率最低的链路，通常瓶颈为源主机和目标主机的接入链路

## 互联网分层

在网络中，协议是一套用于格式化和处理数据的规则，网络协议就像计算机的一种共同语言，一个网络中的计算机可能会使用截然不同的软件和硬件，然而，协议的使用使它们能够相互通信

互联网将网络复杂的功能进行**分层**，每一层实现了其中一个或一组功能，在分层的基础上，互联网对协议以及实现协议的网络硬件和软件进行组织

每层通过**服务访问原语**(primitive)利用直接下层提供的服务实现本层协议，通过本层协议实体相互交互执行本层的协议动作实现本层的功能，功能中包含通过 SAP 向上层提供的服务
- **服务**(Service)：低层实体向上层实体提供它们之间的通信的能力，是通过原语(primitive)来操作的，垂直
- **服务访问点 SAP** (Services Access Point) ：上层使用下层提供服务通过的层间接口，可用于标志不同上层实体

服务的类型：
- 面向连接的服务：两个通信实体建立连接后进行通信
- 无连接的服务：两个对等层实体不建立连接而直接通信

层间数据传输：
![](https://image.jiang849725768.asia/2022/202212032158809.png)
- **数据单元**(Data Unit)
- **层间控制信息**(Interface Control Information)

分层的优劣：
- 优点：
	- 概念化：结构清晰
	- 结构化：易于维护升级
- 缺点：
	- 效率降低
	- 功能冗余

数据在每一层加入该层首部信息**封装**(encapsulation)后传递给下一层，接收端收到后解封装取得信息，每一层传递的数据具有两种类型的字段：**首部字段**和**有效载荷字段**(payload field)，后者通常是来自上一层的数据

### 协议栈

各层的所有协议被称为**协议栈**(protocol stack) ，互联网协议栈通常可以分为五个层次：
- **5** [[Code/3.ComputerNetwork/CN.0a.应用层\|应用层]]
	- 提供网络应用服务
	- FTP，SMTP，HTTP，DNS
	- **报文**(message)
- **4** [[Code/3.ComputerNetwork/CN.0b.传输层\|传输层]]
	- 进程到进程，TCP可靠，UDP快速
	- TCP，UDP
	- **报文段**(segment)：TCP段，UDP数据报
- **3** [[Code/3.ComputerNetwork/CN.0c.网络层\|网络层]]
	- 主机到主机，分组传输，尽力而为，不可靠
	- IP，路由协议
	- **分组**(packet)(如果无连接方式：数据报datagram)
- **2** [[Code/3.ComputerNetwork/CN.0d.链路层\|数据链路层]]
	- 相邻节点点到点通信，不一定可靠
	- **帧**(frame)
- **1** 物理层
	- 数字信号->物理信号->发送->物理信号->数字信号
	- **位**(bit)
- **0** 物理媒体 (physical medium)

### TCP/IP 模型

**TCP/IP 网络模型**为四层模型，包括应用层，传输层，网络层和网络接口层
> ![|600](https://cdn.xiaolincoding.com/gh/xiaolincoder/ImageHost3@main/%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F/%E6%B5%AE%E7%82%B9/%E5%B0%81%E8%A3%85.png)

### OSI 模型

ISO 提出**开放系统互连(OSI) 模型**，包括应用层，表示层，会话层，传输层，网络层，数据链路层，物理层

1. 物理层：负责将数据转换为比特流
2. 数据链路层：负责保证比特流的顺序和出错检测，促进同一网络上两台设备之间的数据传输
    - 常见协议有 MAC、PPP、HDLC、以太网等
3. 网络层：负责数据分组及路由选择，促进两个不同网络之间的数据传输
    - 常见协议有 IP、ARP、ICMP、OSPF、RIP 等
4. 传输层：负责两个设备间的端到端通信，包括数据传输和流程控制
    - 常见协议有 TCP、UDP、SCTP 等
5. 会话层：负责建立、管理和终止两个进程间的通信，还负责同步数据传输与检查点
    - 常见协议有 RPC、NetBIOS 等
6. 表示层：用于确保数据可供应用程序使用，负责完成数据转换、加密和压缩
    - 常见协议有 JPEG、MPEG 等
7. 应用层：负责协议和数据操作，软件应用程序依靠上述操作向用户呈现有效数据
    - 常见协议有 HTTP、FTP、SMTP、DNS 等
    - 客户端软件应用程序不属于应用程序层

两个模型的差异
- OSI 引入了服务、接口、协议、分层的概念，TCP/IP 借鉴了 OSI 的这些概念建立 TCP/IP 模型
- OSI 先有模型，后有协议，先有标准，后进行实践；而 TCP/IP 则相反，先有协议和应用再提出了模型，且是参照的 OSI 模型
- OSI 是一种理论下的模型，而 TCP/IP 已被广泛使用，成为网络互联事实上的标准

![|1050](https://img-blog.csdn.net/2018041112053246?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5NTIxNTU0/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)