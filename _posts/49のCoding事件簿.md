## 49のCoding事件簿
记录各种奇怪的bug和解决过程。
“事件簿”是因为debug时抽丝剥茧的过程和阅读推理小说有异曲同工之处。

### MPI & Blacs
---

2023/7/7
- 加了一个输入参数忘记bcast，程序卡死。后来每个进程分别输出，发现只有0号进程跑了后续流程，才找出这个问题。
（ChatGPT:）这种时候可以用gdb attach到进程上，然后用bt查看调用栈，找到卡死的进程，然后用frame命令切换到卡死的进程，然后用info locals查看局部变量，找到卡死的原因。
- `descinit_`报错：“{   -1,   -1}:  On entry to  DESCINIT parameter number    6 had an illegal value” 。ChatGPT说是因为没有正确初始化通信上下文。
  -  调用Cblacs_pinfo得到进程信息，发现Cblacs_pinfo竟然给出了进程数为-1。
  - 找到原因：前面有析构函数调用了Cblacs_exit(1)，删除了blacs环境。
  - 调用Cblacs_setup重新初始化不起作用：官方文档说除了PVM-blacs（Parallel Virtual Machine blacs）外，其他情况调这个函数没用（等同于Cblacs_pinfo，只输出不输入），因为blacs会假定系统的进程数是固定的。
  - 修改：在合理位置调用Cblacs_exit。其中Cblacs_exit(0)会同时终止MPI，Cblacs_exit(1)则不会。