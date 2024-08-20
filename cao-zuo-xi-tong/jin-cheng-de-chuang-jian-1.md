# Page 1

fork 是一个系统调用，根据咱们讲过的系统调用的流程，流程的最后会在 sys\_call\_table 中找到相应的系统调用 sys\_fork。

sys\_fork 是如何定义的呢？根据 SYSCALL\_DEFINE0 这个宏的定义，下面这段代码就定义了 sys\_fork。

```
SYSCALL_DEFINE0(fork)
{
......
  return _do_fork(SIGCHLD, 0, 0, NULL, NULL, 0);
}
```

sys\_fork 会调用 \_do\_fork。

```
long _do_fork(unsigned long clone_flags,
        unsigned long stack_start,
        unsigned long stack_size,
        int __user *parent_tidptr,
        int __user *child_tidptr,
        unsigned long tls)
{
  struct task_struct *p;
  int trace = 0;
  long nr;


......
  p = copy_process(clone_flags, stack_start, stack_size,
       child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
......
  if (!IS_ERR(p)) {
    struct pid *pid;
    pid = get_task_pid(p, PIDTYPE_PID);
    nr = pid_vnr(pid);


    if (clone_flags & CLONE_PARENT_SETTID)
      put_user(nr, parent_tidptr);


......
    wake_up_new_task(p);
......
    put_pid(pid);
  } 
......
```

\_do\_fork 里面做的第一件大事就是 copy\_process，咱们前面讲过这个思想。如果所有数据结构都从头创建一份太麻烦了，还不如使用惯用“伎俩”，Ctrl C + Ctrl V。

这里我们再把 task\_struct 的结构图拿出来，对比着看如何一个个复制。

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

```
static __latent_entropy struct task_struct *copy_process(
          unsigned long clone_flags,
          unsigned long stack_start,
          unsigned long stack_size,
          int __user *child_tidptr,
          struct pid *pid,
          int trace,
          unsigned long tls,
          int node)
{
  int retval;
  struct task_struct *p;
......
  p = dup_task_struct(current, node);
```

dup\_task\_struct 主要做了下面几件事情：

调用 alloc\_task\_struct\_node 分配一个 task\_struct 结构；

调用 alloc\_thread\_stack\_node 来创建内核栈，这里面调用 \_\_vmalloc\_node\_range 分配一个连续的 THREAD\_SIZE 的内存空间，赋值给 task\_struct 的 void \*stack 成员变量；

调用 arch\_dup\_task\_struct(struct task\_struct \*dst, struct task\_struct \*src)，将 task\_struct 进行复制，其实就是调用 memcpy；

调用 setup\_thread\_stack 设置 thread\_info。

到这里，整个 task\_struct 复制了一份，而且内核栈也创建好了。

我们再接着看 copy\_process。

```
retval = copy_creds(p, clone_flags);
```

轮到权限相关了，copy\_creds 主要做了下面几件事情：

调用 prepare\_creds，准备一个新的 struct cred \*new。如何准备呢？其实还是从内存中分配一个新的 struct cred 结构，然后调用 memcpy 复制一份父进程的 cred；

接着 p->cred = p->real\_cred = get\_cred(new)，将新进程的“我能操作谁”和“谁能操作我”两个权限都指向新的 cred。

接下来，copy\_process 重新设置进程运行的统计量。

```
p->utime = p->stime = p->gtime = 0;
p->start_time = ktime_get_ns();
p->real_start_time = ktime_get_boot_ns();
```

接下来，copy\_process 开始设置调度相关的变量。

```
retval = sched_fork(clone_flags, p);
```

sched\_fork 主要做了下面几件事情：

调用 \_\_sched\_fork，在这里面将 on\_rq 设为 0，初始化 sched\_entity，将里面的 exec\_start、sum\_exec\_runtime、prev\_sum\_exec\_runtime、vruntime 都设为 0。你还记得吗，这几个变量涉及进程的实际运行时间和虚拟运行时间。是否到时间应该被调度了，就靠它们几个；

