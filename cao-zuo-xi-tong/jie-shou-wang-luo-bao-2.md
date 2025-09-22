# 接收网络包2

上一节，我们解析了网络包接收的上半部分，从硬件网卡到 IP 层。这一节，我们接着来解析 TCP 层和 Socket 层都做了哪些事情。

### 网络协议栈的 TCP 层

从 tcp\_v4\_rcv 函数开始，我们的处理逻辑就从 IP 层到了 TCP 层。

\


```c
int tcp_v4_rcv(struct sk_buff *skb)
{
  struct net *net = dev_net(skb->dev);
  const struct iphdr *iph;
  const struct tcphdr *th;
  bool refcounted;
  struct sock *sk;
  int ret;
......
  th = (const struct tcphdr *)skb->data;
  iph = ip_hdr(skb);
......
  TCP_SKB_CB(skb)->seq = ntohl(th->seq);
  TCP_SKB_CB(skb)->end_seq = (TCP_SKB_CB(skb)->seq + th->syn + th->fin + skb->len - th->doff * 4);
  TCP_SKB_CB(skb)->ack_seq = ntohl(th->ack_seq);
  TCP_SKB_CB(skb)->tcp_flags = tcp_flag_byte(th);
  TCP_SKB_CB(skb)->tcp_tw_isn = 0;
  TCP_SKB_CB(skb)->ip_dsfield = ipv4_get_dsfield(iph);
  TCP_SKB_CB(skb)->sacked   = 0;

lookup:
  sk = __inet_lookup_skb(&tcp_hashinfo, skb, __tcp_hdrlen(th), th->source, th->dest, &refcounted);

process:
  if (sk->sk_state == TCP_TIME_WAIT)
    goto do_time_wait;

  if (sk->sk_state == TCP_NEW_SYN_RECV) {
......
  }
......
  th = (const struct tcphdr *)skb->data;
  iph = ip_hdr(skb);

  skb->dev = NULL;

  if (sk->sk_state == TCP_LISTEN) {
    ret = tcp_v4_do_rcv(sk, skb);
    goto put_and_return;
  }
......
  if (!sock_owned_by_user(sk)) {
    if (!tcp_prequeue(sk, skb))
      ret = tcp_v4_do_rcv(sk, skb);
  } else if (tcp_add_backlog(sk, skb)) {
    goto discard_and_relse;
  }
......
}

```

\


在 tcp\_v4\_rcv 中，得到 TCP 的头之后，我们可以开始处理 TCP 层的事情。因为 TCP 层是分状态的，状态被维护在数据结构 struct sock 里面，因而我们要根据 IP 地址以及 TCP 头里面的内容，在 tcp\_hashinfo 中找到这个包对应的 struct sock，从而得到这个包对应的连接的状态。

\


接下来，我们就根据不同的状态做不同的处理，例如，上面代码中的 TCP\_LISTEN、TCP\_NEW\_SYN\_RECV 状态属于连接建立过程中。这个我们在讲三次握手的时候讲过了。再如，TCP\_TIME\_WAIT 状态是连接结束的时候的状态，这个我们暂时可以不用看。

\


接下来，我们来分析最主流的网络包的接收过程，这里面涉及三个队列：

\


* backlog 队列
* prequeue 队列
* sk\_receive\_queue 队列

\


为什么接收网络包的过程，需要在这三个队列里面倒腾过来、倒腾过去呢？这是因为，同样一个网络包要在三个主体之间交接。

\


第一个主体是**软中断的处理过程**。如果你没忘记的话，我们在执行 tcp\_v4\_rcv 函数的时候，依然处于软中断的处理逻辑里，所以必然会占用这个软中断。

\


第二个主体就是**用户态进程**。如果用户态触发系统调用 read 读取网络包，也要从队列里面找。

\


第三个主体就是**内核协议栈**。哪怕用户进程没有调用 read，读取网络包，当网络包来的时候，也得有一个地方收着呀。

\


这时候，我们就能够了解上面代码中 sock\_owned\_by\_user 的意思了，其实就是说，当前这个 sock 是不是正有一个用户态进程等着读数据呢，如果没有，内核协议栈也调用 tcp\_add\_backlog，暂存在 backlog 队列中，并且抓紧离开软中断的处理过程。

