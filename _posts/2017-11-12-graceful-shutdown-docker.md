---
layout: post
title: "如何优雅的关闭容器"
---

最近把 Docker 官方的 [Docker Reference](https://github.com/docker/docker-ce/blob/master/components/cli/docs/reference/builder.md) 文档又读了一遍，发现有些细节深究起来，还是有很多可挖的。针对写 `Dockerfile`，大部分时候只要照葫芦画瓢，基本也不会有什么大的问题，但是如果再深入理解一下那就更有意思了。

要说如何优雅的关闭容器，那就不得不提到信号（Signal）的理念，以及 `Dockerfile` 中 `ENTRYPOINT` 和 `CMD` 指令了。在具体说优雅关闭之前，先了解一下信号这个 Linux 中的基础概念。

## 信号

> 信号是事件发生时对进程的通知机制，有时也称之为软件中断。

信号有不同的类型，Linux 对标准信号的编号为 1~31，可以通过 `kill -l` 获取信号名称：

```
# kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      
 4) SIGILL       5) SIGTRAP      6) SIGABRT      
 7) SIGBUS       8) SIGFPE       9) SIGKILL
 10) SIGUSR1    11) SIGSEGV     12) SIGUSR2
 13) SIGPIPE    14) SIGALRM     15) SIGTERM
... ...
```

实际列出的信号超过了 31 个，有些是其它名称的同义词，有些则是定义但未使用的。以下介绍几个常用的信号：

* `1) SIGHUP` 当终端断开（挂机）时，将发送该信号给终端控制进程。`SIGHUP` 信号还可用于守护进程（比如，init 等）。许多守护进程会在收到 `SIGHUP` 信号时重新进行初始化并重读配置文件。
* `2) SIGINT` 当用户键入终端中断字符（通常为 __Control-C__ ） 时，终端驱动程序将发送该信号给前台进程组。该信号的默认行为是终止进程。
* `3) SIGQUIT` 当用户在键盘上键入退出字符（通常为 __Control-\\__ ）时，该信号将发往前台进程组。默认情况下，该信号终止进程，并生成用于调试的核心转储文件。进程如果陷入无限循环，或者不再响应时，使用 `SIGQUIT` 信号就很合适。
* `9) SIGKILL` 此信号为 “必杀（sure kill）” 信号，处理器程序无法将其阻塞、忽略或者捕获，故而 “一击必杀”，总能终止程序。
* `15) SIGTERM` 这是用来终止进程的标准信号，也是 `kill`、`killall`、`pkill` 命令所发送的默认信号。精心设计的应用程序应当为 `SIGTERM` 信号设置处理器程序，以便其能够预先清除临时文件和释放其它资源，从而全身而退。因此，总是应该先尝试使用 `SIGTERM` 信号来终止进程，而把 `SIGKILL` 作为最后手段，去对付那些不响应 `SIGTERM` 信号的失控进程。
* `20) SIGTSTP` 这是作业控制的停止信号，当用户在键盘上输入挂起字符（通常为 __Control-Z__ ）时，将该信号给前台进程组，使其停止运行。

值得注意的是，__Control-D__ 不会发起信号，它表示 EOF(End-Of-File)，关闭标准输入（stdin）管道（比如可以通过 __Control-D__ 退出当前 shell）。如果程序不读取当前输入的话，是不受 __Control-D__ 影响的。

程序可以针对信号捕捉，然后执行相应函数：

![](images/signal.png)

> 以上知识大部分都来自 《Linux/UNIX 系统编程手册》，想要了解更多的，可以查看该书上册的 20、21、22 章节。

## `ENTRYPOINT`、`CMD`

可能有人会问，说了半天，那信号和优雅的关闭容器有半毛钱的关系啊？话说，这和钱确实没关系，但是和如何优雅关闭容器却关系密切。接着说 `Dockerfile` 中的 `ENTRYPOINT` 和 `CMD` 指令，它们的主要功能是指定容器启动时执行的程序。

`CMD` 有三种格式：

* `CMD ["executable","param1","param2"]`（exec 格式, 推荐使用这种格式）
* `CMD ["param1","param2"]` （作为 `ENTRYPOINT` 指令参数）
* `CMD command param1 param2` （shell 格式，默认 `/bin/sh -c`）

`ENTRYPOINT` 有两种格式：

* `ENTRYPOINT ["executable", "param1", "param2"]` （exec 格式，推荐优先使用这种格式）
* `ENTRYPOINT command param1 param2` （shell 格式）

其中，不管你 `Dockerfile` 用其中哪个指令，两个指令都推荐使用 __exec__ 格式，而不是 __shell__ 格式。原因就是因为使用 __shell__ 格式之后，程序会以 `/bin/sh -c` 的子命令启动，并且 __shell__ 格式下不会传递任何信号给程序。这也就导致，在 `docker stop` 容器的时候，以这种格式运行的程序捕捉不到发送的信号，也就谈不上优雅的关闭了。

```
➜  ~ docker stop --help

Usage:  docker stop [OPTIONS] CONTAINER [CONTAINER...]

Stop one or more running containers

Options:
      --help       Print usage
  -t, --time int   Seconds to wait for stop before killing it (default 10)
```

`docker stop` 停掉容器的时候，默认会发送一个 `SIGTERM` 的信号，默认 10s 后容器没有停止的话，就 `SIGKILL` 强制停止容器。通过 `-t` 选项可以设置等待时间。

```
➜  ~ docker kill --help

Usage:  docker kill [OPTIONS] CONTAINER [CONTAINER...]

Kill one or more running containers

Options:
      --help            Print usage
  -s, --signal string   Signal to send to the container (default "KILL")
```

通过 `docker kill` 的 `-s` 选项还可以指定给容器发送的信号。

所以，说了那么多，只要 `Dockerfile` 中通过 __exec__ 格式执行容器启动命令就相安无事了？那当然是，没有那么简单的了，接下来我们通过实例来看看具体的效果是怎么样的。

## 实例

通过 Go 写一个简单的信号处理器：

``` golang
➜  ~ cat signals.go
package main

import (
    "fmt"
    "os"
    "os/signal"
    "syscall"
)

func main() {
    sigs := make(chan os.Signal, 1)
    done := make(chan bool, 1)

    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)

    go func() {
        sig := <-sigs
        fmt.Println()
        fmt.Println(sig)
        done <- true
    }()

    fmt.Println("awaiting signal")
    <-done
    fmt.Println("exiting")
}
```

### 实例 1

```
➜  ~ GOOS=linux GOARCH=amd64 go build signals.go
➜  ~ ls
Dockerfile signals    signals.go
➜  ~ cat Dockerfile
FROM busybox

COPY signals /signals

CMD ["/signals"]    # exec 格式执行
➜  ~ docker build -t signals .
```

通过 `tmux` 开启两个面板，一个运行容器，一个执行 `docker stop`：

```
➜  ~ docker run -it --rm --name signals signals
awaiting signal

terminated
exiting
```

```
➜  ~ time docker stop signals
signals
docker stop signals  0.01s user 0.02s system 4% cpu 0.732 total
➜  ~
```

可以发现，容器停止之前，程序接收到信号并输出相应信息，并且停止总耗时为 0.732 s，达到了优雅的效果。

修改 `Dockerfile` 中 `CMD` 执行格式，执行相同操作：

```
➜  ~ cat Dockerfile
FROM busybox

COPY signals /signals

CMD /signals        # shell 格式执行
➜  ~ docker build -t signals .
```

```
➜  ~ docker run -it --rm --name signals signals
awaiting signal
➜  ~
```

```
➜  ~ time docker stop signals
signals
docker stop signals  0.01s user 0.01s system 0% cpu 10.719 total
```

通过 __shell__ 格式之后，可以发现容器停止之前，程序并未接收到任何信号，并且停止时间为 10.719s，说明该容器是被强制停止的。

结论很明显，为了优雅的退出容器，我们应该采用 __exec__ 这种格式。

### 实例 2

通过实例 1 我们都会在 `Dockerfile` 中都会通过 __exec__ 这种格式来执行程序了，那如果执行的程序本身也是一个 shell 脚本呢？

```
➜  ~ ls
Dockerfile signals    signals.go start.sh
➜  ~ cat Dockerfile
FROM busybox

COPY signals /signals
COPY start.sh /start.sh     # 引入 shell 脚本启动

CMD ["/start.sh"]
➜  ~ cat start.sh
#!/bin/sh

/signals
➜  ~
```

测试依然引用实例 1 中的方法：

```
➜  ~ docker run -it --rm --name signals signals
awaiting signal
➜  ~
```

```
➜  ~ time docker stop signals
signals
docker stop signals  0.01s user 0.02s system 0% cpu 10.765 total
➜  ~
```

可以发现，即使 `Dockerfile` 中的 `CMD` 指令使用的是 __exec__ 格式，容器中的程序依然没有接收到信号，最后被强制关闭。因为 shell 脚本中执行的原因，导致信号依然没有被传递，我们需要针对 shell 脚本做一些改造：

```
➜  ~ cat start.sh
#!/bin/sh

exec /signals   # 加入 exec 执行
➜  ~ docker build -t signals .
```

```
➜  ~ docker run -it --rm --name signals signals
awaiting signal

terminated
exiting
```

```
➜  ~ time docker stop signals
signals
docker stop signals  0.02s user 0.02s system 4% cpu 0.744 total
➜  ~
```

可以看到，加入 exec 命令之后，程序又可以接收到信号正常退出了。当然，如果你 `Dockerfile` 中的 `CMD` 是以 __shell__ 格式运行的，即使启动脚本中加入 exec 也是无效的。再者，如果你的程序本身不能针对信号做一些处理，也就谈不上优雅关闭了。

--EOF--
