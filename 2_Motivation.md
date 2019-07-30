# Motivation

# 1 Robustness of TCP layer

VPP has implemented TCP layer which follows TCP/IP protocols in RFCs.

Exploiting several functional nodes to compose one integrated packet

processing graph, VPP is capable of decoupling one integrated network

function into standalone graph node which only processes the simplified

processing logic such as tcp listening.

One packet vector,also called as one frame, is processed by VPP host

stack node by node each time, to amortize the Instruction Cache miss

overhead caused by loading new graph node. By default, the size of one frame

would be 256 which means about 256 packets would traverse the whole processing

graph within one specific dispatch cycle. Then one-time graph processing

is represented as one dispatch cycle in VPP.

Based on VPP framework, it is easy to know the following packet processing pipeline:

1.  The network card device receives packets and stores them in its RX ring buffer.
    
2.  According to the FRAME size such as 256, VPP retrieves one packet vector
    
    from the RX ring of the interface.
    
3.  The packet vector is processed by the function graph node by node, rather than one packet traverses several function modules one by one. For instance, the frame is transferred to the `dispatch_node` which is the first graph node, and the next node after the frame has been processed.
    
4.  VPP sends data from the virtual memory shared with external applications to TX ring buffer of the network card.
    
5.  VPP retrieves next packet frame and send it to dispatch node.
    
6.  The `session_queue_node_fn` node in the session layer receives the frame. Before it starts to process the frame, this node has to update the transport time specifically in which the `tcp_do_fastretransmits` function would be called to retransmits those lost packets.
    
7.  VPP continually processes the frame.
    

Under heavy workload such as 3600 clients concurrently sending HTTP requests to

remote HTTP server which has connected to VPP via `LD_PRELOAD`, VPP has to spend

several dispatch cycles to sending data to clients. If all the clients are requiring

large files like 1MB html file, it would take almost three cycles to finish sending

1MB data of one tcp connection out. The normal procedure is that the clients and

remote server establish tcp connections after which the server sending data and

closes the tcp connections.

The TCP layer or the `session_queue_node_fn` function node would have to

call `tcp_do_fastretransmits` to retransmit the lost packets before process

the next new frame/packet vector further. One problem is that the external application

served by VPP has no privilege to know the information of dropped packets. For http server, it just actively close connections that has finished transferring data, which makes the VPP correspondingly clean up those sessions of the tcp connections. Those closed connections possibly have packets needed to be retransmitted in the relative retransmit queues held by VPP tcp worker data structure.

The first challenge we facing here is that accessing attributes of closed connections

would crash VPP daemon because of segment fault caused by reading undefined address.

# 2 VLS lock optimization

VPP provides an integrated host stack framework to process network flows correctly and ensures high degree of backend compatibility which means external applications have no need to modify APIs to utilize VPP host stack.

As for the methods of making use of VPP host stack , there are three ways for applications to utilize.

First of all, modifying socket APIs inside application is one way to make VPP its user space network stack. However, this method needs to change source codes of application network logic module, which inevitably introduce developing and maintenance overhead. So it is not the optimal choice for many applications of which the developers are focusing on functionalities.

Secondly, partitioning the application logic into different `processing graph nodes` to achieve same function inside VPP. In that case, developing time,logic rearranged overhead and considerable maintenance overhead make it not the best way to integrate VPP host stack.

Finally, VPP supports LD_PRELOAD for external applications to utilize its host stack via dynamic shared library. Based on the `libvcl_ldpreload.so` dynamic library provided, networking I/O interfaces in applications such as `socket`,`epoll_wait` will just trigger functions implemented in vppcom library of VPP with the same names. This solution introduces several advantages for applications:

1.  High degree of backend compatibility which means the applications have no need to modify source code to take advantage of VPP host stack.
    
2.  The applications are capable of switching to kernel network stack flexibly for cases of different scenarios.
    
3.  This method is one easy-to-achieve way to make good use of VPP host stack.
    

According to our observations on the usage information of CPU sensitive networking application, vls lock mechanism consumes considerable cpu resource, which could be removed to implement **lockfree ldp** for the applications.

## 2.1 vls layer

In our experimental environments, nginx is used as a http server, which is sensitive to both CPU resource and networking IO resource.

