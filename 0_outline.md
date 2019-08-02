# Outline

# 1 Introduction & Motivation

> 用户态协议栈：Userspace Network Stack ==>(acronym)==> UNS。
> 
> 内核网络协议栈： Kernelspace Network Stack ==>(acronym)==> KNS。

1.  明确所应对的现实场景。比如高性能网络、高性能web服务器等。
    
2.  论证，在当下的场景若要实现自身的目标（高性能），NS占据很大一部分权重。
    
3.  UNS在应对当下的场景，比KNS更为优化。具体的论据：（UNS的优势）
    
    -   降低上下文切换开销
        
    -   fast path（调研IX论文，明确是否存在该论点）
        
    -   容易定制（适用不同的场景，可以开发不同的定制化UNS）
        
4.  在该场景下，论证理想的UNS应当具有怎样的特性：（中括号对应具体的实现方式之一）
    
    -   尽量少的上下文切换开销；【bypass kernel——>提出大部分UNS的天然优势】
        
    -   Cache miss引入的性能波动尽量小；【batch processing——>凸显VPP与使用了批处理的协议栈】
        
    -   （可是当总结一般化VPP所独有的（或者少量UNS））
        
    -   协议栈进程与应用进程之间消息传递要更可靠；【可靠的消息传递机制——>暂时考虑对应第二个优化点】
        
    -   协议栈足够可靠、稳定；【不容易崩溃——>暂时考虑对应第一个优化点】
        
    -   因为UNS控制访问网络设备（NIC），对当下场景的多进程应用，UNS应支持多进程扩展；【单进程UNS支持多进程应用——>暂时考虑对应第三个优化点】
        
5.  列举UNS的学术背景与当下的状况，比对VPP，论证说应其比其他的UNS更具有竞争力。
    
    -   VPP基本满足理想的UNS所应具有的特性。
        
6.  以VPP为例，为该场景提供服务。存在的三个可优化的空间，论证说明其他的UNS【具有/不具有】同样的挑战。
    
7.  针对三个优化点，提出优化的解决方案。
    
    -   如何保证尾部丢包不崩溃【design#1】
        
    -   如何保证IPC可靠性（或者Robustness）【design#2】
        
    -   如何无锁扩展多进程（参考Kernel实现的SOREUSEPORT模块）【design#3】
        
8.  以VPP为例具体实现上述三种优化方案。
    
9.  测试Lockless LDP VPP.
    

1-6小节为Introduction&Motivation；7小节为Design；8小节为Implementation；9小节为Evaluation。
    

# 1.1 Challenges

## 1.1 TCP Retransmit Robustness

1.  如何处理链接尾部丢包问题。
2. 内核协议栈如何处理，VPP如何处理，其他的用户态网络协议栈如何处理；
3. 是否是共性问题；
    
4.  潜在的问题：TCP链接提前释放。应用进程对于是否丢包是无感知的，其行为在将有效数据发送完之后，会调用socket APIs中的“close”主动关闭sockets。
    
5.  VPP存在的问题：存在已被释放的tcp connection还有未被重传的数据报文在postponed延迟重传队列中。
    
    -   问题出现的时机：数据传递结束时。
        
    -   问题出现的标志：非法读取内存地址导致segment fault。
        
    -   问题出现的原因：链接的末尾位的数据报文丢失，导致其重传事件的处理落后tcp connection释放的事件。因为每个dispatch cycle中的session_queue_node_fn中transport_update_time内部会重传【上限为一个FRAME】上一次之前cycle(s)遗留下来的重传事件。
        

