#MVCC 事务2--逻辑时钟
### 逻辑时钟出现的背景
在MVCC的理论中，需要一个能力，能够识别整个系统中的所有事件（事务开始，事务提交）的先后关系。即，将系统中的所有事件都放在一个有向轴上，描述他们的先后关系，所有的事件是全序的。一般在分布式系统特别在分布式的数据库中把有向轴定义为时间轴，而一个事务通常包含一个开始的时间戳和一个结束的时间戳。

但是在分布式系统中，基本上做不到一个统一的时间轴（即使使用NTP服务，也会存在误差和时间跳变的问题， 哪怕是Google Spanner 的 True Time 依然使用的是一个时间范围），因为时间是相对的 （侠义相对论）。

于是Lamport 大师的逻辑时钟闪亮登场，在他定义的逻辑时钟算法里，给分布式系统里面定义了一个有向轴，把分布式系统里面的事件放在有向轴中，从而使得所有事件都达到全序关系状态。Lamport的这篇论文名为《Time, Clocks, and the Ordering of Events in a Distributed System》，在论文中将系统的事件分为三类：

* 本地事件 
* 发生消息事件
* 接收消息事件

###WIKi中关于逻辑时钟的定义

> A process increments its counter before each event in that process;
> When a process sends a message, it includes its counter value with the message;
> On receiving a message, the counter of the recipient is updated, if necessary, to the greater of its current counter and the timestamp in the received message. The counter is then incremented by 1 before the message is considered received.

发送消息的伪码实现：

    time = time + 1;
    time_stamp = time;
    send(message, time_stamp);
接收消息的伪码实现:

    (message, time_stamp) = receive();
    time = max(time_stamp, time) + 1;

###关于逻辑时钟的通俗理解

每个进程都有一个计数器，进程每发生一个事件，本地的计数器加1。

进程在执行一个发生消息的事件时，要把本地时间戳（进程计数器的值）带到消息内部，一个进程接到该消息的时候，要把本地的计数器的值设置为max（消息带的时间戳的值，本地计数器的当前值）。

在数据库中定义事务的开始时间点和提交时间，就可以使用**逻辑时钟**来定义。本地事件发生的逻辑时钟的时间就是本地计数器的值，这样，就可以使用逻辑时钟确定任意进程间任意事件的前后关系。

但是仅使用逻辑时钟会出现一个另外的问题： 从系统外部人的角度来看，通过逻辑时钟定义出的全序关系与实际物理时间上的发生先后顺序不一致，即逻辑时钟这个轴跟我们物理意义的时间轴并没有关系，比如事务A与事务B，在逻辑时钟这个轴上，可能是A先发生，B后发生，但是在物理时间上，可能是B先发生，A后发生。 如何解决这个问题，将会在下一篇介绍混合逻辑时钟和Spanner True Time的文章中讲解。