\


如果有一个用户态进程等待读取数据呢？我们先调用 tcp\_prequeue，也即赶紧放入 prequeue 队列，并且离开软中断的处理过程。在这个函数里面，我们会看到对于 sysctl\_tcp\_low\_latency 的判断，也即是不是要低时延地处理网络包。

\


如果把 sysctl\_tcp\_low\_latency 设置为 0，那就要放在 prequeue 队列中暂存，这样不用等待网络包处理完毕，就可以离开软中断的处理过程，但是会造成比较长的时延。如果把 sysctl\_tcp\_low\_latency 设置为 1，我们还是调用 tcp\_v4\_do\_rcv。

\


```c
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
  struct sock *rsk;

  if (sk->sk_state == TCP_ESTABLISHED) { /* Fast path */
    struct dst_entry *dst = sk->sk_rx_dst;
......
    tcp_rcv_established(sk, skb, tcp_hdr(skb), skb->len);
    return 0;
  }
......
  if (tcp_rcv_state_process(sk, skb)) {
......
  }
  return 0;
......
}

```

\


在 tcp\_v4\_do\_rcv 中，分两种情况，一种情况是连接已经建立，处于 TCP\_ESTABLISHED 状态，调用 tcp\_rcv\_established。另一种情况，就是其他的状态，调用 tcp\_rcv\_state\_process。

\


```c
int tcp_rcv_state_process(struct sock *sk, struct sk_buff *skb)
{
  struct tcp_sock *tp = tcp_sk(sk);
  struct inet_connection_sock *icsk = inet_csk(sk);
  const struct tcphdr *th = tcp_hdr(skb);
  struct request_sock *req;
  int queued = 0;
  bool acceptable;

  switch (sk->sk_state) {
  case TCP_CLOSE:
......
  case TCP_LISTEN:
......
  case TCP_SYN_SENT:
......
  }
......
  switch (sk->sk_state) {
  case TCP_SYN_RECV:
......
  case TCP_FIN_WAIT1: 
......
  case TCP_CLOSING:
......
  case TCP_LAST_ACK:
......
    }

  /* step 7: process the segment text */
  switch (sk->sk_state) {
  case TCP_CLOSE_WAIT:
  case TCP_CLOSING:
  case TCP_LAST_ACK:
......
  case TCP_FIN_WAIT1:
  case TCP_FIN_WAIT2:
......
  case TCP_ESTABLISHED:
......
  }
}

```

\


在 tcp\_rcv\_state\_process 中，如果我们对着 TCP 的状态图进行比对，能看到，对于 TCP 所有状态的处理，其中和连接建立相关的状态，咱们已经分析过，所以我们重点关注连接状态下的工作模式。

\
![](<../.gitbook/assets/image (54).png>)

\


在连接状态下，我们会调用 tcp\_rcv\_established。在这个函数里面，我们会调用 tcp\_data\_queue，将其放入 sk\_receive\_queue 队列进行处理。

\