> So this happens under heavy load? That is we’re getting a large number of connections per second and all exchange a reasonable amount of data? What’s the number of connections we’re getting/s? Is it 3600 (=50*72)?
> 
> I’m asking because we limit the number of packets we can send/dispatch cycle to 1 frame (256 packets). So if we have a lot of connections, we may take several dispatch cycles until we are finally able to honor a postponed rxt. In the meantime, the connection could’ve been cleaned up by nginx.
> 
> > Nginx sends "close" would trigger "session_close" in VPP session api correspondingly, which instead generates one "SESSION_CTRL_EVT_CLOSE" type event in vpp event queue of the particular session worker. When VPP process the event in session_queue_node_fn if the first time, the event would be just postponed. If not the first and the tx queue is still not empty, try to wait for some dispatch cycles. If not both, VPP calls "session_transport_close" in which
> > 
> > -   If tx queue wasn't drained, change state to closed waiting for transport.This way, the transport, if it so wishes, can continue to try sending the outstanding data (in closed state it cannot). It MUST however at one point, either after sending everything or after a timeout, call delete notify.
> >     
> > -   This will finally lead to the complete cleanup of the session via "transport_close".
> >     
> 
> tcp_do_fastretransmits cannot generate more than 1 frame  (256) of packets. Once it reaches that, it postpones all remaining fast  retransmits.
> 
> tcp doesn’t generate more than 256 retransmitted packets per  dispatch cycle. This will change in the future and session layer won’t generate  more than 1 frame of packets (including retransmits) per dispatch cycle.
> 
> ——Florin

TCP拥塞控制通过设定timer进行congestion control and recovery.

1.  Packet loss: 当服务器收到三次duplicated ack数据包之后，判定packet loss发生。同时default RTO(Retransmission Timeout: default 3000 ms)开始计时并重传Dup ACK之后的包；
    