However, before apps use vppcom session APIs to create sessions for communicating with remote clients, it has to lock the specific vcl locked session which is designed to separate vcl workers and external applications.

For instance, the `socket` api would be intercepted by `libvcl_ldpreload.so` when the http server calls `socket` to create socket file descriptor in order to listening to specific IP address and port. Then this `socket` would trigger `vls_create` function in vls layer to create sessions used by VPP host stack. It is necessary to look insight into the source code of `vls_create`.

### 2.1.1 vls_create

vls_handle_t  
vls_create (uint8_t proto, uint8_t is_nonblocking)  
{  
 vcl_session_handle_t sh;  
 vls_handle_t vlsh;  
​  
 vls_mt_guard (0, VLS_MT_OP_SPOOL);  
 sh = vppcom_session_create (proto, is_nonblocking);  
 vls_mt_unguard ();  
 if (sh == INVALID_SESSION_ID)  
 return VLS_INVALID_HANDLE;  
​  
 vlsh = vls_alloc (sh);  
 if (PREDICT_FALSE (vlsh == VLS_INVALID_HANDLE))  
 vppcom_session_close (sh);  
​  
 return vlsh;  
}  
​  
static vls_handle_t  
vls_alloc (vcl_session_handle_t sh)  
{  
 vcl_locked_session_t *vls;  
​  
 vls_table_wlock ();  
 pool_get (vlsm->vls_pool, vls);  
 vls->session_index = vppcom_session_index (sh);  
 vls->worker_index = vppcom_session_worker (sh);  
 vls->vls_index = vls - vlsm->vls_pool;  
 hash_set (vlsm->session_index_to_vlsh_table, vls->session_index,  
 vls->vls_index);  
 clib_spinlock_init (&vls->lock);  
 vls_table_wunlock ();  
 return vls->vls_index;  
}

According to the source code showed above, ldp layer would create `vcl_locked_session_t` for each session which works just like file descriptor on linux network stack. And the `vls_alloc` functions would further exploit read/write lock of one global data structure named `vls_main_t` to update attributes of the new created vls session.

After the listen session or listen file descriptor is created correctly, `vls_accept`, `vls_write` functions would be called to accept new tcp connections and write data to new sessions established for processing requests from remote clients.

vls_handle_t  
vls_accept (vls_handle_t listener_vlsh, vppcom_endpt_t * ep, int flags)  
{  
 vls_handle_t accepted_vlsh;  
 vcl_locked_session_t *vls;  
 int sh;  
​  
 if (!(vls = vls_get_w_dlock (listener_vlsh)))  
 return VPPCOM_EBADFD;  
 if (vcl_n_workers () > 1)  
 vls_mp_checks (vls, 1 /* is_add */ );  
 vls_mt_guard (vls, VLS_MT_OP_SPOOL);  
 sh = vppcom_session_accept (vls_to_sh_tu (vls), ep, flags);  
 vls_mt_unguard ();  
 vls_get_and_unlock (listener_vlsh);  
 if (sh < 0)  
 return sh;  
 accepted_vlsh = vls_alloc (sh);  
 if (PREDICT_FALSE (accepted_vlsh == VLS_INVALID_HANDLE))  
 vppcom_session_close (sh);  
 return accepted_vlsh;  
}  
​  
int  
vls_write (vls_handle_t vlsh, void *buf, size_t nbytes)  
{  
 vcl_locked_session_t *vls;  
 int rv;  
​  
 if (!(vls = vls_get_w_dlock (vlsh)))  
 return VPPCOM_EBADFD;  
​  
 vls_mt_guard (vls, VLS_MT_OP_WRITE);  
 rv = vppcom_session_write (vls_to_sh_tu (vls), buf, nbytes);  
 vls_mt_unguard ();  
 vls_get_and_unlock (vlsh);  
 return rv;  
}  
​  
static vcl_locked_session_t *  
vls_get_w_dlock (vls_handle_t vlsh)  
{  
 vcl_locked_session_t *vls;  
 vls_table_rlock ();  
 vls = vls_get_and_lock (vlsh);  
 if (!vls)  
 vls_table_runlock ();  
 return vls;  
}  
​  
static vcl_locked_session_t *  
vls_get_and_lock (vls_handle_t vlsh)  
{  
 vcl_locked_session_t *vls;  
 if (pool_is_free_index (vlsm->vls_pool, vlsh))  
 return 0;  
 vls = pool_elt_at_index (vlsm->vls_pool, vlsh);  
 clib_spinlock_lock (&vls->lock);  
 return vls;  
}

