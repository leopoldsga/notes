# Introduction

Recently, the number of users of Web applications, specifically HTTP applications,

grows quickly. During some activities such as Double Eleven and 618 in China,

Black Friday in USA, the concurrent connections of those particular websites

stay a very big number.

Essentially, the HTTP Protocol is implemented based on the TCP/IP Protocols.

So the performance of network stack which establish, maintain and close so

many tcp connections, directly influence the performance of HTTP applications.

FD.io's [Vector Packet Processing](https://fdio-vpp.readthedocs.io/en/latest/overview/whatisvpp/index.html)(VPP) technology is fast, scalable and

deterministic packet processing stack that runs on commodity CPUs.VPP eliminates the context

switch overhead caused by socket APIs, and facilitates vector packet

processing to amortize the I-Cache overhead.

Generally speaking, VPP would be a good choice for web applications.In this paper, we aim to

use vpp as the user space host stack to providing integrated network stack for web applications

such as Nginx.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTgzNTc2NzMyMF19
-->