2.  [tcp_retries](https://stackoverflow.com/questions/5227520/how-many-times-will-tcp-retransmit)：当重传次数达到上限，tcp session会被关闭、释放。
    

> The majority of us are well aware of the primary retransmission logic. On the initial packet sequence, there is a timer called Retransmission Timeout (RTO) that has an initial value of three seconds. After each retransmission the value of the RTO is doubled and the computer will retry up to three times. This means that if the sender does not receive the acknowledgement after three seconds (or RTT > 3 seconds), it will resend the packet. At this point the sender will wait for six seconds to get the acknowledgement. If the sender still does not get the acknowledgement, it will retransmit the packet for a third time and wait for 12 seconds, at which point it will give up.
> 
> ——[Understanding RTT Impact on TCP Retransmissions](https://blog.catchpoint.com/2014/04/29/understanding-rtt-impact-on-tcp-retransmissions/)

VPP中TCP链接释放情况如下：

在VPP中，无数据传输【没有数据接收/发送】的tcp connection耗尽生存周期后，被VPP进程通过调用`tcp_connection_cleanup`函数释放链接。

需要知道的是，重传发生的时机。

> At runtime, the VPP platform grabs all available packets from RX rings to form a vector of packets. A packet processing graph is applied, node by node (including plugins) to the entire packet vector.
> 
> ——[Modular, Flexible, and Extensible](https://wiki.fd.io/view/VPP/What_is_VPP%3F)

1.  每个Dispatch cycle都会执行一次[完整的处理图](https://wiki.fd.io/view/VPP/What_is_VPP%3F)，也就表示每次dispatch cycle都绕不开一个节点，叫做`session_queue_node_fn`，该节点的GDB信息如下：
    
    (gdb) p *node  
    $4 = {cacheline0 = 0x7f5f2a63f3c0 "\360\001Kl_\177", function = 0x7f5f6c4b01f0 <session_queue_node_fn>, errors = 0x7f5f2a649f40,  
     clocks_since_last_overflow = 2643842100, max_clock = 31310300, max_clock_n = 0, calls_since_last_overflow = 15045310, vectors_since_last_overflow = 0,  
     perf_counter0_ticks_since_last_overflow = 0, perf_counter1_ticks_since_last_overflow = 0, perf_counter_vectors_since_last_overflow = 0,  
     next_frame_index = 958, node_index = 306, input_main_loops_per_call = 0, main_loop_count_last_dispatch = 113731361, main_loop_vector_stats = {0, 0},  
     flags = 0, state = 0, n_next_nodes = 4, cached_next_index = 0, thread_index = 0, runtime_data = 0x7f5f2a63f412 ""}
    
2.  每次调用`session_queue_node_fn`节点都会通过函数`session_update_dispatch_period`与函数`transport_update_time`，而`transport_update_time`函数又会间接调用`tcp_do_fastretransmits`处理丢包重传；
    
3.  那么重传时找不到相应的tcp connection表示该tcp链接已经被cleanup；
    
4.  VPP在以下几种情况下会主动cleanup tcp connection：
    
    -   建立新链接时无法在session层为新链接分配新session结构；
        
    -   调用`session_stream_accept`新链接创建session失败时cleanup对应的tcp connection；
        
    -   `tcp_connection_reset`、`tcp_connection_close`与`tcp_connection_del`被调用后；
        
        -   `tcp_connection_del`在链接进入CLOSED后才会被调用；
            
        -   `tcp_connection_reset`在链接被reset时调用；
            
        -   `tcp_connection_close`在接收LAST ACK或者TIME_WAIT时段耗尽后会被调用；
            
    -   `tcp_timer_establish_ao_handler`找不到`tcp_connection_t`时；
        
5.  即是正常的TCP链接会在进入CLOSED状态或者收到LAST ACK后被cleanup；
    
6.  在正常的负载情况下，1个FRAME(256)的上限足以应付所丢的数据包重传工作。但是对于大文件如1M文件的传输，在网络拥塞严重时，丢包量增多，但是一个dispatch cycle仅能在"tcp_do_fastretransmits"处重传不超出1个FRAME(256)数量的数据包。也就导致大量的丢失掉的数据包积压在"postponed_fast_rxt"队列。
    

故而，当网络负载大导致大量丢包时，很大概率使得重传队列中超出多个"FRAME"位置的tcp connection被提前释放，导致重传失败。

# 测试

Nginx调用close之后多久，VPP会cleanup tcp connection。

### 1.1.1 应对大量丢包问题

总的来说，这是一个如何应对“大量丢包”状况的问题。

### 1.1.2 丢包问题

当应用进程处理的网络负载过大，网卡带宽不足导致丢包或者因为客户端的延迟过高、网络状态不稳定导致的大量丢包问题出现时，正常处理逻辑时使用TCP的拥塞控制算法进行快速重传。

但是VPP的矢量包处理特性使其每次处理图迭代最多重传一个FRAME也就是256个数据包。当丢包数量达到一定程度比如10K时，部分需要重传的包因为重传上限低的原因滞留在重传队列中，严重时这些需要被重传的数据包所属的tcp链接因消耗完生存时间而被释放后才能开始被处理。因此，重传时无法获取有效的tcp connection属性信息，导致非法内存访问的问题，从而引起VPP守护进程的崩溃。这是网络拥塞导致的丢包问题。

### 1.1.3 tcp链接丢失

FRR(Fast Retransmit and Recovery)使用timer监控丢失的数据包并进行快速恢复。

而丢失的数据包过多，导致重传队列尾部重传周期过长，可以理解为超出VPP的有效重传队列长度，导致tcp链接丢失。可以视之为retransmit congestion。

所以如何有效缓解Retransmit Congestion问题，是我们需要解决的第一个挑战。

> 根据[Packet Dropping Policies for ATM and IP networks](https://ieeexplore.ieee.org/abstract/document/5340708/)论文
> 
> There are two board approaches of congestion management: congestion control and recovery, and congestion avoidance.

## 1.2 Multiple queues communication based on shared memory

1.  VPP采用基于共享内存的消息队列的方式向应用进程传递事件信息；
    
2.  message queue作为信息载体的运行机制；
    
3.  在应用进程接收到事件信息事前，信息被转化存储在不同的队列中，多队列有其存在的理由（避免队列过长，解耦不同状态事件的存储方式）；
    
4.  不同的中间队列引入内存与管理开销，并且增大事件信息在传递过程中出错的风险；
    
5.  VPP事件所携带的有效信息本身保存在原始的队列中(app event queue)，其根据事件的状态(上次迭代的遗留事件与本次迭代的事件)分别存储时间的索引；导致后续处理出错；
    
    1.  VPP的message queue作为存储事件信息的载体；
        
    2.  将事件返回给上层应用时，根据第一队列中的索引处理并返回app_event_queue中具体的事件信息；
        
    3.  第一队列中未处理完的索引存储在第二队列并且释放app_event_queue中具体的事件信息；
        
    4.  二次epoll wait处理第二队列的事件时无法根据其索引找到app_event_queue中的事件，因为已经被释放，所使用的内存可能已经被重复使用；
        

## 1.3 Lock in Socket API

1.  为提供向后兼容性，VPP提供LDP使用方式；
    
2.  通用接口底层实现细节使用了读写锁；
    
3.  VLS层的锁对于I/O密集型的应用来说，耗费了可观的CPU资源；
    
4.  如何在兼容多进程的基础上降低socket APIs下层锁的开销是我们需要面对的第三个挑战。
    

三个挑战如下：

挑战一

如何应对重传崩溃

挑战二

如何保证多队列下信息传递的准确性

挑战三

如何在保证向后兼容特性下降低接口层锁的开销

# 2 Design

PDP策略是网络congestion的一种缓解方式。我们可以将之应用于本地重传拥塞。

## 2.1 PDP应用于Retransmit Congestion Problem

VPP的实现支持TCP Newreno与CUBIC算法。但不能保证不丢包，参考PDP策略，遇无效链接的重传恢复，直接选择放弃重传。具体实现方式为实现**连接状态感知**的TCP recovery。

## 2.2 事件传递方式

1.  依据事件所属的迭代次数分别存储事件有其优势：
    
    1.  避免事件队列过长；
        
    2.  迭代之间事件存储队列解耦；
        
2.  也有其劣势：
    
    1.  增大内存开销；
        
    2.  增加管理开销；
        
    3.  增加事件传输过程中出错的风险。
        
3.  提出单队列事件存储，解决的问题：
    
    1.  降低事件出错风险；
        
    2.  降低内存与管理开销；
        
4.  单队列也有其局限性：
    
    1.  队列长度可能过长；
        
    2.  动态增加队列长度（vec_resize）本身也会引入内存拷贝开销。
        

## 2.3 降低接口层锁开销

**StraIghtforward方式**：去锁

1.  单进程：因为单进程不需要与别的应用进程共享资源；所以vls锁对于单进程是冗余的。
    
2.  多进程：父子进程之间存在共享的listen session，单纯的去锁之后无法保证子进程能够服务。
    
    1.  所以提出**复制共享资源**的方式，解决共享listen session在去锁情况下无法正常执行的问题。
        

# 3 Implementation

## 3.1 TCP connection状态感知

添加判断函数。

## 3.2 单队列事件存储

仅使用一个队列`mq_msg_vector`.

## 3.3 复制共享资源

子进程fork之后重新建立监听链接。

# 4 Evaluation

## 4.1 TCP connection状态感知

VPP重传遇无效TCP链接不崩溃。

## 4.2 单队列事件存储

单队列存储事件降低了事件在传输的过程中出错的概率。

## 4.3 复制共享资源

去锁之后单进程、多进程应用都能正常使用LDP VPP，且吞吐量有不低的提升。
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTgwMzY5Nzc1NywyMTc0NTM1ODgsLTQyND
IzMTQ5NSwtNzkxNTIyMTk3LDEyMDk0MTg0MiwxODY1MTQ4ODMy
LC0xMTQzODAxNDM3LC0xNDk5MTI1MzAwLDEwNDE4MDcyMTBdfQ
==
-->