IM系统架构演进之路

一、引言
====


今天，越来越多的用户被马蜂窝持续积累的笔记、攻略、嗡嗡等优质的分享内容所吸引，在这里激发了去旅行的热情，同时也拉动了马蜂窝交易的增长。在帮助用户做出旅行决策、完成交易的过程中，IM 系统起到了重要的作用。  

IM 系统为用户与商家建立了直接沟通的渠道，帮助用户解答购买旅行产品中的问题，既促成了订单交易，也帮用户打消了疑虑，促成用户旅行愿望的实现。伴随着业务的快速发展，几年间，马蜂窝 IM 系统也经历了几次比较重要的架构演化和转型。  

本文将分享马蜂窝旅游网的IM系统架构从零演进的整个过程，希望能给你的IM技术选型和方案确定带来启发。  

**系列文章：**  


*   《[从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路](http://www.52im.net/thread-2675-1-1.html)》（\* 本文）
*   《[从游击队到正规军(二)：马蜂窝旅游网的IM客户端架构演进和实践总结](http://www.52im.net/thread-2796-1-1.html)》
*   《[从游击队到正规军(三)：基于Go的马蜂窝旅游网分布式IM系统技术实践](http://www.52im.net/thread-2909-1-1.html)》  
    

**关于马蜂窝旅游网：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_timg.jpg](static/image/common/none.gif "从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_timg.jpg")

**timg.jpg** _(8.19 KB, 下载次数: 1158)_

[下载附件](http://www.52im.net/forum.php?mod=attachment&aid=ODA0NHwxOGYyMTczM3wxNzI1NjE3OTExfDB8MjY3NQ%3D%3D&nothumb=yes)  [保存到相册](javascript:;)

5 年前 上传

  

马蜂窝旅游网是中国领先的自由行服务平台，由陈罡和吕刚创立于2006年，从2010年正式开始公司化运营。马蜂窝的景点、餐饮、酒店等点评信息均来自上亿用户的真实分享，每年帮助过亿的旅行者制定自由行方案。

  

二、相关文章
======

  

*   《[一套海量在线用户的移动端IM架构设计实践分享(含详细图文)](http://www.52im.net/thread-812-1-1.html)》
*   《[一套原创分布式即时通讯(IM)系统理论架构方案](http://www.52im.net/thread-151-1-1.html)》
*   《[从零到卓越：京东客服即时通讯系统的技术架构演进历程](http://www.52im.net/thread-152-1-1.html)》
*   《[蘑菇街即时通讯/IM服务器开发之架构选择](http://www.52im.net/thread-31-1-1.html)》
*   《[以微博类应用场景为例，总结海量社交系统的架构设计步骤](http://www.52im.net/thread-1910-1-1.html)》
*   《[一套高可用、易伸缩、高并发的IM群聊、单聊架构方案设计实践](http://www.52im.net/thread-2015-1-1.html)》
*   《[腾讯QQ1.4亿在线用户的技术挑战和架构演进之路PPT](http://www.52im.net/thread-158-1-1.html)》
*   《[微信技术总监谈架构：微信之道——大道至简(演讲全文)](http://www.52im.net/thread-200-1-1.html)》
*   《[如何解读《微信技术总监谈架构：微信之道——大道至简》](http://www.52im.net/thread-201-1-1.html)》
*   《[快速裂变：见证微信强大后台架构从0到1的演进历程（一）](http://www.52im.net/thread-168-1-1.html)》
*   《[瓜子IM智能客服系统的数据架构设计（整理自现场演讲，有配套PPT）](http://www.52im.net/thread-2807-1-1.html)》
*   《[阿里钉钉技术分享：企业级IM王者——钉钉在后端架构上的过人之处](http://www.52im.net/thread-2848-1-1.html)》  
    

  

三、IM 1.0：初期阶段
=============

特点：低并发

功能：消息收发，消息队列，http长连接


初期为了支持业务快速上线，且当时版本流量较低，对并发要求不高，IM 系统的技术架构主要以简单和可用为目的，实现的功能也很基础。  

IM 1.0 使用 PHP 开发，实现了 IM 基本的用户/客服接入、消息收发、咨询列表管理功能。用户咨询时，会通过平均分配的策略分配给客服，记录用户和客服的关联关系。用户/客服发送消息时，通过调用消息转发模块，将消息投递到对方的 Redis 阻塞队列里。收消息则通过 HTTP 长连接调用消息轮询模块，有消息时即刻返回，没有消息则阻塞一段时间返回，这里阻塞的目的是降低轮询的间隔。  

**消息收发模型如下图所示：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_1.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061831231.jpeg)

  

上图模型中消息轮询模块的长连接请求是通过 php-fpm 挂载在阻塞队列上，当该请求变多时，如果不能及时释放 php-fpm 进程，会对服务器性能消耗较大，负载很高。  

为了解决这个问题，我们对消息轮询模块进行了优化，选用基于 OpenResty 框架，利用 Lua 协程的方式来优化 php-fmp 长时间挂载的问题。Lua 协程会通过对 Nginx 转发的请求标记判断是否拦截网络请求，如果拦截，则会将阻塞操作交给 Lua 协程来处理，及时释放 php-fmp，缓解对服务器性能的消耗。  

**优化的处理流程见下图：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_2.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061832482.jpeg)  

四、IM 2.0：需求定制阶段
===============


伴随着业务的快速增长，IM 系统在短期内面临着大量定制需求的增加，开发了许多新的业务模块。面对大量的用户咨询，客服的服务能力已经招架不住。  

**因此，IM 2.0 将重心放在提升业务功能体验上，比如：**  


*   1）在处理用户的咨询：平均、权重、排队
*   2）为了提升客服的效率，客服的咨询回复也增加了自动回复、FAQ 。  


以一个典型的用户咨询场景为例，当用户打开 App 或者网页时，会通过连接层建立长连接，之后在咨询入口发起咨询时，会携带着消息线索初始化消息链路，建立一条可复用、可检索的消息线；发送消息时，通过消息服务将消息存储到 DB 中，同时会根据消息线检索当前咨询是否被分配到客服，调用分配服务的目的是为当前咨询完善客服信息；最后将客服信息更新到链路关系中。  

这样，一条完整的消息链路就建立完毕，之后用户/客服发出的消息通过转发服务传输给对方。  

**以上处理流程如下图所示：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_3.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061845567.jpeg)

  

五、IM 3.0：服务拆分阶段
===============

  

5.1、概述
------

为了削弱各模块之间的耦合，采用微服务架构

拆分成4大服务：

客服服务，用户服务，IM服务，数据服务


业务量在不断积累，随着模块增加，IM 系统的代码膨胀得很快。由于代码规范没有统一、接口职责不够单一、模块间耦合较多等种原因，改动一个需求很可能会影响到其它模块，使新需求的开发和维护成本都很高。  

为了解决这种局面，IM 系统必须要进行架构升级，首要任务就是服务的拆分。目前，经过拆分后的 IM 系统整体分为 4 块大的服务，包括客服服务、用户服务、IM 服务、数据服务。  

![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_4.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061846482.jpeg)

  

**如上图，我们来进行一下解读：**  


*   **1）客服服务：**围绕提升客服效率和用户体验提供多种方式，如提供群组管理、成员管理、质检服务等来提升客服团队的运营和管理水平；通过分配服务、转接服务来使用户的接待效率更灵活高效；支持自动回复、FAQ、知识库服务等来提升客服咨询的回复效率等；
*   **2）用户服务：**分析用户行为，为用户做兴趣推荐及用户画像，以及统计用户对马蜂窝商家客服的满意度；
*   **3）IM 服务：**支持单聊和群聊模式，提供实时消息通知、离线消息推送、历史消息漫游、联系人、文件上传与存储、消息内容风控检测等；
*   **4）数据服务：**通过采集用户咨询的来源入口、是否咨询下单、是否有客服接待、用户咨询以及客服回复的时间信息等，定义数据指标，通过数据分析进行离线数据运算，最终对外提供数据统计信息。主要的指标信息有 30 秒、1 分钟回复率、咨询人数、无应答次数、平均应答时间、咨询销售额、咨询转化率、推荐转化率、分时接待压力、值班情况、服务评分等。  
    

  

5.2、用户状态流转
----------

**现有的 IM 系统 中，用户咨询时一个完整的用户状态流转如下图所示：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_5.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061847890.jpeg)

  

**如上图所示：**  


*   1）用户点击咨询按钮触发事件，此时用户状态进入初始态；
*   2）发送消息时，系统更改用户状态为待分配，通过调用分配服务分配了对应的客服后，用户状态更改为已分配、未解决；
*   3）当客服解决了用户或者客服回复后用户长时间未说话，触发系统自动解决的操作，此时用户状态更改为已解决，一个咨询流程结束。  

  

5.3、IM 服务的重构
------------


在服务拆分的过程中，我们需要考虑特定服务的通用性、可用性和降级策略，同时需要尽可能地降低服务间的依赖，避免由于单一服务不可用导致整体服务瘫痪的风险。  

在这期间，公司其它业务线对 IM 服务的使用需求也越来越多，使用频次和量级也开始加大。初期阶段的 IM 服务当连接量大时，只能通过修改代码实现水平扩容；新业务接入时，还需要在业务服务器上配置 Openresty 环境及 Lua 协程代码，业务接入非常不便，IM 服务的通用性也很差。  

考虑到以上问题，我们对 IM 服务进行了全面重构，目标是将 IM 服务抽取成独立的模块，不依赖其它业务，对外提供统一的集成和调用方式。考虑到 IM 服务对并发处理高和损耗低的要求，选择了 Go 语言来开发此模块（关于Go语言的详细实践总结，请详读《[从游击队到正规军(三)：基于Go的马蜂窝旅游网分布式IM系统技术实践](http://www.52im.net/thread-2909-1-1.html)》一文）。  

**新的 IM 服务设计如下图：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_6.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061847721.jpeg)

  

**其中，比较重要的 Proxy 层和 Exchange 层提供了以下服务：**  


*   **1）路由规则：**例如 ip-hash、轮询、最小连接数等，通过规则将客户端散列到不同的 ChannelManager 实例上；
*   **2）对客户端接入的管理：**接入后的连接信息会同步到 DispatchTable 模块，方便 Dispatcher 进行检索；
*   **3）ChannelManager 与客户端间的通信协议：**包括客户端请求建立连接、断线重连、主动断开、心跳、通知、收发消息、消息的 QoS 等；
*   **4）对外提供单发、群发消息的 REST 接口：**这里需要根据场景来决定是否使用，例如用户咨询客服的场景就需要通过这个接口下发消息。  
    


**针对上述第“4）”点，主要原因在以下 3 点：**  


*   1）发消息时会有创建消息线、分配管家等逻辑，这些逻辑目前是 PHP 实现，IM 服务需要知道 PHP 的执行结果，一种方式是使用 Go 重新实现，另外一种方式是通过 REST 接口调用 PHP 返回，这样会带来 IM 服务和 PHP 业务过多的网络交互，影响性能；
*   2）转发消息时，ChannelManager 多个实例间需要互相通信，例如 ChannelManager1 上的用户 A 给 ChannelManager2 上的客服 B 发消息，如果实例间无通信机制，消息无法转发。当要再扩展 ChannelManager 实例时，新增实例需要和其它已存在实例分别建立通信，增加了系统扩展的复杂度；
*   3）如果客户端不支持 WebSocket 协议，作为降级方案的 HTTP 长连接轮循只能用来收消息，发消息需要通过短连接来处理。其它场景不需要消息转发，只用来给 ChannelManager 传输消息的场景，可通过 WebSocket 直接发送。  
    

  

5.4、改造后的 IM 服务调用流程
------------------


初始化消息线及分配客服过程由 PHP 业务完成。需要消息转发时，PHP 业务调用 Dispatcher 服务的发消息接口，Dispatcher 服务通过共享的 Dispatcher Table 数据，检索出接收者所在的 ChannelManager 实例，将消息通过 RPC 的方式发送到实例上，ChannelManager 通过 WebSocket 将消息推送给客户端。  

**IM 服务调用流程如下图所示：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_7.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061848678.jpeg)

  

当连接数超过当前 ChannelManager 集群承载的上限时，只需扩展 ChannelManager 实例，由 ETCD 动态的通知到监听侧，从而做到平滑扩容。目前浏览器版本的 JS-SDK 已经开发完毕，其它业务线通过接入文档，就能方便的集成 IM 服务。  

**在 Exchange 层的设计中，有 3 个问题需要考。**  

**_1）多端消息同步：_**  

现在客户端有 PC 浏览器、Windows 客户端、H5、iOS/Android，如果一个用户登录了多端，当有消息过来时，需要查找出这个用户的所有连接，当用户的某个端断线后，需要定位到这一个连接。  

上面提到过，连接信息都是存储在 DispatcherTable 模块中，因此 DispatcherTable 模块要能根据用户信息快速检索出连接信息。DispatcherTable 模块的设计用到了 Redis 的 Hash 存储，当客户端与 ChannelManager 建立连接后，需要同步的元数据有 uid(用户信息)、uniquefield(唯一值，一个连接对应的唯一值)、wsid(连接标示符)、clientip(客户端 ip)、serverip(服务端 ip)、channel(渠道)，对应的结构大致如下:  

![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_8.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061848120.jpeg)

  

这样通过 key(uid) 能找到一个用户多个端的连接，通过 key+field 能定位到一条连接。连接信息的默认过期时间为 2 小时，目的是避免因客户端连接异常中断导致服务端没有捕获到，从而在 DispatcherTable 中存储了一些过期数据。  

**_2）用户在线状态同步：_**  

比如一个用户先后和 4 个客服咨询过，那么这个用户会出现在 4 个客服的咨询列表里。当用户上线时，要保证 4 个客服看到用户都是在线状态。  

要做到这一点有两种方案：  


*   一种是客服通过轮询获取用户的状态，但这样当用户在线状态没有变化时，会发起很多无效的请求；
*   另外一种是用户上线时，给客服推送上线通知，这样会造成消息扩散，每一个咨询过的客服都需要扩散通知。  
    


我们最终采取的是第二种方式，在推送的过程中，只给在线的客服推送用户状态。  

![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_9.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061849822.jpeg)

  

**_3）消息的不丢失，不重复：_**  

为了避免消息丢失，对于采用长连接轮询方式的我们会在发起请求时，带上客户端已读消息的 ID，由服务端计算出差值消息然后返回；使用 WebSocket 方式的，服务端会在推送给客户端消息后，等待客户端的 ACK，如果客户端没有 ACK，服务端会尝试多次推送。  

这时就需要客户端根据消息 ID 做消息重复的处理，避免客户端可能已收到消息，但是由于其它原因导致 ACK 确认失败，触发重试，导致消息重复。  

5.5、IM 服务的消息流
-------------


上文提到过 IM 服务需要支持多终端，同时在角色上又分为用户端和商家端，为了能让通知、消息在输出时根据域名、终端、角色动态输出差异化的内容，引入了 DDD （领域驱动设计）的建模方法来对消息进行处理。  

**处理过程如下图所示：**  
![从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路_10.jpg](https://raw.githubusercontent.com/yinhuiSpace/picgoimg/main/img/202409061849458.jpeg)

  

六、小结和展望
=======


伴随着马蜂窝「内容+交易」模式的不断深化，IM 系统架构也经历着演化和升级的不同阶段，从初期粗旷无序的模式走向统一管理，逐渐规范、形成规模。  

我们取得了一些进步，当然，还有更长的路要走。未来，结合公司业务的发展脚步和团队的技术能力，我们将不断进行 IM 系统的优化。  

目前我们正在计划将消息轮询模块中的服务端代码用 Go 替换，使其不再依赖 PHP 及 OpenResty 环境，实现更好地解耦；另外，我们将基于 TensorFlow 实现向智慧客服的探索，通过训练数据模型、分析数据，进一步提升人工客服的解决效率，提升用户体验，更好地为业务赋能。  

附录：更多IM架构设计方面的文章
================


**\[1\] 有关IM架构设计的文章：**  
《[浅谈IM系统的架构设计](http://www.52im.net/thread-307-1-1.html)》  
《[简述移动端IM开发的那些坑：架构设计、通信协议和客户端](http://www.52im.net/thread-289-1-1.html)》  
《[一套海量在线用户的移动端IM架构设计实践分享(含详细图文)](http://www.52im.net/thread-812-1-1.html)》  
《[一套原创分布式即时通讯(IM)系统理论架构方案](http://www.52im.net/thread-151-1-1.html)》  
《[从零到卓越：京东客服即时通讯系统的技术架构演进历程](http://www.52im.net/thread-152-1-1.html)》  
《[蘑菇街即时通讯/IM服务器开发之架构选择](http://www.52im.net/thread-31-1-1.html)》  
《[腾讯QQ1.4亿在线用户的技术挑战和架构演进之路PPT](http://www.52im.net/thread-158-1-1.html)》  
《[微信后台基于时间序的海量数据冷热分级架构设计实践](http://www.52im.net/thread-895-1-1.html)》  
《[微信技术总监谈架构：微信之道——大道至简(演讲全文)](http://www.52im.net/thread-200-1-1.html)》  
《[如何解读《微信技术总监谈架构：微信之道——大道至简》](http://www.52im.net/thread-201-1-1.html)》  
《[快速裂变：见证微信强大后台架构从0到1的演进历程（一）](http://www.52im.net/thread-168-1-1.html)》  
《[17年的实践：腾讯海量产品的技术方法论](http://www.52im.net/thread-159-1-1.html)》  
《[移动端IM中大规模群消息的推送如何保证效率、实时性？](http://www.52im.net/thread-1221-1-1.html)》  
《[现代IM系统中聊天消息的同步和存储方案探讨](http://www.52im.net/thread-1230-1-1.html)》  
《[IM开发基础知识补课(二)：如何设计大量图片文件的服务端存储架构？](http://www.52im.net/thread-1356-1-1.html)》  
《[IM开发基础知识补课(三)：快速理解服务端数据库读写分离原理及实践建议](http://www.52im.net/thread-1366-1-1.html)》  
《[IM开发基础知识补课(四)：正确理解HTTP短连接中的Cookie、Session和Token](http://www.52im.net/thread-1525-1-1.html)》  
《[WhatsApp技术实践分享：32人工程团队创造的技术神话](http://www.52im.net/thread-1542-1-1.html)》  
《[微信朋友圈千亿访问量背后的技术挑战和实践总结](http://www.52im.net/thread-1569-1-1.html)》  
《[王者荣耀2亿用户量的背后：产品定位、技术架构、网络方案等](http://www.52im.net/thread-1595-1-1.html)》  
《[IM系统的MQ消息中间件选型：Kafka还是RabbitMQ？](http://www.52im.net/thread-1647-1-1.html)》  
《[腾讯资深架构师干货总结：一文读懂大型分布式系统设计的方方面面](http://www.52im.net/thread-1811-1-1.html)》  
《[以微博类应用场景为例，总结海量社交系统的架构设计步骤](http://www.52im.net/thread-1910-1-1.html)》  
《[快速理解高性能HTTP服务端的负载均衡技术原理](http://www.52im.net/thread-1950-1-1.html)》  
《[子弹短信光鲜的背后：网易云信首席架构师分享亿级IM平台的技术实践](http://www.52im.net/thread-1961-1-1.html)》  
《[知乎技术分享：从单机到2000万QPS并发的Redis高性能缓存实践之路](http://www.52im.net/thread-1968-1-1.html)》  
《[IM开发基础知识补课(五)：通俗易懂，正确理解并用好MQ消息队列](http://www.52im.net/thread-1979-1-1.html)》  
《[微信技术分享：微信的海量IM聊天消息序列号生成实践（算法原理篇）](http://www.52im.net/thread-1998-1-1.html)》  
《[微信技术分享：微信的海量IM聊天消息序列号生成实践（容灾方案篇）](http://www.52im.net/thread-1999-1-1.html)》  
《[新手入门：零基础理解大型分布式架构的演进历史、技术原理、最佳实践](http://www.52im.net/thread-2007-1-1.html)》  
《[一套高可用、易伸缩、高并发的IM群聊、单聊架构方案设计实践](http://www.52im.net/thread-2015-1-1.html)》  
《[阿里技术分享：深度揭秘阿里数据库技术方案的10年变迁史](http://www.52im.net/thread-2050-1-1.html)》  
《[阿里技术分享：阿里自研金融级数据库OceanBase的艰辛成长之路](http://www.52im.net/thread-2072-1-1.html)》  
《[社交软件红包技术解密(一)：全面解密QQ红包技术方案——架构、技术实现等](http://www.52im.net/thread-2202-1-1.html)》  
《[社交软件红包技术解密(二)：解密微信摇一摇红包从0到1的技术演进](http://www.52im.net/thread-2519-1-1.html)》  
《[社交软件红包技术解密(三)：微信摇一摇红包雨背后的技术细节](http://www.52im.net/thread-2533-1-1.html)》  
《[社交软件红包技术解密(四)：微信红包系统是如何应对高并发的](http://www.52im.net/thread-2548-1-1.html)》  
《[社交软件红包技术解密(五)：微信红包系统是如何实现高可用性的](http://www.52im.net/thread-2564-1-1.html)》  
《[社交软件红包技术解密(六)：微信红包系统的存储层架构演进实践](http://www.52im.net/thread-2568-1-1.html)》  
《[社交软件红包技术解密(七)：支付宝红包的海量高并发技术实践](http://www.52im.net/thread-2573-1-1.html)》  
《[社交软件红包技术解密(八)：全面解密微博红包技术方案](http://www.52im.net/thread-2576-1-1.html)》  
《[社交软件红包技术解密(九)：谈谈手Q红包的功能逻辑、容灾、运维、架构等](http://www.52im.net/thread-2583-1-1.html)》  
《[即时通讯新手入门：一文读懂什么是Nginx？它能否实现IM的负载均衡？](http://www.52im.net/thread-2600-1-1.html)》  
《[即时通讯新手入门：快速理解RPC技术——基本概念、原理和用途](http://www.52im.net/thread-2620-1-1.html)》  
《[多维度对比5款主流分布式MQ消息队列，妈妈再也不担心我的技术选型了](http://www.52im.net/thread-2625-1-1.html)》  
《[从游击队到正规军(一)：马蜂窝旅游网的IM系统架构演进之路](http://www.52im.net/thread-2675-1-1.html)》  
《[从游击队到正规军(二)：马蜂窝旅游网的IM客户端架构演进和实践总结](http://www.52im.net/thread-2796-1-1.html)》  
《[IM开发基础知识补课(六)：数据库用NoSQL还是SQL？读这篇就够了！](http://www.52im.net/thread-2759-1-1.html)》  
《[瓜子IM智能客服系统的数据架构设计（整理自现场演讲，有配套PPT）](http://www.52im.net/thread-2807-1-1.html)》  
《[阿里钉钉技术分享：企业级IM王者——钉钉在后端架构上的过人之处](http://www.52im.net/thread-2848-1-1.html)》  
\>> [更多同类文章 ……](http://www.52im.net/forum.php?mod=collection&action=view&ctid=7)  

**\[2\] 更多其它架构设计相关文章：**  
《[腾讯资深架构师干货总结：一文读懂大型分布式系统设计的方方面面](http://www.52im.net/thread-1811-1-1.html)》  
《[快速理解高性能HTTP服务端的负载均衡技术原理](http://www.52im.net/thread-1950-1-1.html)》  
《[子弹短信光鲜的背后：网易云信首席架构师分享亿级IM平台的技术实践](http://www.52im.net/thread-1961-1-1.html)》  
《[知乎技术分享：从单机到2000万QPS并发的Redis高性能缓存实践之路](http://www.52im.net/thread-1968-1-1.html)》  
《[新手入门：零基础理解大型分布式架构的演进历史、技术原理、最佳实践](http://www.52im.net/thread-2007-1-1.html)》  
《[阿里技术分享：深度揭秘阿里数据库技术方案的10年变迁史](http://www.52im.net/thread-2050-1-1.html)》  
《[阿里技术分享：阿里自研金融级数据库OceanBase的艰辛成长之路](http://www.52im.net/thread-2072-1-1.html)》  
《[达达O2O后台架构演进实践：从0到4000高并发请求背后的努力](http://www.52im.net/thread-2141-1-1.html)》  
《[优秀后端架构师必会知识：史上最全MySQL大表优化方案总结](http://www.52im.net/thread-2157-1-1.html)》  
《[小米技术分享：解密小米抢购系统千万高并发架构的演进和实践](http://www.52im.net/thread-2323-1-1.html)》  
《[一篇读懂分布式架构下的负载均衡技术：分类、原理、算法、常见方案等](http://www.52im.net/thread-2494-1-1.html)》  
《[通俗易懂：如何设计能支撑百万并发的数据库架构？](http://www.52im.net/thread-2510-1-1.html)》  
《[多维度对比5款主流分布式MQ消息队列，妈妈再也不担心我的技术选型了](http://www.52im.net/thread-2625-1-1.html)》  
《[从新手到架构师，一篇就够：从100到1000万高并发的架构演进之路](http://www.52im.net/thread-2665-1-1.html)》  
《[美团技术分享：深度解密美团的分布式ID生成算法](http://www.52im.net/thread-2751-1-1.html)》  
《[12306抢票带来的启示：看我如何用Go实现百万QPS的秒杀系统(含源码)](http://www.52im.net/thread-2771-1-1.html)》  
\>> [更多同类文章 ……](http://www.52im.net/forum.php?mod=collection&action=view&ctid=22)

[![即时通讯网 - 即时通讯开发者社区！](template/qu_115style/img/cngeeker_logo_32_32.png)](/thread-2675-1-1.html "即时通讯网 - 即时通讯开发者社区！") 来源：[即时通讯网](/thread-2675-1-1.html "即时通讯网 - 即时通讯开发者社区！") - 即时通讯开发者社区！

**标签:**[即时通讯架构](misc.php?mod=tag&id=88 "即时通讯架构") [移动IM开发](misc.php?mod=tag&id=17 "移动IM开发")

上一篇：[跟着源码学IM(二)：自已开发IM很难？手把手教你撸一个Andriod版IM](forum.php?mod=viewthread&tid=2671 "跟着源码学IM(二)：自已开发IM很难？手把手教你撸一个Andriod版IM")▪下一篇：[我想问下，MobileIMSDK和openfire哪个比较好呢？](forum.php?mod=viewthread&tid=2689 "我想问下，MobileIMSDK和openfire哪个比较好呢？")

//console.log('id=>>>>>>>>>>>>> postmessage\_'+13229); jQuery(function(){ function HNode(tagName,text,targetObj){ this.tagName = tagName; this.text = text; this.targetObj = ""; } // 1. 拿到array var headerTagArray = \['H1', 'H2', 'H3', 'H4', 'H5', 'H6'\]; var headerTagMap = new Map(\[\["H1",false\],\["H2",false\],\["H3",false\],\["H4",false\],\["H5",false\],\["H6",false\]\]); var maxHeaderTag = "H6"; var treeArray = \[\]; var jq\_tree = jQuery("<ul>"); jQuery.each(jQuery('#postmessage\_'+13229+' :header'), function(i, val) { var tagName = val.tagName; //console.log("X=====i="+i+", tagName="+tagName+", "+('#postmessage\_'+13229+' :header')); if(headerTagArray.indexOf(tagName) > -1) { // find max header tag if(tagName < maxHeaderTag){ maxHeaderTag = tagName; } // find all exist header tag if(!headerTagMap.get(tagName)){ headerTagMap.set(tagName,true) } var node = new HNode(tagName,jQuery(this).text()); node.targetObj = jQuery(this); treeArray.push(node); } }); jQuery("#js\_tiezi\_menuTree").append(jq\_tree); console.log(treeArray); console.log(maxHeaderTag); console.log(headerTagMap); //console.log("=====A"); // 2. 拿到一个层级 if(treeArray.length==0){ return; } var levelIndex = 1; headerTagMap.forEach(function (value, key, map) { if(value){ headerTagMap.set(key,"level"+levelIndex++); } }); console.log(headerTagMap); //console.log("=====B"); // 3. 拿到全部层级 var anchorIndex = 0 ; for(var p = 0 ; p < treeArray.length ; p++){ var hnode = treeArray\[p\]; var jq\_targetObj = hnode.targetObj; var anchorId = "ccccc" + anchorIndex++; jq\_targetObj.attr("id",anchorIndex); var className = headerTagMap.get(treeArray\[p\].tagName); if(className != false){ var jq\_li = q\_jq("<li class='"+(className)+"' ><a class = 'catalogue-item' rel=\\"nofollow\\" href='#"+(anchorIndex)+"'>"+hnode.text+"</a></li>"); jq\_tree.append(jq\_li); } } if((jq\_tree.children().size()) >=3){ jQuery(".g\_js\_tiezi\_menu\_zone").show(); } jQuery(document).on('click', '.catalogue-item', function(e) { var \_id = q\_jq(this).attr("href"); jQuery('html,body').animate({scrollTop: jQuery(\_id).offset().top}, 200); e.preventDefault(); return false; }); }); jQuery(document).ready(function () { jQuery(window).scroll(function () { var items = jQuery(":header","#postmessage\_"+13229); var menu = jQuery("#js\_tiezi\_menuTree"); var top = jQuery(document).scrollTop(); var currentId = ""; //滚动条现在所在位置的item id items.each(function () { var m = jQuery(this); //console.log(m.html()); //注意：m.offset().top代表每一个item的顶部位置 if (top > m.offset().top - 50) { currentId = "#" + m.attr("id"); //console.log("currentId= " + currentId); } else { return false; } }); var currentLink = menu.find(".active"); if(currentLink){ currentLink.removeClass("active"); } if(currentId == ""){ menu.find("li:eq(0)").addClass("active"); return ; } menu.find("\[href='" + currentId + "'\]").parent().addClass("active"); //console.log("====currentId " +currentId+" , "+ menu.find("\[href='" + currentId + "'\]").html()); //if (currentId && currentLink.attr("href") != currentId) { // currentLink.removeClass("current"); //menu.find("\[href=" + currentId + "\]").addClass("current"); // } }); }); jQuery(function(){ var top = jQuery(".g\_js\_tiezi\_menu\_zone").offset().top; jQuery(window).scroll(function(){ //console.log('AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA:'+jQuery(window).scrollTop() +" " + top); if(jQuery(window).scrollTop() >= top){ jQuery(".g\_js\_tiezi\_menu\_zone").addClass("g\_js\_tiezi\_menu\_fixed"); }else{ jQuery(".g\_js\_tiezi\_menu\_zone").removeClass("g\_js\_tiezi\_menu\_fixed"); } }); });

### 本帖已收录至以下技术专辑

*   [IM架构篇](http://www.52im.net/forum.php?mod=collection&action=view&ctid=7 "IM架构篇")|主题 92·关注 53

相关文章

*   [微信团队分享：微信后台在海量并发请求下是如何做到不崩溃的](http://www.52im.net/thread-3930-1-1.html "微信团队分享：微信后台在海量并发请求下是如何做到不崩溃的")
*   [微信Windows端IM消息数据库的优化实践：查询慢、体积大、文件损坏等](http://www.52im.net/thread-4034-1-1.html "微信Windows端IM消息数据库的优化实践：查询慢、体积大、文件损坏等")
*   [求教关于IM用户信息和群聊信息缓存的差异更新和拉取问题](http://www.52im.net/thread-4035-1-1.html "求教关于IM用户信息和群聊信息缓存的差异更新和拉取问题")
*   [为啥IM的app与web端发离线消息或查看历史消息都是中文乱码？](http://www.52im.net/thread-4041-1-1.html "为啥IM的app与web端发离线消息或查看历史消息都是中文乱码？")
*   [求解IM群聊中消息排序的问题，到底按什么排序合适](http://www.52im.net/thread-4043-1-1.html "求解IM群聊中消息排序的问题，到底按什么排序合适")
*   [求教关于IM用户在新端拉取上万条数据的优化问题](http://www.52im.net/thread-4048-1-1.html "求教关于IM用户在新端拉取上万条数据的优化问题")
*   [求教IM中的离线群聊消息大家都是怎么做的](http://www.52im.net/thread-4673-1-1.html "求教IM中的离线群聊消息大家都是怎么做的")
*   [求教IM服务端的聊天会话更新策略，群聊会话如何更新](http://www.52im.net/thread-4674-1-1.html "求教IM服务端的聊天会话更新策略，群聊会话如何更新")
*   [求助im中聊天消息的消息id是怎么生成的](http://www.52im.net/thread-4676-1-1.html "求助im中聊天消息的消息id是怎么生成的")
*   [求教IM系统中，聊天消息、会话等的数据库结构如何设计](http://www.52im.net/thread-4679-1-1.html "求教IM系统中，聊天消息、会话等的数据库结构如何设计")

推荐方案

*   [![MobileIMSDK-即时通讯网(52im.net)](template/qu_115style/img/52im_fangan2_mb.png)](http://www.52im.net/forum-89-1.html)
    
    ### [MobileIMSDK](http://www.52im.net/forum-89-1.html "MobileIMSDK")[(v6.5精编版)](https://shop143012492.taobao.com/ "捐助作者，得MobileIMSDK精编注释版！")
    
    轻量级开源移动端即时通讯框架。
    
    [快速入门](http://www.52im.net/thread-52-1-1.html) / [性能](http://www.52im.net/thread-57-1-1.html) / [指南](http://www.52im.net/forum.php?mod=collection&action=view&ctid=1) / [提问](http://www.52im.net/thread-60-1-1.html)
    
*   [![MobileIMSDK-Web-即时通讯网(52im.net)](template/qu_115style/img/52im_fangan2_mbweb.png)](http://www.52im.net/forum-110-1.html)
    
    ### [MobileIMSDK-Web](http://www.52im.net/forum-110-1.html "MobileIMSDK-Web")[(有偿开源)](https://shop143012492.taobao.com/ "支持作者，有偿获取MobileIMSDK-Web的全部源码和资料！")
    
    轻量级Web端即时通讯框架。
    
    [详细介绍](http://www.52im.net/thread-959-1-1.html) / [精编源码](https://item.taobao.com/item.htm?id=621500349058) / [手册教程](https://item.taobao.com/item.htm?id=556718940694)
    
*   [![RainbowAV-即时通讯网(52im.net)](template/qu_115style/img/52im_fangan2_av.png)](http://www.52im.net/forum-110-1.html)
    
    ### [RainbowAVnew](http://www.52im.net/forum-112-1.html "RainbowAV")[(有偿开源)](https://shop143012492.taobao.com/ "支持作者，有偿获取RainbowAV的全部源码和资料！")
    
    移动端实时音视频框架。
    
    [详细介绍](http://www.52im.net/thread-1027-1-1.html) / [性能测试](http://www.52im.net/thread-1039-1-1.html) / [安装体验](https://fir.xcxwo.com/w2cn)
    
*   [![RainbowChat-即时通讯网(52im.net)](template/qu_115style/img/52im_fangan2_rb.png)](http://www.52im.net/forum-90-1.html)
    
    ### [RainbowChat](http://www.52im.net/forum-90-1.html "RainbowChat")[(技术转让)](http://www.52im.net/thread-1115-1-1.html "查看RainbowChat 技术转让说明及授权协议书范本！")
    
    基于MobileIMSDK的移动IM系统。
    
    [详细介绍](http://www.52im.net/thread-19-1-1.html) / [产品截图](http://www.52im.net/thread-20-1-1.html) / [安装体验](https://fir.xcxwo.com/t4xw)
    
*   [![RainbowChat_Web-即时通讯网(52im.net)](template/qu_115style/img/52im_fangan2_rbweb.png)](http://www.52im.net/forum-115-1.html)
    
    ### [RainbowChat-Web](http://www.52im.net/forum-115-1.html "RainbowChat-Web")[(技术转让)](http://www.52im.net/thread-2489-1-1.html "查看RainbowChat-Web 技术转让说明及授权协议书范本！")
    
    一套产品级Web端IM系统。
    
    [详细介绍](http://www.52im.net/thread-2483-1-1.html) / [产品截图](http://www.52im.net/thread-2470-1-1.html) / [演示视频](http://www.52im.net/thread-2491-1-1.html)
    

aimgcount\[13229\] = \['8044','8045','8046','8047','8048','8049','8050','8051','8052','8054','8055'\]; attachimggroup(13229); attachimgshow(13229); var aimgfid = 0;

​    

[返回列表](http://www.52im.net/forum-103-1.html) [发新帖](javascript:; "发新帖") [发表评论](forum.php?mod=post&action=reply&fid=103&tid=2675&extra=page%3D1&page=1)

var connect\_qzone\_share\_url = ''; var connect\_weibo\_share\_url = ''; var connect\_thread\_info = { thread\_url: 'http://www.52im.net/thread-2675-1-1.html', thread\_id: '2675', post\_id: '', forum\_id: '103', author\_id: '', author: '' }; connect\_autoshare = ''; connect\_isbind = ''; if(connect\_autoshare == 1 && connect\_isbind) { \_attachEvent(window, 'load', function(){ connect\_share(connect\_weibo\_share\_url, connect\_openid); }); }

#### 精华之王

精华主题数超过100个。

#### 白金版主

连续任职达2年以上的合格正式版主

#### 终身成就

为论区做出突出贡献的开发者、版主等。

document.onkeyup = function(e){keyPageScroll(e, 0, 0, 'forum.php?mod=viewthread&tid=2675', 1);}

function succeedhandle\_followmod(url, msg, values) { var fObj = $('followmod\_'+values\['fuid'\]); if(values\['type'\] == 'add') { fObj.innerHTML = '不收听'; fObj.href = 'home.php?mod=spacecp&ac=follow&op=del&fuid='+values\['fuid'\]; } else if(values\['type'\] == 'del') { fObj.innerHTML = '收听TA'; fObj.href = 'home.php?mod=spacecp&ac=follow&op=add&hash=52fbffd5&fuid='+values\['fuid'\]; } } fixed\_avatar(\[13229\], 1);

打赏楼主 ×

![使用微信打赏！](template/qu_115style/img/js_wpay_s.png "使用微信打赏！") ![使用支付宝打赏！](template/qu_115style/img/js_apay_s.png "使用支付宝打赏！")

即时通讯技术的分享和传播需要您的支持，感谢打赏！

function showDashang(){ jQuery("#dashang\_main").show(); } function hideDashang(){ jQuery("#dashang\_main").hide(); } jQuery("#dashang\_show\_btn").click(function () { showDashang(); }); jQuery("#dashang\_close\_btn").click(function () { hideDashang(); });

var mw\_brush = eval({"applescript":\["AppleScript","true"\],"actionscript3":\["Actionscript3","true"\],"bash":\["Bash shell","true"\],"c":\["C","true"\],"cpp":\["C++","true"\],"csharp":\["C#","true"\],"css":\["CSS","true"\],"delphi":\["Delphi","true"\],"erlang":\["Erlang","true"\],"groovy":\["Groovy","true"\],"html":\["HTML","true"\],"java":\["Java","true"\],"javafx":\["JavaFX","true"\],"javascript":\["JavaScript","true"\],"pascal":\["Pascal","true"\],"perl":\["Perl","true"\],"php":\["PHP","true"\],"text":\["Plain Text","true"\],"powershell":\["PowerShell","true"\],"python":\["Python","true"\],"ruby":\["Ruby","true"\],"rails":\["Ruby on Rails","true"\],"sass":\["Sass","true"\],"scala":\["Scala","true"\],"scss":\["Scss","true"\],"shell":\["Shell","true"\],"sql":\["SQL","true"\],"vb":\["Visual Basic","true"\],"vbnet":\["Visual Basic .NET","true"\],"xhtml":\["XHTML","true"\],"xml":\["XML","true"\],"xslt":\["XSLT","true"\],"objc":\["Objective-C","true"\],"asm":\["Asm","true"\],"golang":\["Golang","true"\],"lua":\["Lua","true"\],"typescript":\["TypeScript","true"\]}); var mw\_gutter = 1; var mw\_lang\_codebox = { 'select\_lang': '请输入要插入的代码<br>选择语言：', 'show\_gutter': '显示行号：', 'your\_code': '你的代码：', 'close': '关闭', 'submit': '提交', 'cancle': '取消' }; mw\_syntaxhighlighter("fastpost");

#### 即时通讯网　

实时推送、IM等即时通讯相关技术的学习、交流与分享的平台。专业的资料、专业的人、专业的社区！让即时通讯技术能更好传播与分享。

平等 开放 分享 传承

商务/合作：[business@52im.net](#)  
投稿/报道：[contact@52im.net](#)

#### 友情链接[\[友链交换\]](http://www.52im.net/thread-402-1-1.html)

*   [UCloud用户社区](https://uclub.ucloud.cn "UCloud用户社区连接你我,UCloud用户可以在本社区提出你的疑问,分享你的经验")
*   [OpenSNS](http://www.opensns.cn "OpenSNS是基于OneThink的轻量级社交化用户中心框架，系统秉持简约的设计风格，注重交流，为用户提供了一套轻量级的社交方案。")
*   [网易云信](https://yunxin.163.com/?invite=zxkhsCFu "网易云信 - 融合通信云服务专家")
*   [网易易盾](https://dun.163.com/ "网易易盾-智能可靠的内容安全&业务安全服务")
*   [星环科技](https://www.transwarp.cn/ "星环科技致力于打造企业级大数据基础软件，围绕数据的集成、存储、治理、建模、分析、挖掘和流通等数据全生命周期提供基础软件与服务，构建明日数据世界。")
*   [即构RTC](https://www.zego.im/ "即构RTC")
*   [anyRTC](https://www.anyrtc.io/ "anyRTC--全球WebRTC先行者")
*   [融云](http://www.rongcloud.cn/ "安全，可靠的全球互联网通信云服务商")
*   [环信](http://www.easemob.com "即时通讯云服务商，连接人与人，连接人与商业")

#### 关于

*   [关于我们](topic-about.html?mobile=no)
*   [活跃QQ群](topic-qqgroup.html?mobile=no)
*   [在线文档](topic-docs.html?mobile=no)
*   [网址导航](topic-sitelist.html?mobile=no)
*   [广告投放 new](http://www.52im.net/thread-1188-1-1.html)

#### 微信公众号new

![即时通讯网微信公众号](template/qu_115style/img/52im_qr_small6.png "扫码关注即时通讯网微信公众号！")

—— 打开微信扫一扫，关注本站的公众号 ——

Copyright © 2014-2024 即时通讯网 - 即时通讯开发者社区 / 版本 V4.4

苏州网际时代信息科技有限公司 [(苏ICP备16005070号-1）](https://beian.miit.gov.cn/)

Processed in 0.125000 second(s), 37 queries , Gzip On.

**返回顶部**

\_attachEvent(window, 'scroll', function () { showTopLink(); });checkBlind();