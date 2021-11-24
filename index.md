
<br>
<span id="head"> </span> 

 :fire:**[点击查看精选 PCIe 系列文章](#pcie_protocol)**:fire:

---
> :loudspeaker:&emsp;**声明**：
> -  🥭 **作者主页：【[MangoPapa的CSDN主页](https://blog.csdn.net/weixin_40357487)】。**
> - ⚠️ **本文首发于CSDN，转载或引用请注明出处【[点击查看原文](https://blog.csdn.net/weixin_40357487/article/details/121265623)】。**
> - ⚠️ **本文为非盈利性质，目的为 <font color=red>个人学习记录</font> 及 <font color=gree>知识分享</font>。因个人能力受限，存在协议解读不正确的可能。若您参考本文进行产品设计或进行其他事项并造成了不良后果，本人不承担相关法律责任。**
> - ⚠️ **若本文所采用图片或相关引用侵犯了您的合法权益，请联系我进行删除。**
> - 😄 **欢迎大家指出文章错误，欢迎同行与我交流 ~**
> - 📧 **邮箱：<font color=blue>mangopapa@yeah.net**
> 

---
[toc]

---


> # 0. 前言
> &emsp;&emsp;前段时间看PCIe Spec PRS部分看得云里雾里，问了几个同行都说没用过，网上找PRS、PRI相关资料也仅得只言片语，所以写 [【PCIe ATS介绍】](https://blog.csdn.net/weixin_40357487/article/details/120245027)的时候有意绕过了PRS这部分。ATS博文发出后，不断有同行问我PRS咋没写，想学一下。我心想，想学PRS的大有人在，不论对错我还是写写吧，总得有个人先分享。我写出来给大家看看，沟通沟通交流交流，互通有无查缺补漏，可能就懂了。

<br>

---

<table><td bgcolor=blue><center><font size=5 color=white> — 1. 背景介绍 —</center></td></table>

---

# 1. 背景介绍
## 1.1 从ATS到ATS+PRS
&emsp;&emsp;Page Request Services(PRS)，页请求服务，是Address Translation Services (ATS)地址转换服务的扩展项。若支持ATS的EP发送一笔地址转换请求，但RC地址转换代理（Translation Agent，TA）的地址转换保护表（Address Translation & Protection Table，ATPT）中没找到该虚拟地址对应的物理地址，这时候设备仍然想访问这个地址的数据，该怎么办？有两种解决办法：① 放弃ATS服务，EP端继续发送未转换地址的读写访问，让RC端的SMMU/IOMMU对未转换地址进行转换处理，这种方式显然不如ATS性能好；② RC端把这一段地址映射添加到RC TA ATPT中，继续采用ATS进行访问。PRS便是属于第二种方式。

&emsp;&emsp;ATS不一定支持PRS，但PRS一定需要ATS，为什么呢？因为PRS是ATS的扩展啊，两者紧密配合，携手并进。

&emsp;&emsp;简单讲， <mark> <font color=red>**PRS就是在数据Unpin情况下实现EP DMA的一种机制**。</font> </mark>

<br>

## 1.2 PRS的优势
&emsp;&emsp;如果Device仅采用ATS来搬运大量的数据，从单个设备来看，我肯定希望把所有数据都Pin在主存里，不用置换页，多好。但是，主存空间就那么大，你占用得多，那其他进程分得的就少，从系统层面看势必会影响整体性能。从全局利益出发，你ATS少Pin一点，剩下的用PRS来干。

&emsp;&emsp;PRS采用动态存储机制，在需要访问内存某页的时候就把其Pin在主存，使用完就释放，保证了整体系统性能。

<br>

---

<table><td bgcolor=blue><center><font size=5 color=white> —  2. PRS机制 —</center></td></table>

---

# 2. PRS机制
&emsp;&emsp;为了在EP和RC之间沟通地址转换事宜，PRS机制用到了两种消息：**页请求消息** 和 **页请求组响应消息**。

&emsp;&emsp;<font color=darkorange>**ATS**</font>+<font color=gree>**PRS**</font>协同工作流程大致如下：


1. <font color=darkorange>EP发送地址转换请求给RC;
2. <font color=darkorange>RC未找到相关地址映射，反馈转换失败消息给EP；
3. <font color=gree>EP发送页请求消息给RC;
4. <font color=gree>如果虚拟地址对应主存空间，那就直接pin住；如果虚拟地址对应外存空间，从外存搬到主存后pin住，并生成虚拟地址到物理地址的映射；
5. <font color=gree>RC反馈页请求组响应消息给EP；
6. <font color=darkorange>EP再次发送地址转换请求给RC；
7. <font color=darkorange>RC回复转换完成消息CplD给EP；
8. <font color=darkorange>EP采用转换后的地址完成访问。

<br>

## 2.1 页请求消息（Page Request Message）
&emsp;&emsp;页请求是指Function需要访问主存中的页，或言Function要访问主存中的某段地址范围。页请求消息是EP发给RC的消息，EP中的Function给其相关RC发送页请求消息来请求页访问。

<br>

### 2.1.1 页请求消息格式
&emsp;&emsp;页请求服务允许把多笔页请求归类为一组，组成页请求组（Page Request Groups, PRGs）。一个PRG包括一笔或多笔页请求，每一笔页请求单独发送一次页请求消息，同一PRG中的多个页请求消息采用统一的PRG Index（9bit），用以在当前Function中唯一标识该PRG。

&emsp;&emsp;页请求消息格式如下图所示（图1），整体由4DW组成，前2DW为标准的PCIe消息头标，后2DW为请求页的未转换地址、访问许可、PRG Index等信息。

![图1 Page Request Message](https://img-blog.csdnimg.cn/cd2d7834b40043eea51611a7be32af2c.png#pic_center)

<center><font size=2>图1 Page Request Message<br><br>

&emsp;&emsp;各主要字段意义介绍如下：

- **Fmt**： 001b，表示当前TLP为4DW头标、无数据的TLP。
- **Type**： 1000*b，表示当前TLP为路由到RC的Msg。
- **Length**： 0，标识Data长度为0。
- **Message Code**：4，页请求消息的消息码。
- **Page Address**：请求页的未转换地址，地址低12bit为0，4KB对齐，当然也可以采用更大的页，比如8KB，届时RC需要忽略page_address[13]，以此类推。
- **Page Request Group Index**：PRG Index。正常情况下，RC响应某个页请求时，应采用与页请求消息相同的PRG Index。
- **L，Last Request in PRG**：由于RC不知道一组PRG有多少页请求消息，所以一组PRG中最后发送的页请求消息应进行特殊标记，告知RC这是当前PRG消息的最后一笔。该bit置1表示当前页请求消息是其所在PRG中的最后一笔页请求消息。该bit置0表示当前页请求组的页请求消息还没发完，后续还有采用当前PRG Index的页请求消息。对于PRG内只有一笔页请求的情况，该位置1。对于PRG内有多笔页请求的情况，除最后一笔页请求之外，其他所有页请求消息的L字段都应置0。
- **W，Write Access Request**：写访问请求，该位置1表明Function接下来要写该页，RC收到W=1的页请求消息后可以把相关页标记为<u>脏页*</u>。ATS中跟这个字段相关联的由地址转换请求中的NW字段以及转换完成消息中的R/W字段<font color=grey size=3>（注：脏页是指高速缓存中数据被进程修改过的页，脏页需要在合适的时间刷入磁盘。）
- **R，Read Access Request**：读访问请求，该位置1表明Function接下来要读该页。对于代有PASID TLP Prefix且Executed Requested可执行请求位为0的页请求，R字段必须置1（要先读出来才能执行）。

<br>

### 2.1.2 页请求消息排序
&emsp;&emsp;如上所述，RC不知道一组PRG有多少页请求消息，对于PRG内有多笔页请求的情况，除最后一笔页请求之外，其他所有页请求消息的L字段都应置0。为了提升多Function、多PRG的系统性能，使用过程中往往会开启宽松排序或ID排序，这就导致一个问题，即最后一笔有可能会被重新排序从而超过其前面发送的同组页请求消息，导致RC误判为这组PRG消息已经结束，从而导致不可预知的风险。为了解决该问题，最后一笔页请求消息必须关闭宽松排序，确保同组PRG消息中该笔请求消息最后发送也是最后达到。PCIe事务排序规则请参考 [【PCIe事务排序】](https://blog.csdn.net/weixin_40357487/article/details/120162461)。

<br>

### 2.1.3 页请求接口信用量
&emsp;&emsp;页请求接口（Page Request Interface，PRI）控制PCIe Controller收发页请求/响应消息。每个Function的页请求接口都分配有特定数目的页请求消息信用量。RC或系统软件可以以任何合适的方式对信用量进行划分（比如在多个PRG见分配不同的信用量），RC采用任何方式来确保PRI正确计量信用量都是可行的。页请求接口不能采用超出所分配最大范围的请求，否则在超出root缓存上限后页请求机制会被关闭。页请求接口能够处理的页请数目上限是静态分配好的，在开启PRI的时候已经分配好，想要更改只能关闭并重启PRI。

&emsp;&emsp;对于PRG内有多笔页请求的情况，每一笔页请求消息占用一个信用量，而非一组页请求消息占用一个信用量。

<br>

>:warning:<font color=red>**注意**：
>- RC在收到页请求消息后不能选择忽视，必须进行响应，否则发送页请求消息的Function会一直等待。
>- 同一个请求组内包含多个针对相同页的页请求，或者多个PRG内请求同一页，均是合规的。
>- Function在不查看页状态情况下便一次发送针对该页的多笔页请求，仍然是合法的。
>- 页请求消息TC=0，RC收到TC不为0的页请求消息，应当做畸形包处理，Switch等中间路由节点不必关注该消息的TC字段。

<br>

## 2.2 PRG响应消息（PRG Response Message）
&emsp;&emsp;PRG响应消息是RC发给EP的消息，用于提示Function其请求的页已经准备好了（Pin在主存里了），EP可以采用ATS继续下一步操作了。PRG响应消息基于ID路由，路由到发出页请求的Function。对于PRG Index相同的一组页请求消息，RC只回复一次PRG响应消息，且RC在收到PRG最后一笔请求消息之前不会发送正常的请求响应消息（当然中间有可能收到响应出错的消息哇）。

<br>

### 2.2.1 PRG响应消息格式

![图2 PRG Response Message](https://img-blog.csdnimg.cn/062e39889dd4490188bede9e308135a5.png#pic_center)

<center><font size=2>图2 PRG Response Message<br><br>

&emsp;&emsp;PRG响应消息格式如图2所示，部分字段解释：
- **Type**：10010b，表示基于ID路由的Msg。
- **Message Code**：5，表示PRG响应消息。
- **Response Code**：用以指示页请求响应成功或失败，0000b -> 成功，0001 -> 无效请求，1111b -> 响应失败。
- **Page Request Group Index**：用以指示该响应消息所响应的PRG。

<br>

### 2.2.2 PRG响应顺序
&emsp;&emsp;对于多组PRG请求的情况，PRS机制不保证页请求完成的顺序。若某个Function要求在某个页请求B之前必须完成页请求A，那么Function最好是在收到页请求A的完成响应消息之后再发送页请求B。

<br>

### 2.2.3 PRG响应未成功
&emsp;&emsp;同一组内所有的页请求是同一根绳上的蚂蚱，旅进旅退。对于一组页请求消息，有可能部分页请求能够满足，部分页请求无法满足。对于这种情况，PRG响应消息统一判定为PRG请求失败，RC此时不会特意挑出满足页请求条件的页进行Pin。页请求失败的原因大致有以下几点：

1. 页请求的未转换地址无效。
2. 开启了PASID TLP Prefix，但部分页请求未携带PASID TLP Prefix或PASID无效（例如超出capabilitiy设置的范围）。
3. 页请求TLP Prefix中指示请求执行，但页请求消息中指示页不可读（R=0）。
4. 所请求的页不具备所要求的访问属性。
5. 由于某些原因导致系统无法响应页请求。

&emsp;&emsp;以上1~5点页请求失败跟系统动态运行有关，一次失败，重新发送一次是有可能成功的。如果PRG响应消息反馈了一笔预料之外的PRG Index，则把页请求能力结构页请求控制寄存器的对应位置1，便于软件发现并报告处理。

>:warning:<font color=red>**注意：**
>- Function在发出一系列页请求之后会等待RC反馈响应消息，此时在正常收发其他TLP的情况下该Function应有足够的能力处理来自RC的响应消息，确保不丢消息并正确解析，不然Function会一直等在那儿，会造成死锁。
>- PRG响应消息TC=0，EP收到TC不为0的PRG响应消息，应当做畸形包处理，Switch等中间路由节点不必关注该消息的TC字段。

<br>

## 2.3 带有PASID TLP Prefix的PRS
&emsp;&emsp;往期PASID TLP Prefix介绍请参考[【PASID TLP Prefix介绍】](https://blog.csdn.net/weixin_40357487/article/details/120480687)。

<br>

### 2.3.1 PASID TLP Prefix使用
&emsp;&emsp;支持PASID TLP Prefix的Function同样可以发送带有PASID TLP Prefix的页请求消息，PASID字段携带有所访问页的进程地址空间、执行请求指示位、特权模式请求指示位等信息。若PRG中某一页请求消息携带PASID TLP Prefix，那么该组所有页请求消息均需携带相同的PASID TLP Prefix，不允许有些携带有些不携带，也不允许携带不同的PASID TLP Prefix。

&emsp;&emsp;若页请求消息携带TLP Prefix，正常情况下RC反馈的PRG响应消息也携带PASID TLP Prefix。PRG响应消息TLP Prefix的PASID与对应页请求消息的TLP Prefix PASID相同，执行请求和特权模式请求位预留。

&emsp;&emsp;PASID TLP Prefix能力与ATS（Address Translation Services）及PRI（Page Request Interface）相互独立，具备PASID TLP Prefix能力的组件可以不支持ATS或PRI，支持ATS或PRI的组件也可以不支持PASID TLP Prefix。

<br>

### 2.3.2 PRG请求的PASID TLP Prefix管理
**（1）启动**
&emsp;&emsp;开启PASID TLP Prefix能力的Function在发送页请求消息时会携带PASID TLP Prefix。
<br>

**（2）结束** 
&emsp;&emsp;带有PASID TLP Prefix的页请求消息发送过程中，打算关闭了PASID功能，此时还没收到响应消息，怎么办？两种方法——

&emsp;&emsp;**不发送停止消息**。决定结束使用某PASID后，不能继续往待发送页请求消息队列中添加新的该PASID的页请求消息，发送L=1的该PASID页请求消息以结束发送，继续等待PRG响应消息并使用。<font color=red>（存疑，没太看懂）

&emsp;&emsp;**发送停止消息**。决定结束使用某PASID后，不能继续往待发送页请求消息队列中添加新的该PASID的页请求消息，发送L=1的该PASID页请求消息以结束发送，当收到PRG响应消息后，只对信用量进行回收，但并不进行后续的ATS访问（请求了个寂寞），想要ATS访问该页需重新发送页请求。Function发送停止标志消息后，EP可以再次发送带有该PASID的页请求消息，RC再次接收到该PASID的页请求消息时，已经是另一个新故事了。<font color=red>（存疑，没太看懂）

<br>

### 2.3.3 停止标记消息（Stop Marker Message）
&emsp;&emsp;如果页请求消息W和R字段都为0，说明啥？正常理解便是：我要访问一个页，不读也不写。此处用这种特殊的页请求消息来表示停止指定PASID的页请求。

&emsp;&emsp;停止标记消息用以指示Function已经发完所有某指定PASID的页请求消息，该消息子在该PASID页请求消息发送完之后发送，两者遵循强排序规则，确保停止标记消息在该PASID所有页请求消息之后到达RC。停止标记消息与页请求消息格式相同（图3），停止标记消息没有PRG Index，不占用页请求消息的信用量，RC在收到停止标志消息后不会回复任何响应消息。

&emsp;&emsp;发送停止标记消息是带有PASID TLP Prefix的，TLP Prefix中开启ID 排序，关闭宽松排序，TC为0；消息主体部分L=1,W=0， R=0，地址域和PRG Index域预留，Marker Type字段为0表示该消息为停止标记消息。

![图3 Stop Marker Message](https://img-blog.csdnimg.cn/4ae439c964aa4a77829f79dd2cc4480a.png#pic_center)

<center><font size=2>图3 Stop Marker Message<br><br>

&emsp;&emsp;停止标记消息出现以下几种异常情况时，至于RC作何反应，目前spec尚未做明确规定：

- RC在收到某PASID的停止标记消息之前未收到其最后一笔页请求消息
- Marker Typer非0
- 未携带TLP Prefix
- TLP Prefix携带的PASID跟进行中页请求消息PASID不符

<br>

---

<table><td bgcolor=blue><center><font size=5 color=white> — 3. PRS配置 —</center></td></table>

---

# 3. PRS配置
&emsp;&emsp;实现了页请求扩展能力结构的组件才能使用页请求服务。页请求能力结构只能在EP或RCiEP中实现。在SR-IOV系统中，PF及其对应的所有VF公用一个页请求扩展能力结构，各Function发出的页请求消息通过不同的Function id进行区分。

&emsp;&emsp;页请求扩展能力结构如图4所示，有扩展能力头标（图5）、页请求状态寄存器（图6）、页请求控制寄存器（图7）、outstanding的页请求能力及分配的outstanding页请求。各字段的意思从图中显而易见，不再赘述。

&emsp;&emsp;PRS reset后，对于的信用量计数器归位，未发出的页请求消息不再发送，未收到的PRG响应消息也不再等待。

![图4 Page Request Extended Capability Structure](https://img-blog.csdnimg.cn/8c25a90797464dcdb9e881c7ecd66cee.png#pic_center)

<center><font size=2>图4 Page Request Extended Capability Structure<br><br>

![图5 Page Request Extended Capability Header](https://img-blog.csdnimg.cn/2e97947f448942b591bafc1fd1f5d171.png#pic_center)

<center><font size=2>图5 Page Request Extended Capability Header<br><br>

![图6 Page Request Control Register](https://img-blog.csdnimg.cn/55e2525bcdcc4441ad158e1d4b8877bc.png#pic_center)

<center><font size=2>图6 Page Request Control Register<br><br>

![图7 Page Request Status Register](https://img-blog.csdnimg.cn/43e7860ccdd84d6d89c8ee6bc598057e.png#pic_center)

<center><font size=2>图7 Page Request Status Register<br><br>

<br>

---

<table><td bgcolor=blue><center><font size=5 color=white> — 4. 深入讨论 —</center></td></table>

---

# 4. 深入讨论

&emsp;&emsp;关于【页请求服务】，大家在讨论些什么？[【点击查看，欢迎跟帖讨论】](https://bbs.csdn.net/topics/603321828)

<br>

---

<table><td bgcolor=blue><center><font size=5 color=white> — 参考 —</center></td></table>

---

# 参考
1. PCI Express Base Specification Revision 5.0 Version 1.0 (22 May 2019)
2. [ARM SMMU Spec 1](https://documentation-service.arm.com/static/60804fe25e70d934bc69f12d?token=)
3. [ARM SMMU Spec 2](https://documentation-service.arm.com/static/5f900d34f86e16515cdc08fb?token=)




<br>

---
<table><td bgcolor=blue><center><font size=5 color=white> — END —</center></td></table>

---

><span id="pcie_protocol">:fire: <font color=red size=4>**精选往期 PCIe 协议系列文章**</font>:fire:</span> <br><br>
> - [【最新技术早知道】PCIe Gen5 还没用上，Gen6 就来了？PCIe 6.0 系列文章之：《PCIe 6.0，到底 6 在哪？》](https://blog.csdn.net/weixin_40357487/article/details/120714950?spm=1001.2014.3001.5501)
> - [【PCIe 6.0】颠覆性技术！你NRZ相守20年又怎样？看我PAM4如何上位PCIe 6.0 ！](https://blog.csdn.net/weixin_40357487/article/details/120775889)
 > - [PCIe事务排序（Transaction Ordering）](https://blog.csdn.net/weixin_40357487/article/details/120162461)
 > - [PCIe地址转换服务（ATS）详解](https://blog.csdn.net/weixin_40357487/article/details/120245027?spm=1001.2014.3001.5501)
 > - [PCIe访问控制服务（ACS）](https://blog.csdn.net/weixin_40357487/article/details/120295827?spm=1001.2014.3001.5501)
 > - [PCIe锁定事务（Locked Transactions）介绍](https://blog.csdn.net/weixin_40357487/article/details/120320613?spm=1001.2014.3001.5501)
 > - [PCIe TLP Prefix & PASID TLP Prefix介绍](https://blog.csdn.net/weixin_40357487/article/details/120480687?spm=1001.2014.3001.5501)

---
:arrow_up:**[ 返回顶部](#head)** :arrow_up:
