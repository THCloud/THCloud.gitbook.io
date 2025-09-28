# Go

### GC

时机：

* 堆内存增长到阈值
* deamon协程定时触发
* 手动调用触发

方法：三色标记

* 白色：回收
* 灰色：扫到但是引用对象没扫
* 黑色：扫完，不回收
* 从根对象不停取灰色，扫引用，扫完变黑，并发清除
* 只关注指针引用；栈变量有连续栈机制



### Mem

tcmalloc

* 每个thread有thread cache
* threadcache不足的时候找全局central cache
* 大对象分配走page heap

go

* mcache分配tiny和small对象
* 每个m所在的p有一个mcache
* mcache有若干个mspan
* mcentral枷锁分配

区别：tiny对象，绑定p



### GMP

G：

* 状态有runnable，running，syscall，waiting，dead
* 有自己栈指针和计数器

M：

* 有P，next P，G0
* G0是特殊goroutine，栈为线程栈

P：

* 有Mcache，G，实现工作流窃取

Sched：记录全局这些信息

调度：

* 新建G，挂p，如果p不满，挂到m上
* m循环schedule
* g的新建调用wakeupP

抢占调度时机

* sysmon在gc时候检查g运行耗时
* morestack时候触发schedule
* 新建G时候

系统调用时候

* M解绑p
* m阻塞，G转化状态
* io完成，唤醒G



### 定时m