```c
static void tcp_data_queue(struct sock *sk, struct sk_buff *skb)
{
  struct tcp_sock *tp = tcp_sk(sk);
  bool fragstolen = false;
......
  if (TCP_SKB_CB(skb)->seq == tp->rcv_nxt) {
    if (tcp_receive_window(tp) == 0)
      goto out_of_window;

    /* Ok. In sequence. In window. */
    if (tp->ucopy.task == current &&
        tp->copied_seq == tp->rcv_nxt && tp->ucopy.len &&
        sock_owned_by_user(sk) && !tp->urg_data) {
      int chunk = min_t(unsigned int, skb->len,
            tp->ucopy.len);

      __set_current_state(TASK_RUNNING);

      if (!skb_copy_datagram_msg(skb, 0, tp->ucopy.msg, chunk)) {
        tp->ucopy.len -= chunk;
        tp->copied_seq += chunk;
        eaten = (chunk == skb->len);
        tcp_rcv_space_adjust(sk);
      }
    }

    if (eaten <= 0) {
queue_and_out:
......
      eaten = tcp_queue_rcv(sk, skb, 0, &fragstolen);
    }
    tcp_rcv_nxt_update(tp, TCP_SKB_CB(skb)->end_seq);
......
    if (!RB_EMPTY_ROOT(&tp->out_of_order_queue)) {
      tcp_ofo_queue(sk);
......
    }
......
    return;
  }

  if (!after(TCP_SKB_CB(skb)->end_seq, tp->rcv_nxt)) {
    /* A retransmit, 2nd most common case.  Force an immediate ack. */
    tcp_dsack_set(sk, TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq);

out_of_window:
    tcp_enter_quickack_mode(sk);
    inet_csk_schedule_ack(sk);
drop:
    tcp_drop(sk, skb);
    return;
  }

  /* Out of window. F.e. zero window probe. */
  if (!before(TCP_SKB_CB(skb)->seq, tp->rcv_nxt + tcp_receive_window(tp)))
    goto out_of_window;

  tcp_enter_quickack_mode(sk);

  if (before(TCP_SKB_CB(skb)->seq, tp->rcv_nxt)) {
    /* Partial packet, seq < rcv_next < end_seq */
    tcp_dsack_set(sk, TCP_SKB_CB(skb)->seq, tp->rcv_nxt);
    /* If window is closed, drop tail of packet. But after
     * remembering D-SACK for its head made in previous line.
     */
    if (!tcp_receive_window(tp))
      goto out_of_window;
    goto queue_and_out;
  }

  tcp_data_queue_ofo(sk, skb);
}

```

\


在 tcp\_data\_queue 中，对于收到的网络包，我们要分情况进行处理。

\


第一种情况，seq == tp->rcv\_nxt，说明来的网络包正是我服务端期望的下一个网络包。这个时候我们判断 sock\_owned\_by\_user，也即用户进程也是正在等待读取，这种情况下，就直接 skb\_copy\_datagram\_msg，将网络包拷贝给用户进程就可以了。

\


如果用户进程没有正在等待读取，或者因为内存原因没有能够拷贝成功，tcp\_queue\_rcv 里面还是将网络包放入 sk\_receive\_queue 队列。

\


接下来，tcp\_rcv\_nxt\_update 将 tp->rcv\_nxt 设置为 end\_seq，也即当前的网络包接收成功后，更新下一个期待的网络包。

\


这个时候，我们还会判断一下另一个队列，out\_of\_order\_queue，也看看乱序队列的情况，看看乱序队列里面的包，会不会因为这个新的网络包的到来，也能放入到 sk\_receive\_queue 队列中。

\


例如，客户端发送的网络包序号为 5、6、7、8、9。在 5 还没有到达的时候，服务端的 rcv\_nxt 应该是 5，也即期望下一个网络包是 5。但是由于中间网络通路的问题，5、6 还没到达服务端，7、8 已经到达了服务端了，这就出现了乱序。

\


乱序的包不能进入 sk\_receive\_queue 队列。因为一旦进入到这个队列，意味着可以发送给用户进程。然而，按照 TCP 的定义，用户进程应该是按顺序收到包的，没有排好序，就不能给用户进程。所以，7、8 不能进入 sk\_receive\_queue 队列，只能暂时放在 out\_of\_order\_queue 乱序队列中。

\


当 5、6 到达的时候，5、6 先进入 sk\_receive\_queue 队列。这个时候我们再来看 out\_of\_order\_queue 乱序队列中的 7、8，发现能够接上。于是，7、8 也能进入 sk\_receive\_queue 队列了。tcp\_ofo\_queue 函数就是做这个事情的。

\


至此第一种情况处理完毕。

\


第二种情况，end\_seq 不大于 rcv\_nxt，也即服务端期望网络包 5。但是，来了一个网络包 3，怎样才会出现这种情况呢？肯定是服务端早就收到了网络包 3，但是 ACK 没有到达客户端，中途丢了，那客户端就认为网络包 3 没有发送成功，于是又发送了一遍，这种情况下，要赶紧给客户端再发送一次 ACK，表示早就收到了。

\


