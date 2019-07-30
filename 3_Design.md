# Design

# 1 TCP retransmission handling

According to the TCP retransmission of VPP, we could give the following pipeline.

![](https://github.com/Guoao-Sun/notes/blob/master/Figures/](https://github.com/Guoao-Sun/notes/blob/master/Figures/)

## 1.1 Solution

We enable the `tcp_do_fastretransmits` module with one judge function that just drop the invalid tcp connections in retransmission queues of corresponding vcl worker. And our solution has been accepted by vpp community.

# 2 VLS lock optimization

| RPS | 0KB | 1KB | 10KB |100KB  |
|--|--|--|--|--|
| LDP with VLS | 196,603 | 123,617 | 95,386 | 26,956 |
| LDP without VLS | 212,816 | 145,495 | 114,220 | 29,328 |
| Improved rate | 8% | 17% | 20% | 8% |


Considering single-process applications, we find out that it is not necessary for VPP to enable VLS layer which locks specific vcl sessions during every vcl session access from applications. And those lock options really consume a lot considerable CPU resource that could be used for other CPU-bound functions of applications.

So we believe it a good choice to bypass the VLS layer for single-process applications and based on this idea, we remove the VLS layer of VPP and have done some tests which is showed in the sheet.

As for multi-process applications, TBD.

## 3 Consistency between event queues

Our primary solution is to abandon the `unhandled_evts_vector` and do not reset the `app message queue` for containing messages from VPP. In this case, attributes of every session retrieved from message queue could be kept correctly.

Using this primary idea, we provided one patched that is already accepted by vpp community.
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjA0MjQ5NjAsLTIwODg3NDY2MTJdfQ==
-->