设置进程的状态 p->state = TASK\_NEW；

初始化优先级 prio、normal\_prio、static\_prio；

设置调度类，如果是普通进程，就设置为 p->sched\_class = \&fair\_sched\_class；

调用调度类的 task\_fork 函数，对于 CFS 来讲，就是调用 task\_fork\_fair。在这个函数里，先调用 update\_curr，对于当前的进程进行统计量更新，然后把子进程和父进程的 vruntime 设成一样，最后调用 place\_entity，初始化 sched\_entity。这里有一个变量 sysctl\_sched\_child\_runs\_first，可以设置父进程和子进程谁先运行。如果设置了子进程先运行，即便两个进程的 vruntime 一样，也要把子进程的 sched\_entity 放在前面，然后调用 resched\_curr，标记当前运行的进程 TIF\_NEED\_RESCHED，也就是说，把父进程设置为应该被调度，这样下次调度的时候，父进程会被子进程抢占。

接下来，copy\_process 开始初始化与文件和文件系统相关的变量。

```
retval = copy_files(clone_flags, p);
retval = copy_fs(clone_flags, p);
```

copy\_files 主要用于复制一个进程打开的文件信息。这些信息用一个结构 files\_struct 来维护，每个打开的文件都有一个文件描述符。在 copy\_files 函数里面调用 dup\_fd，在这里面会创建一个新的 files\_struct，然后将所有的文件描述符数组 fdtable 拷贝一份。

copy\_fs 主要用于复制一个进程的目录信息。这些信息用一个结构 fs\_struct 来维护。一个进程有自己的根目录和根文件系统 root，也有当前目录 pwd 和当前目录的文件系统，都在 fs\_struct 里面维护。copy\_fs 函数里面调用 copy\_fs\_struct，创建一个新的 fs\_struct，并复制原来进程的 fs\_struct。

接下来，copy\_process 开始初始化与信号相关的变量。

```
init_sigpending(&p->pending);
retval = copy_sighand(clone_flags, p);
retval = copy_signal(clone_flags, p);
```

copy\_sighand 会分配一个新的 sighand\_struct。这里最主要的是维护信号处理函数，在 copy\_sighand 里面会调用 memcpy，将信号处理函数 sighand->action 从父进程复制到子进程。

init\_sigpending 和 copy\_signal 用于初始化，并且复制用于维护发给这个进程的信号的数据结构。copy\_signal 函数会分配一个新的 signal\_struct，并进行初始化。

接下来，copy\_process 开始复制进程内存空间。

```
retval = copy_mm(clone_flags, p);
```

进程都有自己的内存空间，用 mm\_struct 结构来表示。copy\_mm 函数中调用 dup\_mm，分配一个新的 mm\_struct 结构，调用 memcpy 复制这个结构。dup\_mmap 用于复制内存空间中内存映射的部分。前面讲系统调用的时候，我们说过，mmap 可以分配大块的内存，其实 mmap 也可以将一个文件映射到内存中，方便可以像读写内存一样读写文件，这个在内存管理那节我们讲。

接下来，copy\_process 开始分配 pid，设置 tid，group\_leader，并且建立进程之间的亲缘关系。

```
  INIT_LIST_HEAD(&p->children);
  INIT_LIST_HEAD(&p->sibling);
......
    p->pid = pid_nr(pid);
  if (clone_flags & CLONE_THREAD) {
    p->exit_signal = -1;
    p->group_leader = current->group_leader;
    p->tgid = current->tgid;
  } else {
    if (clone_flags & CLONE_PARENT)
      p->exit_signal = current->group_leader->exit_signal;
    else
      p->exit_signal = (clone_flags & CSIGNAL);
    p->group_leader = p;
    p->tgid = p->pid;
  }
......
  if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
    p->real_parent = current->real_parent;
    p->parent_exec_id = current->parent_exec_id;
  } else {
    p->real_parent = current;
    p->parent_exec_id = current->self_exec_id;
  }
```

