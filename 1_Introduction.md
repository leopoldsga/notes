Pipeline:
1. Web server的需求
	- Low latency
	- High performance
2. 协议栈对Web server的重要性
3. User space network stack outperforms stack in kernel
4. Fro, web server, Userspace stack should offers many features
	- Enough CPU resource
		- No context switches
		- Good cache locality
	- No memory copy
	- High packet processing speed
	- Full-stack functionality
5. Among those userspace network stacks(mTCP, netmap, lwIP, F-stack, VPP, Seastar), VPP is arguably the best choice for web servers.[Evidence here]
6. However, VPP has some limitations
	- Handling packet lossing.
	- Message transferring
	- Lock overhead.
<!--stackedit_data:
eyJoaXN0b3J5IjpbNTUzMDY4NDI3LDE1MDYwNzMxNjgsLTE4MT
YzMjM4NTMsMTgzNTc2NzMyMF19
-->