Pipeline:
1. Web server的需求
	- Low latency
	- High performance
		- Enough CPU resource
			- Eliminate/Mitigate context switch overhead
			- Good cache locality
		- High packet processing rate
		- Eliminate/Mitigate memory copy
2. 协议栈对Web server的重要性
3. User space network stack outperforms stack in kernel
4. Fro, web server, Userspace stack should offers many features
	- Enough CPU resource
		- No context switches
		- Good cache locality
	- No memory copy
	- 
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwMDQ4MzgyNzgsMTgzNTc2NzMyMF19
-->