好了，copy\_process 要结束了，上面图中的组件也初始化的差不多了。

\_do\_fork 做的第二件大事是 wake\_up\_new\_task。新任务刚刚建立，有没有机会抢占别人，获得 CPU 呢？

```
void wake_up_new_task(struct task_struct *p)
{
  struct rq_flags rf;
  struct rq *rq;
......
  p->state = TASK_RUNNING;
......
  activate_task(rq, p, ENQUEUE_NOCLOCK);
  p->on_rq = TASK_ON_RQ_QUEUED;
  trace_sched_wakeup_new(p);
  check_preempt_curr(rq, p, WF_FORK);
......
}
```

首先，我们需要将进程的状态设置为 TASK\_RUNNING。

activate\_task 函数中会调用 enqueue\_task。

```
static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
{
.....
  p->sched_class->enqueue_task(rq, p, flags);
}
```

如果是 CFS 的调度类，则执行相应的 enqueue\_task\_fair。

```
static void
enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
{
  struct cfs_rq *cfs_rq;
  struct sched_entity *se = &p->se;
......
  cfs_rq = cfs_rq_of(se);
  enqueue_entity(cfs_rq, se, flags);
......
  cfs_rq->h_nr_running++;
......
}
```

在 enqueue\_task\_fair 中取出的队列就是 cfs\_rq，然后调用 enqueue\_entity。

在 enqueue\_entity 函数里面，会调用 update\_curr，更新运行的统计量，然后调用 \_\_enqueue\_entity，将 sched\_entity 加入到红黑树里面，然后将 se->on\_rq = 1 设置在队列上。

回到 enqueue\_task\_fair 后，将这个队列上运行的进程数目加一。然后，wake\_up\_new\_task 会调用 check\_preempt\_curr，看是否能够抢占当前进程。

在 check\_preempt\_curr 中，会调用相应的调度类的 rq->curr->sched\_class->check\_preempt\_curr(rq, p, flags)。对于 CFS 调度类来讲，调用的是 check\_preempt\_wakeup。

```
static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
{
  struct task_struct *curr = rq->curr;
  struct sched_entity *se = &curr->se, *pse = &p->se;
  struct cfs_rq *cfs_rq = task_cfs_rq(curr);
......
  if (test_tsk_need_resched(curr))
    return;
......
  find_matching_se(&se, &pse);
  update_curr(cfs_rq_of(se));
  if (wakeup_preempt_entity(se, pse) == 1) {
    goto preempt;
  }
  return;
preempt:
  resched_curr(rq);
......
}
```

在 check\_preempt\_wakeup 函数中，前面调用 task\_fork\_fair 的时候，设置 sysctl\_sched\_child\_runs\_first 了，已经将当前父进程的 TIF\_NEED\_RESCHED 设置了，则直接返回。

否则，check\_preempt\_wakeup 还是会调用 update\_curr 更新一次统计量，然后 wakeup\_preempt\_entity 将父进程和子进程 PK 一次，看是不是要抢占，如果要则调用 resched\_curr 标记父进程为 TIF\_NEED\_RESCHED。

如果新创建的进程应该抢占父进程，在什么时间抢占呢？别忘了 fork 是一个系统调用，从系统调用返回的时候，是抢占的一个好时机，如果父进程判断自己已经被设置为 TIF\_NEED\_RESCHED，就让子进程先跑，抢占自己。

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

这个图的上半部分是复制 task\_struct 结构，你可以对照着右面的 task\_struct 结构图，看这里面的成员是如何一部分一部分地被复制的。图的下半部分是唤醒新创建的子进程，如果条件满足，就会将当前进程设置应该被调度的标识位，就等着当前进程执行 \_\_schedule 了。
