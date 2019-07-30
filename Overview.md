# Overview
> 用户态协议栈：Userspace Network Stack ==>(acronym)==> UNS。
> 
> 内核网络协议栈： Kernelspace Network Stack ==>(acronym)==> KNS。

1.  明确所应对的现实场景。比如高性能网络、高性能web服务器等。
    
2.  论证，在当下的场景若要实现自身的目标（高性能），UNS占据很大一部分权重。
    
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
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzE4OTMxMjQzXX0=
-->