[toc]



# GO

1. panic什么时候处理？
   1. 可以在defer里面的recover里面去捕获。

2. go的内存管理。
3. go的调度.

   1. go一般采用三级的调度。首先从runnext里面取goroutine，如果runnext没有，就去local queue去取。如果local queue没有，就去global queue里面去取。
   2. 
4. 往一个已经关闭的channel里面发送数据会怎么样？
   1. 会panic。

5. go的垃圾回收