第三种情况，seq 不小于 rcv\_nxt + tcp\_receive\_window。这说明客户端发送得太猛了。本来 seq 肯定应该在接收窗口里面的，这样服务端才来得及处理，结果现在超出了接收窗口，说明客户端一下子把服务端给塞满了。

\


这种情况下，服务端不能再接收数据包了，只能发送 ACK 了，在 ACK 中会将接收窗口为 0 的情况告知客户端，客户端就知道不能再发送了。这个时候双方只能交互窗口探测数据包，直到服务端因为用户进程把数据读走了，空出接收窗口，才能在 ACK 里面再次告诉客户端，又有窗口了，又能发送数据包了。

\


第四种情况，seq 小于 rcv\_nxt，但是 end\_seq 大于 rcv\_nxt，这说明从 seq 到 rcv\_nxt 这部分网络包原来的 ACK 客户端没有收到，所以重新发送了一次，从 rcv\_nxt 到 end\_seq 时新发送的，可以放入 sk\_receive\_queue 队列。

\


当前四种情况都排除掉了，说明网络包一定是一个乱序包了。这里有点儿难理解，我们还是用上面那个乱序的例子仔细分析一下 rcv\_nxt=5。

\


我们假设 tcp\_receive\_window 也是 5，也即超过 10 服务端就接收不了了。当前来的这个网络包既不在 rcv\_nxt 之前（不是 3 这种），也不在 rcv\_nxt + tcp\_receive\_window 之后（不是 11 这种），说明这正在我们期望的接收窗口里面，但是又不是 rcv\_nxt（不是我们马上期望的网络包 5），这正是上面的例子中网络包 7、8 的情况。

\


对于网络包 7、8，我们只好调用 tcp\_data\_queue\_ofo 进入 out\_of\_order\_queue 乱序队列，但是没有关系，当网络包 5、6 到来的时候，我们会走第一种情况，把 7、8 拿出来放到 sk\_receive\_queue 队列中。

\


至此，网络协议栈的处理过程就结束了。

### Socket 层

当接收的网络包进入各种队列之后，接下来我们就要等待用户进程去读取它们了。

\


读取一个 socket，就像读取一个文件一样，读取 socket 的文件描述符，通过 read 系统调用。

\


read 系统调用对于一个文件描述符的操作，大致过程都是类似的，在文件系统那一节，我们已经详细解析过。最终它会调用到用来表示一个打开文件的结构 stuct file 指向的 file\_operations 操作。

\


对于 socket 来讲，它的 file\_operations 定义如下：

\


```c
static const struct file_operations socket_file_ops = {
  .owner =  THIS_MODULE,
  .llseek =  no_llseek,
  .read_iter =  sock_read_iter,
  .write_iter =  sock_write_iter,
  .poll =    sock_poll,
  .unlocked_ioctl = sock_ioctl,
  .mmap =    sock_mmap,
  .release =  sock_close,
  .fasync =  sock_fasync,
  .sendpage =  sock_sendpage,
  .splice_write = generic_splice_sendpage,
  .splice_read =  sock_splice_read,
};

```

\


按照文件系统的读取流程，调用的是 sock\_read\_iter。

\


```c
static ssize_t sock_read_iter(struct kiocb *iocb, struct iov_iter *to)
{
  struct file *file = iocb->ki_filp;
  struct socket *sock = file->private_data;
  struct msghdr msg = {.msg_iter = *to,
           .msg_iocb = iocb};
  ssize_t res;

  if (file->f_flags & O_NONBLOCK)
    msg.msg_flags = MSG_DONTWAIT;
......
  res = sock_recvmsg(sock, &msg, msg.msg_flags);
  *to = msg.msg_iter;
  return res;
}

```

\


在 sock\_read\_iter 中，通过 VFS 中的 struct file，将创建好的 socket 结构拿出来，然后调用 sock\_recvmsg，sock\_recvmsg 会调用 sock\_recvmsg\_nosec。

\


```c
static inline int sock_recvmsg_nosec(struct socket *sock, struct msghdr *msg, int flags)
{
  return sock->ops->recvmsg(sock, msg, msg_data_left(msg), flags);
}

```

\


这里调用了 socket 的 ops 的 recvmsg，这个我们遇到好几次了。根据 inet\_stream\_ops 的定义，这里调用的是 inet\_recvmsg。

