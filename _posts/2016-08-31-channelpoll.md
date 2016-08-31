---
layout: blog
categories: go
title: golang连接池实现
tags: go channel 连接池
excerpt: golang连接池实现
---

开始正文之前我们先来讨论下什么是连接池？顾名思义，连接池是用来存放连接的。这里的连接首先需要满足两个条件：

1. 必须是长连接。一般将client和server间只进行一次交互的连接称为短连接，可以进行多次交互的连接称为长连接。连接池里的连接是需要复用的，因此必须是长连接。此外短连接和长连接的使用需要client和server达成共识，比如client想要使用长连接，而server只支持短连接，即便client不去释放连接，server也会将该连接释放掉。

2. 需要有连接探活机制，不断的踢出不可用连接，并填充可用连接。一个典型的场景是server端重启了，这时client和server间连接就会变得不可用。

接下来，我们就可以构思下怎么去实现连接池了。我们可以用队列（或者链表）去存放可用连接，每次从队首获取可用连接，并将使用后的连接放进队列中，当然在对连接进行操作的时候必须加锁。看起来似乎不错，但还有更优雅的实现方式---channel。因为channel就是用来在进程间通信的，而且channel使用的是race机制，是不需要加锁的。

# 连接池初始化

```go
var (
    cpool *channelPool
)

type channelPool struct {
    conns chan net.Conn
}

func InitChannelPool(host string, maxnum int) {

    cpool = &channelPool{
        conns: make(chan net.Conn, maxnum),
    }

    succnum := 0
    for i := 0; i < maxnum; i++ {
        conn, err := net.Dial("tcp", host)
        if err != nil {
            // put nil in channelPool
            cpool.conns <- nil
        } else {
            succnum++
            cpool.conns <- conn
        }
    }

    if succnum == 0 {
        fmt.Println("host may be wrong")
    }

}
```

这里可自行控制创建成功的力度，nil表示连接池中当前连接不可用。我们在获取到nil连接后，可以接着往里面填充可用连接。

# 获取连接

```go
func GetConn(timeout int, host string) (code int, conn net.Conn) {

    select {
    case conn = <-cpool.conns:
        code = 0
        break
    case <-time.After(time.Duration(timeout) * time.Millisecond):
        code = -1
        return
    }

    if conn == nil {
        var err error
        conn, err = net.DialTimeout("tcp", host, time.Duration(timeout) * time.Millisecond)
        if err != nil {
            code = -1
            cpool.conns <- nil
        }
    }

   return 

}
```

这里采用超时机制是为了确保连接池中连接个数的稳定。还有一种实现方式是如果连接池中可用连接为0，立即重新建立新的连接。

# 释放连接

```go
defer func() {
    cpool.conns <- conn
}()
```

最好放在defer函数中，以免执行失败。

# 探活

探活机制可以通过client和server双方周期性约定一些应答来实现。考虑到连接建立成本并不是太高，一种比较取巧的方法是，当client端接收到server响应的报文为空时，将该连接置为nil即可。