Based on the source code of vls, we could notice that each vls option such as `vls_create`, `vls_accept` and `vls_write` would enable the write lock of the global structure named `vlsm`. That happens because the status of the member of `vlsm` would be changed, and lock is used to keep the status of those members of `vlsm` correct and robust.

However, this lock consumes too much cpu resource of CPU sensitive applications such as a http server processing high-concurrency networking flows. As a result, how to optimize the overhead introduced by this vls lock mechanism becomes our second challenge.

# 3 Consistency between event queues

Epoll is supported by VPP for applications using epoll as event-driven communication mechanism. In this very aspect, every vcl worker representing one application process, contains one `app_event_queue` queue to which VPP sends events. In addition, one vcl worker has `mq_msg_vector` that is one vector acting as buffer for message queue messages from VPP, and one session event queue named `unhandled_evts_vector` in which the unhandled session events are stored until next `epoll wait` cycle.

During each `epoll wait` call triggered by external process, all the events from VPP would be dequeued into the message queue named `mq_msg_vector` and then be put into events returned to the particular process. Those events having not been sending to applications, are stored in the `unhandled_evts_vector` events queue and wait for next epoll wait call.

According to logic of epoll in VPP, it processes events in `unhandled_evts_vector` queue first before put events from`mq_msg_vector`queue into events queue, which are transformed into epoll events compatible with epoll mechanism on kernel.

![]([https://github.com/Guoao-Sun/notes/tree/master/Figures/vppcom_epoll_wait.png)

These two queues work fine with normal application which is not I/O sensitive. Because there would not be such many networking events during each dispatch cycle of VPP that one `epoll wait` cycle (default to be 512 maxevents) is not capable of handling all the events. So the second queue named `unhandled_evts_vector` is likely to containing no session events.

As for I/O sensitive applications such as http server who may have to processing millions of tcp connections at a time, which inevitably incurs more than 512 or 4096 events each dispatch cycle. Then the remaining unhandled events kept in `mq_msg_vector` would be put into the `unhandled_evts_vector` and returned to application process with the following `epoll wait`.

typedef struct svm_msg_q_ring_  
{  
 volatile u32 cursize;  /**< current size of the ring */  
 u32 nitems;  /**< max size of the ring */  
 volatile u32 head;  /**< current head (for dequeue) */  
 volatile u32 tail;  /**< current tail (for enqueue) */  
 u32 elsize;  /**< size of an element */  
 u8 *data;  /**< chunk of memory for msg data */  
} __clib_packed svm_msg_q_ring_t;  
​  
typedef struct  
{  
 u8 event_type;  
 u8 postponed;  
 union  
 {  
 u32 session_index;  
 session_handle_t session_handle;  
 session_rpc_args_t rpc_args;  
 struct  
 {  
 u8 data[0];//There has real message info in the followed continuous memory.  
 };  
 };  
} __clib_packed session_event_t;  
​  
typedef struct session_accepted_msg_  
{  
 u32 context;  
 u64 listener_handle;  
 u64 handle;  
 uword server_rx_fifo;  
 uword server_tx_fifo;  
 u64 segment_handle;  
 uword vpp_event_queue_address;  
 transport_endpoint_t rmt;  
} __clib_packed session_accepted_msg_t;

VPP is not capable of ensuring that events kept in the two event queues give same attributes. That is because that VPP exploit one zero-length array and dynamically allocating memory to store message in session event structure.

And when `vcl_epoll_wait_handle_mq_event` processes those events in `unhandled_evts_vector`, and it is likely to get the wrong content from the recorded address in those events. This happens because the message queue is freed once all messages stored in its event rings are dequeued into `mq_msg_vector` and `unhandled_evts_vector` queues. Then the address recorded by the mentioned zero-length array somehow may be reset and allocated again such that those address will not record event attributes.

How to make sure that the two queues be consistent with each other is the third challenge we are going to handle.
<!--stackedit_data:
eyJoaXN0b3J5IjpbOTY3MjU0OTMxLDg4Njg1Nzk4XX0=
-->