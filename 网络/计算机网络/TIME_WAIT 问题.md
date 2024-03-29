## TIME_WAIT 问题

### TIME_WAIT 发生的场景

假设当你发现线上的服务的可用性变得时好时坏，一段时间可以对外提供服务，一段时间突然又不可以，登录到线上机器使用 netstat 命令可能会发现，主机上有成千上万处于 TIME_WAIT 状态的连接。

为什么 TIME_WAIT 会导致该问题呢？我们的应用服务器需要通过发起 TCP 连接对外提供服务。每个连接会占用一个本地端口，挡在高并发的情况下，TIME_WAIT 状态的连接过多，躲到把本机可用的端口耗尽，应用服务对外表现得症状，就是不能正常工作了。当过一段时间之后，处于 TIME_WAIT 的连接被系统回收并关闭后，释放出本地端口可供使用，应用服务对外表现为，可以正常工作。这样周而复始，便会出现了一会儿不可以，过一两分钟又可以正常工作的现象。

为什么会产生这么多的 TIME_WAIT 连接呢？

![四次挥手](image/四次挥手.png)

TCP 连接终止时，主机 1 先发送 FIN 保温，主机 2 进入 CLOSE_WAIT 状态，并发送一个 ACK 应答，同时，主机 2 通过 read 调用获得 EOF，并将此结果通知应用程序进行主动关闭操作，发送 FIN 报文。主机 1 在接收到 FIN 报文发送 ACK 应答，此时主机 1 进入 TIME_WAIT 状态。

主机 1 在 TIME_WAIT 停留持续时间时固定的，是最长分节生命期 MSL（maximum segment lifetime）的两倍，一般称之为 2MSL。和大多数 BSD 派生的系统一样，Linux 系统里有一个硬编码的字段，名称为 TCP_TIMEWAIT_LEN，其值为 60 秒。也就是说，Linux 系统停留在 TIME_WAIT 的时间为固定的 60 秒。

```c
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-        WAIT state, about 60 seconds	*/
```

过了这个时间之后，主机 1 就进入了 CLOSE 状态。

**只有发起连接终止的一方会进入 TIME_WAIT 状态。**

### TIME_WAIT 的作用

涉及两个方面

首先，这样做是为了确保最后的 ACK 能让被动关闭方接收，从而帮助其正常关闭。

TCP 在设计的时候，做了充分的容错性设计，比如，TCP 假设报文会出错，需要重传。在这里，如果图中主机 1 的 ACK 报文没有传输成功，那么主机 2 就会重新发送 FIN 报文。如果主机 1 没有维护 TIME_WAIT 状态，而直接进入 CLOSE 状态，它就失去了当前状态的上下文，只能回复一个 RST 操作，从而导致被动关闭方出现错误。现在主机 1 知道自己处于 TIME_WAIT 状态，就可以在接收到 FIN 报文之后，重新发出一个 ACK 报文，使得主机 2 可以进入正常的 CLOSE 状态。

第二个原因和连接 “化身” 和报文迷走有关系，为了让旧连接的重复分节在网络中自然消失。

在网络中，经常会发生报文经过一段时间才能到达目的地的情况，产生的原因是多种多样的，如路由器重启，链路突然出现故障等。如果迷走报文到达时，发现 TCP 连接四元组（源IP，源端口，目的 IP，目的端口）所代表的连接不复存在，那么很简单，这个保温自然丢弃。

我们考虑这样一个场景，在原连接中断后，又重新创建了一个原连接的 “化身”，说是化身其实是因为这个连接和原先的连接四元组完全相同，如果迷失报文经过一段时间也到达，那么这个报文会被误认为是连接“化身”的一个 TCP 分节，这样就会对 TCP 通信产生影响。

所以，TCP 就设计出了这么一个机制，经过 2MSL 这个时间，足以让两个方向上的分组都被丢弃，使得原来连接的分组在网络中都自然消失，再出现的分组一定都是新化身所产生的。

### TIME_WAIT 的危害

过多的 TIME_WAIT 的主要危害有两种。

第一是内存资源占用，这个目前看来不是太严重，基本可以忽略。

第二是对端口资源的占用，一个 TCP 连接至少消耗一个本地端口。要知道，端口资源也是有限的，一般可以开启的端口为 32768 ~ 61000，也可以通过 `net.ipv4.ip_local_port_range` 指定，如果 TIME_WAIT 状态过多，会导致无法创建新连接。这个也是我们在一开始讲到的那个例子。

### 如何优化 TIME_WAIT？ 

在高并发的情况下，如果我们想对 TIME_WAIT 做一些优化，来解决我们一开始提到的例子，该如何办呢？

#### ip_conntrack

顾名思义就是跟踪连接。一旦激活了此模块，就能在系统参数里发现很多用来控制网络连接状态超时的设置，其中自然也包括 TIME_WAIT：

```shell
shell> modprobe ip_conntrack
shell> sysctl net.ipv4.netfilter.ip_conntrack_tcp_timeout_time_wait
```

我们可以尝试缩小它的设置，比如十秒，甚至一秒，具体设置成多少合适取决于网络情况而定，但 ip_conntrack 引入的问题可能比解决的还多，比如性能会大幅下降，所以不建议使用。

#### tcp_tw_recycle