\


```c
int inet_recvmsg(struct socket *sock, struct msghdr *msg, size_t size,
     int flags)
{
  struct sock *sk = sock->sk;
  int addr_len = 0;
  int err;
......
  err = sk->sk_prot->recvmsg(sk, msg, size, flags & MSG_DONTWAIT,
           flags & ~MSG_DONTWAIT, &addr_len);
......
}

```

\


这里面，从 socket 结构，我们可以得到更底层的 sock 结构，然后调用 sk\_prot 的 recvmsg 方法。这个同样遇到好几次了，根据 tcp\_prot 的定义，调用的是 tcp\_recvmsg。

\


```c
int tcp_recvmsg(struct sock *sk, struct msghdr *msg, size_t len, int nonblock,
    int flags, int *addr_len)
{
  struct tcp_sock *tp = tcp_sk(sk);
  int copied = 0;
  u32 peek_seq;
  u32 *seq;
  unsigned long used;
  int err;
  int target;    /* Read at least this many bytes */
  long timeo;
  struct task_struct *user_recv = NULL;
  struct sk_buff *skb, *last;
.....
  do {
    u32 offset;
......
    /* Next get a buffer. */
    last = skb_peek_tail(&sk->sk_receive_queue);
    skb_queue_walk(&sk->sk_receive_queue, skb) {
      last = skb;
      offset = *seq - TCP_SKB_CB(skb)->seq;
      if (offset < skb->len)
        goto found_ok_skb;
......
    }
......
    if (!sysctl_tcp_low_latency && tp->ucopy.task == user_recv) {
      /* Install new reader */
      if (!user_recv && !(flags & (MSG_TRUNC | MSG_PEEK))) {
        user_recv = current;
        tp->ucopy.task = user_recv;
        tp->ucopy.msg = msg;
      }

      tp->ucopy.len = len;
      /* Look: we have the following (pseudo)queues:
       *
       * 1. packets in flight
       * 2. backlog
       * 3. prequeue
       * 4. receive_queue
       *
       * Each queue can be processed only if the next ones
       * are empty. 
       */
      if (!skb_queue_empty(&tp->ucopy.prequeue))
        goto do_prequeue;
    }

    if (copied >= target) {
      /* Do not sleep, just process backlog. */
      release_sock(sk);
      lock_sock(sk);
    } else {
      sk_wait_data(sk, &timeo, last);
    }

    if (user_recv) {
      int chunk;
      chunk = len - tp->ucopy.len;
      if (chunk != 0) {
        len -= chunk;
        copied += chunk;
      }

      if (tp->rcv_nxt == tp->copied_seq &&
          !skb_queue_empty(&tp->ucopy.prequeue)) {
do_prequeue:
        tcp_prequeue_process(sk);

        chunk = len - tp->ucopy.len;
        if (chunk != 0) {
          len -= chunk;
          copied += chunk;
        }
      }
    }
    continue;
  found_ok_skb:
    /* Ok so how much can we use? */
    used = skb->len - offset;
    if (len < used)
      used = len;

    if (!(flags & MSG_TRUNC)) {
      err = skb_copy_datagram_msg(skb, offset, msg, used);
......
    }

    *seq += used;
    copied += used;
    len -= used;

    tcp_rcv_space_adjust(sk);
......
  } while (len > 0);
......
}

```

\


tcp\_recvmsg 这个函数比较长，里面逻辑也很复杂，好在里面有一段注释概括了这里面的逻辑。注释里面提到了三个队列，receive\_queue 队列、prequeue 队列和 backlog 队列。这里面，我们需要把前一个队列处理完毕，才处理后一个队列。

\


tcp\_recvmsg 的整个逻辑也是这样执行的：这里面有一个 while 循环，不断地读取网络包。

\


这里，我们会先处理 sk\_receive\_queue 队列。如果找到了网络包，就跳到 found\_ok\_skb 这里。这里会调用 skb\_copy\_datagram\_msg，将网络包拷贝到用户进程中，然后直接进入下一层循环。

\


