---
layout:     post                  
title:      go pprof
subtitle:   go pprof分析goroutine阻塞
date:       2020-04-26
author:     Liangjf
header-img: img/post_bg_20200426_4.jpg
catalog: true                      
tags:                       
    - go
---


# go pprof分析goroutine阻塞

## go pprof命令介绍
- pprof web
```http://172.16.7.16:8998/debug/pprof```

- 动态图查看
```http://172.16.7.16:8998/debug/charts```

- 火焰图
```pprof -http 172.16.7.16:7070 http://172.16.7.16:8998/debug/pprof/profile```


## 定位goroutine泄露的2种方法
### 1、pprofcmd命令行
- top：显示正运行到某个函数goroutine的数量
- traces：显示所有goroutine的调用栈
- list：列出代码详细的信息。

### 2、对比pprof文件
- 1. 间隔n分钟执行一下:

```go tool pprof http://localhost:8998/debug/pprof/goroutine```

    fetching profile over HTTP from http://localhost:8998/debug/pprof/goroutine
    Saved profile in /home/liangjf/pprof/pprof.producttestcms.goroutine.002.pb.gz
    File: producttestcms
    Type: goroutine
    Time: Apr 3, 2020 at 1:50pm (CST)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof)

- 2. 对比两个pprof文件:

```go tool pprof -base```

```/home/liangjf/pprof/pprof.producttestcms.goroutine.001.pb.gz```

```/home/liangjf/pprof/pprof.producttestcms.goroutine.002.pb.gz```

    File: producttestcms
    Type: goroutine
    Time: Apr 3, 2020 at 1:48pm (CST)
    Entering interactive mode (type "help" for commands, "o" for options)
    (pprof) top
    Showing nodes accounting for 1, 33.33% of 3 total
    Showing top 10 nodes out of 15
          flat  flat%   sum%        cum   cum%
             1 33.33% 33.33%          1 33.33%  syscall.Syscall
             0     0% 33.33%          1 33.33%  bufio.(*Reader).ReadLine
             0     0% 33.33%          1 33.33%  bufio.(*Reader).ReadSlice
             0     0% 33.33%          1 33.33%  bufio.(*Reader).fill
             0     0% 33.33%          1 33.33%  internal/poll.(*FD).Read
             0     0% 33.33%          1 33.33%  net.(*conn).Read
             0     0% 33.33%          1 33.33%  net.(*netFD).Read
             0     0% 33.33%          1 33.33%  net/http.(*conn).readRequest
             0     0% 33.33%          1 33.33%  net/http.(*conn).serve
             0     0% 33.33%          1 33.33%  net/http.(*connReader).Read

新建了一个goroutine, 因为我http请求了一次, 正常


Web可视化:
- 查看某条调用路径上，当前阻塞在此goroutine的数量
- 查看所有goroutine的运行栈（调用路径），可以显示阻塞在此的时间

        http://ip:port/debug/pprof/goroutine?debug=1
        http://ip:port/debug/pprof/goroutine?debug=2


> http://ip:port/debug/pprof/goroutine?debug=1
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_bg_20200426_21?raw=true)

> http://ip:port/debug/pprof/goroutine?debug=2
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_bg_20200426_22?raw=true)

![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_bg_20200426_23?raw=true)

看看beego框架的源码, 进入对应的源码位置
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_bg_20200426_24?raw=true)

发现sql也有goroutine一直阻塞

```go 1.13 /src/database/sql/sql.go:711```

    func OpenDB(c driver.Connector) *DB {
        ctx, cancel := context.WithCancel(context.Background())
        db := &DB{
            connector:    c,
            openerCh:     make(chan struct{}, connectionRequestQueueSize),
            resetterCh:   make(chan *driverConn, 50),
            lastPut:      make(map[*driverConn]string),
            connRequests: make(map[uint64]chan connRequest),
            stop:         cancel,
        }

        go db.connectionOpener(ctx)
        go db.connectionResetter(ctx)

        return db
    }
    
通过源码追踪，是beego orm框架打开数据库connectInit-->Open-->OpenDB-->go 
db.connectionOpener(ctx)-->case <-ctx.Done()是orm内部open数据库, 然后初始化连接池, 一直阻塞直至调用Stop() 通知ctx done 释放sql goroutine

