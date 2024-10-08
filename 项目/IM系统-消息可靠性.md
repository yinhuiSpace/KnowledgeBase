IM消息机制（一）：保证在线实时消息的可靠投递
=======================

![作者头像](https://ask.qcloudimg.com/http-save/yehe-6090469/8c0ffd8a755c3c4da187a781cebed5af.jpeg)

### 一、概述

消息的可靠性，即消息的不丢失和不重复，是IM系统中的一个难点。当初QQ在技术上（当时叫OICQ）因为以下两点原因才打败了ICQ：

*   QQ的消息投递可靠（消息不丢失，不重复）
*   QQ的垃圾消息少（它antispam做得好，这也是一个难点，但不是本文重点讨论的内容）

### 二、报文类型

IM的客户端与服务器通过发送报文（也就是请求包）来完成消息的传递。

报文分为三种：

1.  请求报文（request，后简称为为R）
2.  应答报文（acknowledge，后简称为A）
3.  通知报文（notify，后简称为N）

这三种报文的解释如下：

![](https://ask.qcloudimg.com/http-save/yehe-6090469/dcbcd736b3c07d9e33d93df5a7dba2c1.png)

*   R：客户端主动发送给服务器的报文
*   A：服务器被动应答客户端的报文，一个A一定对应一个R
*   N：服务器主动发送给客户端的报文

### 三、普通消息投递流程

用户A给用户B发送一个“你好”，很容易想到，流程如下：

![](https://ask.qcloudimg.com/http-save/yehe-6090469/f9ae8b61eaaec12290caf66906e8a600.png)

1.  client-A向im-server发送一个消息请求包，即msg:R
2.  im-server在成功处理后，回复client-A一个消息响应包，即msg:A
3.  如果此时client-B在线，则im-server主动向client-B发送一个消息通知包，即msg:N（当然，如果client-B不在线，则消息会存储离线）

### 四、上述消息投递流程出现的问题

从流程图中容易看到，发送方client-A收到msg:A后，只能说明im-server成功接收到了消息，并不能说明client-B接收到了消息。在若干场景下，可能出现msg:N包丢失，且发送方client-A完全不知道，例如：

1.  服务器崩溃，msg:N包未发出
2.  网络抖动，msg:N包被网络设备丢弃
3.  client-B崩溃，msg:N包未接收

结论是悲观的：接收方client-B是否有收到msg:N，发送方client-A完全不可控，那怎么办呢？

### 五、应用层确认+im消息可靠投递的六个报文

我们来参考网络传输协议的实现：UDP是一种不可靠的传输层协议，TCP是一种可靠的传输层协议，TCP是如何做到可靠的？答案是：

超时、重传、确认。

消息送达保证机制

（实际上IM中，数据通讯层无论用的是UDP还是TCP协议，都同样需要消息送达保证（即QoS）机制，原因在于IM的通信是A端-Server-B端的3方通信，而非传统C/S或B/S这种2方通信）。

要想实现应用层的消息可靠投递，必须加入应用层的确认机制，即：要想让发送方client-A确保接收方client-B收到了消息，必须让接收方client-B给一个消息的确认，这个应用层的确认的流程，与消息的发送流程类似：

1.  client-B向im-server发送一个ack请求包，即ack:R
2.  im-server在成功处理后，回复client-B一个ack响应包，即ack:A
3.  则im-server主动向client-A发送一个ack通知包，即ack:N

至此，发送“你好”的client-A，在收到了ack:N报文后，才能确认client-B真正接收到了“你好”。

你会发现，一条消息的发送，分别包含（上）（下）两个半场，即msg的R/A/N三个报文，ack的R/A/N三个报文。**一个应用层即时通讯消息的可靠投递，共涉及6个报文，这就是im系统中消息投递的最核心技术** （如果某个im系统不包含这6个报文，不要谈什么消息的可靠性）。

### 六、可靠消息投递存在什么问题

期望六个报文完成消息的可靠投递，但实际情况下：

1.  msg:R，msg:A 报文可能丢失： 此时直接提示“发送失败”即可，问题不大
2.  msg:N，ack:R，ack:A，ack:N这四个报文都可能丢失： （原因如1.4所述，可能是服务器奔溃、网络抖动、或者客户端奔溃），此时client-A都收不到期待的ack:N报文，即client-A不能确认client-B是否收到“你好”

那怎么办呢？

### 七、消息的超时与重传

client-A发出了msg:R，收到了msg:A之后，在一个期待的时间内，如果没有收到ack:N，client-A会尝试将msg:R重发。可能client-A同时发出了很多消息，故client-A需要在本地维护一个等待ack队列，并配合timer超时机制，来记录哪些消息没有收到ack:N，以定时重发。

![](https://ask.qcloudimg.com/http-save/yehe-6090469/c3aa02a4ee980b4efedde500b4d6a427.png)

一旦收到了ack:N，说明client-B收到了“你好”消息，对应的消息将从“等待ack队列”中移除。

### 八、消息的重传存在什么问题

第六节提到过提到过，msg:N报文，ack:N报文都有可能丢失：

*   msg:N 报文丢失：说明client-B之前压根没有收到“你好”报文，超时与重传机制十分有效
*   ack:N 报文丢失：说明client-B之前已经收到了“你好”报文（只是client-A不知道而已），超时与重传机制将导致client-B收到重复的消息

**启示：**

平时使用qq，或许大伙都有类似的体验，弹出一个对话框“因为网络原因，消息发送失败，是否要重发”，此时，有可能是对方没有收到消息（发送方网络不好，msg:N丢失），也可能已经收到了消息（接收方网络不好，反复重传后，ack:N依然丢失），出现这个提示时，大伙不妨和对端确认一下，看是哪种情况。

### 九、消息的去重

解决方法也很简单，由发送方client-A生成一个消息去重的msgid，保存在“等待ack队列”里，同一条消息使用相同的msgid来重传，供client-B去重，而不影响用户体验。

### 十、其它

1.  上述设计理念，由客户端重传，可以保证服务端无状态性（架构设计基本准则）
2.  如果client-B不在线，im-server保存了离线消息后，要伪造ack:N发送给client-A
3.  离线消息的拉取，为了保证消息的可靠性，也需要有ack机制，但由于拉取离线消息不存在N报文，故实际情况要简单的多，即先发送offline:R报文拉取消息，收到offline:A后，再发送offlineack:R删除离线消息

### 十一、总结

1.  im系统是通过超时、重传、确认、去重的机制来保证消息的可靠投递，不丢不重
2.  切记，一个“你好”的发送，包含上半场msg:R/A/N与下半场ack:R/A/N的6个报文

个人消息是一个1对1的ack，群消息就没有这么简单了，群消息存在一个扩散系数，im群消息的可靠投递问题感兴趣的可查阅相关资料。
