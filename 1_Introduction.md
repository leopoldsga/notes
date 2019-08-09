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
5. Among those userspace network stacks(mTCP, netmap, lwIP, ClickNF, F-stack, VP)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwNTIwNTM0MTgsMTgzNTc2NzMyMF19
-->