直到 sk\_receive\_queue 队列处理完毕，我们才到了 sysctl\_tcp\_low\_latency 判断。如果不需要低时延，则会有 prequeue 队列。于是，我们能就跳到 do\_prequeue 这里，调用 tcp\_prequeue\_process 进行处理。

\


如果 sysctl\_tcp\_low\_latency 设置为 1，也即没有 prequeue 队列，或者 prequeue 队列为空，则需要处理 backlog 队列，在 release\_sock 函数中处理。

\


release\_sock 会调用 \_\_release\_sock，这里面会依次处理队列中的网络包。

\


```c
void release_sock(struct sock *sk)
{
......
  if (sk->sk_backlog.tail)
    __release_sock(sk);
......
}

static void __release_sock(struct sock *sk)
  __releases(&sk->sk_lock.slock)
  __acquires(&sk->sk_lock.slock)
{
  struct sk_buff *skb, *next;

  while ((skb = sk->sk_backlog.head) != NULL) {
    sk->sk_backlog.head = sk->sk_backlog.tail = NULL;
    do {
      next = skb->next;
      prefetch(next);
      skb->next = NULL;
      sk_backlog_rcv(sk, skb);
      cond_resched();
      skb = next;
    } while (skb != NULL);
  }
......
}

```

\


最后，哪里都没有网络包，我们只好调用 sk\_wait\_data，继续等待在哪里，等待网络包的到来。

\


至此，网络包的接收过程到此结束。

### 总结时刻

这一节我们讲完了接收网络包，我们来从头串一下，整个过程可以分成以下几个层次。

\


* 硬件网卡接收到网络包之后，通过 DMA 技术，将网络包放入 Ring Buffer；
* 硬件网卡通过中断通知 CPU 新的网络包的到来；
* 网卡驱动程序会注册中断处理函数 ixgb\_intr；
* 中断处理函数处理完需要暂时屏蔽中断的核心流程之后，通过软中断 NET\_RX\_SOFTIRQ 触发接下来的处理过程；
* NET\_RX\_SOFTIRQ 软中断处理函数 net\_rx\_action，net\_rx\_action 会调用 napi\_poll，进而调用 ixgb\_clean\_rx\_irq，从 Ring Buffer 中读取数据到内核 struct sk\_buff；
* 调用 netif\_receive\_skb 进入内核网络协议栈，进行一些关于 VLAN 的二层逻辑处理后，调用 ip\_rcv 进入三层 IP 层；
* 在 IP 层，会处理 iptables 规则，然后调用 ip\_local\_deliver 交给更上层 TCP 层；
* 在 TCP 层调用 tcp\_v4\_rcv，这里面有三个队列需要处理，如果当前的 Socket 不是正在被读；取，则放入 backlog 队列，如果正在被读取，不需要很实时的话，则放入 prequeue 队列，其他情况调用 tcp\_v4\_do\_rcv；
* 在 tcp\_v4\_do\_rcv 中，如果是处于 TCP\_ESTABLISHED 状态，调用 tcp\_rcv\_established，其他的状态，调用 tcp\_rcv\_state\_process；
* 在 tcp\_rcv\_established 中，调用 tcp\_data\_queue，如果序列号能够接的上，则放入 sk\_receive\_queue 队列；如果序列号接不上，则暂时放入 out\_of\_order\_queue 队列，等序列号能够接上的时候，再放入 sk\_receive\_queue 队列。

\


至此内核接收网络包的过程到此结束，接下来就是用户态读取网络包的过程，这个过程分成几个层次。

\


* VFS 层：read 系统调用找到 struct file，根据里面的 file\_operations 的定义，调用 sock\_read\_iter 函数。sock\_read\_iter 函数调用 sock\_recvmsg 函数。
* Socket 层：从 struct file 里面的 private\_data 得到 struct socket，根据里面 ops 的定义，调用 inet\_recvmsg 函数。
* Sock 层：从 struct socket 里面的 sk 得到 struct sock，根据里面 sk\_prot 的定义，调用 tcp\_recvmsg 函数。
* TCP 层：tcp\_recvmsg 函数会依次读取 receive\_queue 队列、prequeue 队列和 backlog 队列。

<figure><img src="../.gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>
