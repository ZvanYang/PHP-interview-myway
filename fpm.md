# fpm进程管理

介绍下三种不同的进程管理方式：

1. static: 这种方式比较简单，在启动时master按照pm.max_children配置fork出相应数量的worker进程，即worker进程数是固定不变的
2. dynamic: 动态进程管理，首先在fpm启动时按照pm.start_servers初始化一定数量的worker，运行期间如果master发现空闲worker数低于pm.min_spare_servers配置数(表示请求比较多，worker处理不过来了)则会fork worker进程，但总的worker数不能超过pm.max_children，如果master发现空闲worker数超过了pm.max_spare_servers(表示闲着的worker太多了)则会杀掉一些worker，避免占用过多资源，master通过这4个值来控制worker数
3. ondemand: 这种方式一般很少用，在启动时不分配worker进程，等到有请求了后再通知master进程fork worker进程，总的worker数不超过pm.max_children，处理完成后worker进程不会立即退出，当空闲时间超过pm.process_idle_timeout后再退出

worker会将自己的状态更新到fpm_scoreboard_proc_s->request_stage，master就是通过这个字段来判断worker是否是空闲。

**各自的优缺点**

1. 静态模式，不用动态扩容，提升性能。缺点：只需要考虑max_children数量，数量取决于cpu的个数和应用的响应时间，一次启动固定大小进程浪费系统资源
2. 动态模式，动态扩容， 不浪费资源；缺点：所有worker都在工作，新的请求到来需要等待创建worker进程，最长等待1s
3. 按需分配，会更节省资源。但是大流量下会使master变得很忙碌。

# fpm与NGINX的通信

fpm的master通过【共享内存】与worker进行通信，同时监听worker 的状态，已处理请求数。要杀死一个worker通过发送信号的方式来实现。
fpm的master进程与worker进程之间不会直接进行通信，master通过共享内存获取worker进程的信息，比如worker进程当前状态、已处理请求数等，当master进程要杀掉一个worker进程时则通过发送信号的方式通知worker进程。
![交互流程](/Users/zvan/code/mianshi/PHP-interview-myway/images/nginx/fastcgi.png)
![执行过程](/Users/zvan/code/mianshi/PHP-interview-myway/images/nginx/工作模式.png)

