---
layout: post
title: Python并发编程中的KeyboardInterrupt
---

## Info
使用脚本程序时我们时常会用到`<Ctrl-c>`触发`KeyboardInterrupt`来进行退出，本文记录了在几种典型并发模型中实现此类退出的方式。

### 基础知识

- signal
	-  信号是一种**进程**间的通讯机制，`<Ctrl-c>`实际上发出的是`SIGINT`信号，用于终止进程的运行。需要注意的是，线程并不是处理信号的单元。

- join
	- `x.join()`：阻塞调用该语句的线程/进程，直到`x`执行结束。阻塞并不意味着不接收信号，只是直到阻塞结束再处理该信号。

### multiprocessing
由于Python的GIL，多进程承担了更多的并发任务。

#### 0x00 缺陷方案

```python
from multiprocessing import Pool


def main():
    p = Pool(3)
    for _ in range(6):
        p.apply_async(handler)
    p.close()
    try:
        p.join()
    except KeyboardInterrupt:
        p.terminate()
        p.join()
        print '[-] main exit'

def handler():
    try:
        while 1:
            pass
    except KeyboardInterrupt:
        print '[-] handler exit'


if __name__ == '__main__':
    main()
```

运行结果如下：

```python
^C[-] handler exit
[-] handler exit
[-] handler exit
^C[-] handler exit
[-] handler exit
[-] handler exit
[-] main exit
```

根据运行结果来看，`<Ctrl-c>`发出的信号首先到达由子进程处理，待子进程处理完后父进程再处理，这是由于`p.join()`阻塞了父进程，使信号不能第一时间被处理。在子进程结束后，`p.join()`返回之前，主进程才处理接收到的信号。

另外一点，实际上此次退出使用了两次`<Ctrl-c>`，因为`Pool(3)`规定了一次装载3个子进程，而我们的任务有6个。所以当第一次使用`<Ctrl-c>`时实际上只是退出了3个子进程，第二次再退出3个后再退出父进程。那么当任务数远大于我们分配的子进程数时，这个方案就是不可行的。

#### 0x01 可行方案

```python
...
def main():
    p = Pool(3)
    r_list = []
    for _ in range(6):
        r_list.append(p.apply_async(handler))
    p.close()
    try:
        for r in r_list:
            r.get()
    except KeyboardInterrupt:
        p.terminate()
        print '[-] main exit'
    p.join()
...
```

执行结果如下：

```python
^C[-] handler exit
[-] handler exit
[-] handler exit
[-] main exit
```

当按下`<Ctrl-c>`时，当前`Pool`中的任务退出，相应的`r.get()`得到返回，使`r.get()`不再阻塞父进程，从而父进程得以处理之前得到的信号，程序退出。

由于`r.get()`对应的是任务，所以只要有一个任务结束父进程就可处理退出信号，所以不存在第一个方案的缺陷。

### threading
多线程在I/O密集型的任务中仍有很重要的地位，其利用如下方式实现`<Ctrl-c>`退出：

```python
for i in range(thread_num):
    thread = threading.Thread(target=thread_handler)
    thread.daemon = True
    thread.start()
while threading.activeCount() > 1:
    try:
        time.sleep(1)
    except KeyboardInterrupt:
        exit('User aborted')
```

`thread.deamon = True`将所有子线程设置成守护线程。根据Python守护线程的规定，当所有非守护线程退出的时候所有守护线程自动退出。所以当主线程（非守护线程）也就是我们的进程处理`SIGINT`信号并退出后，所有子线程也就都退出了。至于`while`循环则是为了保证主线程在正常情况下不退出，以免影响子线程的执行。

### gevent
协程也是一种很常用的并发模型，在Python中使用gevent来实现协程，并与multiprocessing联合来充分利用多核。使用`<Ctrl-c>`退出协程的代码如下：

```python
import gevent
from gevent import monkey; monkey.patch_socket()


def main():
    try:
        jobs = [gevent.spawn(handler) for _ in range(5)]
        gevent.joinall(jobs)
    except KeyboardInterrupt:
        print '[-] main exit'

def handler():
    while 1:
        pass


if __name__ == '__main__':
    main()
```

运行结果如下：

```python
^CTraceback (most recent call last):
  File "/home/.../python2.7/site-packages/gevent/greenlet.py", line 327, in run
    result = self._run(*self.args, **self.kwargs)
  File "gv_kbi.py", line 17, in handler
    pass
KeyboardInterrupt
<Greenlet at 0x7f478de72a50: handler> failed with KeyboardInterrupt

[-] main exit
```

我们并没有在greenlet中处理异常，因为在greenlet中处理异常相当于加快了greenlet的结束，使gevent切换到另一个greenlet，并没有终止程序的运行。所以使greenlet中的异常向上抛出，在主程序中捕获并处理异常使程序退出。

