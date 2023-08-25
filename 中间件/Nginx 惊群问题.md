### 什么是惊群

惊群简单来说就是多个进程或者线程在等待同一个事件，当事件发生时，所有线程和进程都会被内核唤醒。唤醒后通常只有一个进程获得了该事件并进行处理，其他进程发现获取事件失败后又继续进入了等待状态，在一定程度上降低了系统性能。
具体来说惊群通常发生在服务器的监听等待调用上，服务器创建监听 `socket`，后 `fork` 多个进程，在每个进程中调用 `accept` 或者 `epoll_wait` 等待终端的连接。
`accept` 和 `epoll_wait` 在历史上都存在惊群效应，但目前都已经解决了（具体内核版本待查）。`accept` 的惊群效应解决的比较早，至少在 2.6.18 的内核上没有遇到惊群问题。`epoll_wait` 的惊群问题在这个 patch epoll: add EPOLLEXCLUSIVE flag 21 Jan 2016 已经解决了，通过添加 `EPOLLEXCLUSIVE` 标志标识在唤醒时，只唤醒一个等待进程。相比而言，内核在解决 `accept` 的惊群时是作为一个问题进行了修复，即无需设置标志，而对 `epoll_wait` 则作为添加一个功能选项，这主要是因为 `accept` 等待的是一个 `socket`，并且这个 `socket` 的连接只能被一个进程处理，内核可以很明确的进行这个预设，因此 accept 只唤醒一个进程才是更优的选择。而对于 `epoll_wait`，等待的是多个 `socket` 上的事件，有连接事件，读写事件等等，这些事件可以同时被一个进程处理，也可以同时被多个进程分别处理，内核不能进行唯一进程处理的假定，因此提供一个设置标志让用户决定。
在建立连接的时候，`Nginx` 处于充分发挥多核 CPU 架构性能的考虑，使用了多个 `worker` 子进程监听相同端口的设计，这样多个子进程在 accept 建立新连接时会有争抢，这会带来著名的"惊群"问题，子进程数量越多越明显，这会造成系统性能的下降。
一般情况下，有多少 CPU 核心就有配置多少个 `worker` 子进程。假设现在没有用户连入服务器，某一时刻恰好所有的子进程都休眠且等待新连接的系统调用（如 epoll_wait ），这时有一个用户向服务器发起了连接，内核在收到 TCP 的 SYN 包时，会激活所有的休眠 worker 子进程。最终只有最先开始执行 `accept` 的子进程可以成功建立新连接，而其他 `worker` 子进程都将 `accept` 失败。这些 `accept` 失败的子进程被内核唤醒是不必要的，他们被唤醒会的执行很可能是多余的，那么这一时刻他们占用了本不需要占用的资源，引发了不必要的进程切换，增加了系统开销。

### Nginx 怎么解决惊群

前面提到内核解决 `epoll` 的惊群效应是比较晚的，因此 `Nginx` 自身解决了该问题（更准确的说是避免了）。其具体思路是：不让多个进程在同一时间监听接受连接的 `socket`，而是让每个进程轮流监听，这样当有连接过来的时候，就只有一个进程在监听那肯定就没有惊群的问题。具体做法是：利用一把进程间锁，每个进程中都尝试获得这把锁，如果获取成功将监听 `socket` 加入 `wait` 集合中，并设置超时等待连接到来，没有获得所的进程则将监听socket从wait集合去除。这里只是简单讨论 `Nginx` 在处理惊群问题基本做法，实际其代码还处理了很多细节问题，例如简单的连接的负载均衡、定时事件处理等等。
核心的代码如下:

```c
void ngx_process_events_and_timers(ngx_cycle_t *cycle)
{
    ...
    //这里面会对监听socket处理
    //1、获得锁则加入wait集合
    //2、没有获得则去除
    if (ngx_trylock_accept_mutex(cycle) == NGX_ERROR) {
        return;
    }

    ...

    //设置网络读写事件延迟处理标志，即在释放锁后处理
    if (ngx_accept_mutex_held) {
        flags |= NGX_POST_EVENTS;
    } 

    ...

    //这里面epollwait等待网络事件
    //网络连接事件，放入ngx_posted_accept_events队列
    //网络读写事件，放入ngx_posted_events队列
    (void) ngx_process_events(cycle, timer, flags);

    ...

    //先处理网络连接事件，只有获取到锁，这里才会有连接事件
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);

    //释放锁，让其他进程也能够拿到
    if (ngx_accept_mutex_held) {
        ngx_shmtx_unlock(&ngx_accept_mutex);
    }

    //处理网络读写事件
    ngx_event_process_posted(cycle, &ngx_posted_events);
}
```

