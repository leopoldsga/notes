Pipeline:
1. Web server的需求
	- Low latency
	- High performance
2. 协议栈对Web server的重要性
3. User space network stack has been proved to deliver high performance
	- For instance, DPDK, mTCP, netmap, VPP and Seastar. Snabb and OVS.
4. Fro, web server, Userspace stack should offers many features
	- Enough CPU resource
		- No context switches
		- Good cache locality
	- No memory copy
	- High packet processing speed
	- Full-stack functionality
5. Among those userspace network stacks(mTCP, netmap, lwIP, F-stack, VPP, Seastar), VPP is arguably the best choice for web servers.[Evidence here]
6. However, VPP has some limitations:
	- Handling packet lossing.
	- Message transferring
	- Lock overhead.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwMTA0MTM1NzksLTExOTkxNzMzNzcsLT
ExMTA3Nzc0MTcsMTUwNjA3MzE2OCwtMTgxNjMyMzg1MywxODM1
NzY3MzIwXX0=
-->