顾名思义就是回收 TIME_WAIT 连接。可以说这个内核参数已经变成了大众处理 TIME_WAIT 的万金油，如果你在网络上搜索 TIME_WAIT 的解决方案，十有八九会推荐设置它，不过这里隐藏着一个不易察觉的陷阱：

当多个客户端通过 NAT 方式联网并与服务端交互时，服务端看到的是同一个 IP，也就是说对服务端而言这些客户端实际上等于同一个，可惜由于这些客户端的时间戳可能存在差异，于是乎从服务端的视角看，便可能出现时间戳错乱的现象，进而直接导致时间戳小的数据包被丢弃。参考：[tcp_tw_recycle和tcp_timestamps导致connect失败问题](http://blog.sina.com.cn/s/blog_781b0c850100znjd.html)。

#### tcp_max_tw_buckets

控制 TIME_WAIT 总数。[官网文档](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)说这个选项只是为了阻止一些简单的 Dos 攻击，平常不要人为降低它。如果缩小了它，那么系统会将多余的 TIME_WAIT 删除掉，日志里会显示：`TCP: time wait bucket table overflow`

通过 sysctl 命令，将系统值调小。这个值默认为 18000，当系统中处于 TIME_WAIT 的连接一旦超过这个值时，系统就会将所有的 TIME_WAIT 连接状态重置，并且只打印出警告信息。这个方法过于暴力，而且治标不治本，带来的问题远比解决的问题多，不推荐使用。

#### 调低 TCP_TIMEWAIT_LEN，重新编译系统

这时一个不错的方法，缺点是需要 一点 内核方面知识，能够重新编译内核。

#### SO_LINGER 的设置

英文单词 ”linger“ 的意思为停留，我们可以通过设置套接字选项，来设置调用 close 或者 shutdown 关闭连接时的行为。

```c
int setsockopt(int sockfd, int level, int optname, const void *optval, socklent_t optlen);

struct linger {
    int l_onoff;    /* 0=off, nonzero=on */
    int l_linger;   /* linger time, POSIX specifies units as seconds */
}
```

设置 linger 参数有几种可能：

1. 如果 l_onoff 为 0，那么关闭本选项。l_linger 的值被忽略，这对应了默认行为，close 或 shutdown 立即返回。如果在套接字发送缓冲区中有数据残留，系统会将试着把这些数据发送出去。
2. 如果 l_onoff 为非 0，且 l_linger 值也为 0，那么调用 close 后，会立刻发送一个 RST 标志给对端，该 TCP 连接将跳过四次挥手，也就跳过了 TIME_WAIT 状态，直接关闭。这种关闭的方式称为 ”强行关闭“。在这种情况下，排队数据不会被发送，被动关闭方也不知道对端已经彻底断开。只有当被动关闭方正阻塞在 recv() 调用上时，接受到 RST 时，会立刻得到一个 ”connet reset by peer“ 的异常。

```c
struct linger so_linger;
so_linger.l_onoff = 1;
so_linger.l_linger = 0;
setsockopt(s, SOL_SOCKET, SO_LINGER, &so_linger, sizeof(so_linger));
```

3. 如果 l_onoff 为非0，且 l_linger 的值也非 0，那么调用 close 后，调用 close 的线程就将阻塞，直到数据被发送出去，或者设置的 l_linger 计时时间到。

第二种可能为跨院 TIME_WAIT 状态提供了一个可能，不过是一个非常危险的行为，不值得提倡。

#### tcp_tw_reuse

顾名思义就是复用 TIME_WAIT 连接。当创建新连接的时候，如果可能的话会考虑复用相应的 TIME_WAIT 连接。通常认为 `tcp_tw_reuse` 比 `tcp_tw_recycle` 安全一些，这时因为一来 TIME_WAIT 创建时间超过一秒才可能会被复用；二来只有连接的时间戳是递增的时候才会被复用。

Linux 系统对于 net.ipv4.tcp_tw_reuse 的解释如下：

```c
Allow to reuse TIME-WAIT sockets for new connections when it is safe from protocol viewpoint. Default value is 0.It should not be changed without advice/request of technical experts.
```

这段话的大意是从协议角度理解如果是安全可控的，可以复用处于 TIME_WAIT 的套接字为新的连接所用。

那么什么是协议角度理解的安全可控呢？主要有两点：

1. 只适用于连接发起方（C/S 模型中的客户端）；
2. 对应的 TIME_WAIT 状态的连接创建时间超过 1 秒才可以被复用。

使用这个选项，还有一个前提，需要打开对 TCP 时间戳的支持，即 `net.ipv4.tcp_timestamps=1` (默认即为1)。

为什么说只能在连接的发起方，而不能在被连接方使用。举例来说：客户端向服务端发起 HTTP 请求，服务端响应后主动关闭连接，于是 TIME_WAIT 便留在了服务端，此类情况使用 `tcp_tw_reuse` 是无效的，因为服务端是被连接方，所以不存在复用连接一说。

TCP 协议也在与时俱进，RFC 1323 中实现了 TCP 拓展规范，以便保证 TCP 的高可用，并引入了新的 TCP 选项，两个 4 字节的时间戳字段，用于记录 TCP 发送方的当前时间戳和从对端接收到的最新时间戳。由于引入了时间戳，前面提到的 2MSL 问题就不复存在了，因为重复的数据包会因为时间戳过期被自然丢弃。





### 参考资料

- 极客时间《网络编程实战》
- https://blog.huoding.com/2013/12/31/316