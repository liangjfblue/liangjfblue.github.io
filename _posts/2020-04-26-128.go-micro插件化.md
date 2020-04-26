---
layout:     post                  
title:      插件化和设计模式
subtitle:   从go-micro中看插件化和设计模式的思想
date:       2020-04-26
author:     Liangjf
header-img: img/post_bg_20200426_3.jpg
catalog: true                      
tags:                       
    - go
---


# 从go-micro中看插件化和设计模式的思想

go-micro以组件插件化著称，那么是如何实现的？

先抛出结论：go-micro的插件化基本是借助 **函数选项模式**实现默认参数和可选初始化参数；借助接口来实现多态的特性，只要实现对应插件的接口就可以注册；借助map来实现存放支持的插件，并通过导入包import _，调用init函数来初始化插件，或者代码注入的方式。    

## 以**服务注册**插件来进行分析

**服务注册**插件在registry包下，源码目录结构如下

```github.com/micro/go-micro@v1.16.0/registry```

    .
    ├── cache
    │   ├── cache.go
    │   ├── options.go
    │   └── README.md
    ├── encoding.go
    ├── encoding_test.go
    ├── etcd
    │   ├── etcd.go
    │   ├── options.go
    │   └── watcher.go
    ├── mdns
    │   └── mdns.go
    ├── mdns_registry.go
    ├── mdns_test.go
    ├── mdns_watcher.go
    ├── memory
    │   ├── memory.go
    │   ├── memory_test.go
    │   ├── memory_watcher.go
    │   ├── options.go
    │   ├── watcher.go
    │   └── watcher_test.go
    ├── options.go
    ├── registry.go
    ├── service
    │   ├── handler
    │   ├── proto
    │   ├── service.go
    │   ├── util.go
    │   └── watcher.go
    ├── service.go
    ├── util.go
    ├── util_test.go
    ├── watcher.go
    └── watcher_test.go

- cache：用于缓存服务列表信息
- etcd：etcd作为服务发现注册中心
- mdns：go-micro内置默认的服务发现注册中心（局域网，广播方式）
- memory：纯内存方式作为服务发现注册中心（一般生产环境不用）
- service：实现rpc接口，其他模块是通过rpc来调用服务发现模块register的接口  
  - Register(*Service, ...RegisterOption) error
  - Deregister(*Service) error
  - GetService(string) ([]*Service, error)
  - ListServices() ([]*Service, error)
  - Watch(...WatchOption) (Watcher, error)


选择对应的服务发现注册中心很简单，在micro new时会创建plugin.go文件，导入包

    import (
        // registry
        // etcd v3
        _ "github.com/micro/go-plugins/registry/etcd"
    )

这样子就会调用etcd注册到Registries表中

```github.com/micro/go-plugins@v1.5.1/registry/etcdv3/etcd.go```

    func init() {
        cmd.DefaultRegistries["etcdv3"] = etcd.NewRegistry
    }

    func NewRegistry(opts ...registry.Option) registry.Registry {
        return etcd.NewRegistry(opts...)
    }

## go-micro的插件包go-plugins
go-plugins是go-micro的插件包，包含了`agent`，`broker`，`client`，`codec`，`registry`，`transport`，`wrapper`等模块，里面又包含了多种插件，任君选择。

    ➜  go-plugins@v1.5.1 tree -L 1
    .
    ├── agent
    ├── broker
    ├── client
    ├── codec
    ├── config
    ├── go.mod
    ├── go.sum
    ├── LICENSE
    ├── micro
    ├── plugin.go
    ├── proxy
    ├── README.md
    ├── registry
    ├── server
    ├── service
    ├── store
    ├── sync
    ├── template.go
    ├── transport
    ├── web
    └── wrapper

来看服务发现注册相关的，registry包。

    ➜  registry tree -L 1  
    .
    ├── cache
    ├── consul
    ├── etcd
    ├── etcdv3
    ├── eureka
    ├── gossip
    ├── kubernetes
    ├── mdns
    ├── memory
    ├── multi
    ├── nats
    ├── proxy
    ├── service
    └── zookeeper

可以看到，支持了好多种开源组件作为插件来作为服务注册中心。但是go-micro框架只自带了etcd，mdns，memory三种。其他都是社区开发支持的。

### 为什么可以如此方便的插件化，可插拔的替换呢？

实质是这些插件都实现了下面的接口

    type Registry interface {
        Init(...Option) error
        Options() Options
        Register(*Service, ...RegisterOption) error
        Deregister(*Service) error
        GetService(string) ([]*Service, error)
        ListServices() ([]*Service, error)
        Watch(...WatchOption) (Watcher, error)
        String() string
    }

然后通过一个注册表DefaultRegistries来存放所有的插件

```DefaultRegistries```是一个map，key是插件名字，value是对应的插件结构

```map[string]func(...registry.Option) registry.Registry```

看下图，所有的插件都有这样的代码，这里就是通过导入包时，借助init函数的机制来达到注册插件的目的。

    func init() {
        cmd.DefaultRegistries["etcdv3"] = etcd.NewRegistry
    }

    func NewRegistry(opts ...registry.Option) registry.Registry {
        return etcd.NewRegistry(opts...)
    }

![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_bg_DefaultRegistries?raw=true)

以上是插件插拔注册的机制。

插件的真正底层选择替换默认值（mdns）有两种选择：
- 1、插件包导入+命令行选择
- 2、编写代码，参数选项方式注入

#### 第一种（插件包导入+命令行选择）：
**插件包导入**

在micro new时会创建plugin.go文件，导入包

    import (
        // registry
        // etcd v3
        _ "github.com/micro/go-plugins/registry/etcd"
    )

    func init() {
	    cmd.DefaultRegistries["etcdv3"] = etcd.NewRegistry
    }

    func NewRegistry(opts ...registry.Option) registry.Registry {
        return etcd.NewRegistry(opts...)
    }

**命令行选择**

```github.com/micro/go-micro@v1.16.0/config/cmd/cmd.go:254```

从这里可以看到cmd命令结构持有DefaultRegistries，执行命令
```micro --registry=etcd```就可以选择对应的插件

就会触发此处代码：

    if name := ctx.String("registry"); len(name) > 0 && (*c.opts.Registry).String() != name {
        r, ok := c.opts.Registries[name]
        if !ok {
            return fmt.Errorf("Registry %s not found", name)
        }

        *c.opts.Registry = r()
        serverOpts = append(serverOpts, server.Registry(*c.opts.Registry))
        clientOpts = append(clientOpts, client.Registry(*c.opts.Registry))

        if err := (*c.opts.Selector).Init(selector.Registry(*c.opts.Registry)); err != nil {
            log.Fatalf("Error configuring registry: %v", err)
        }

        clientOpts = append(clientOpts, client.Selector(*c.opts.Selector))

        if err := (*c.opts.Broker).Init(broker.Registry(*c.opts.Registry)); err != nil {
            log.Fatalf("Error configuring broker: %v", err)
        }
    }

关键是

    r, ok := c.opts.Registries[name]
    *c.opts.Registry = r()

**更新替换，micro的选项参数结构Options结构的Registry**


#### 第二种（编写代码，参数选项方式注入）：

    reg := etcdv3.NewRegistry(func(op *registry.Options) {
		op.Addrs = []string{
			"http://192.168.0.112:9002",
		}
		op.Timeout = 5 * time.Second
	})

	s.service = micro.NewService(
		micro.Name(s.serviceName),
		micro.Version(s.serviceVersion),
		micro.Registry(reg),
	)

Registry选项函数，更新其他模块的服务发现注册插件为新注册的

    func Registry(r registry.Registry) Option {
        return func(o *Options) {
            o.Registry = r
            o.Client.Init(client.Registry(r))
            o.Server.Init(server.Registry(r))
            o.Client.Options().Selector.Init(selector.Registry(r))
            o.Broker.Init(broker.Registry(r))
        }
    }

其实，最终都是更新Options结构的Registry，因为它是micro的选项参数结构

```github.com/micro/go-micro@v1.16.0/options.go:17```

    type Options struct {
        Broker    broker.Broker
        Cmd       cmd.Cmd
        Client    client.Client
        Server    server.Server
        Registry  registry.Registry
        Transport transport.Transport

        // Before and After funcs
        BeforeStart []func() error
        BeforeStop  []func() error
        AfterStart  []func() error
        AfterStop   []func() error

        Context context.Context
    }

到这里为止，从插件的导入，插件的注册，插件的选择的流程已经结束了。

## 插件化实现的流程

### 插件导入
- 插件包导入+命令行。

    import (
        _ "github.com/micro/go-plugins/registry/etcd"
    )

    micro --registry=etcd --registry_address=192.168.0.112:9002

- 代码注入。```micro.NewService()```

### 插件注册
- 插件包导入+命令行
    
```github.com/micro/go-micro@v1.16.0/config/cmd/cmd.go:263```

命令行```micro --registry=etcd --registry_address=192.168.0.112:9002```会触发以下代码执行

    if name := ctx.String("registry"); len(name) > 0 && (*c.opts.Registry).String() != name {
		r, ok := c.opts.Registries[name]
		if !ok {
			return fmt.Errorf("Registry %s not found", name)
		}

		*c.opts.Registry = r()
		//更新初始化其他模块的Registrie参数
        ...
	}

- 代码注入

    func Registry(r registry.Registry) Option {
        return func(o *Options) {
            o.Registry = r
            // Update Client and Server
            o.Client.Init(client.Registry(r))
            o.Server.Init(server.Registry(r))
            // Update Selector
            o.Client.Options().Selector.Init(selector.Registry(r))
            // Update Broker
            o.Broker.Init(broker.Registry(r))
        }
    }   

### 插件的选择使用
**两种方式都是更新替换Options结构的o.Registry，并更新其他模块的Registry**


不仅在go开发中，在所有开发中，根据设计模式指导，业务不依赖底层实现，而依赖接口，也就是不依赖具体而依赖抽象，这样的好处是业务依赖的组件，只要是实现了接口的组件，都可以随意替换，而不影响上层接口，只是底层的逻辑改变，做到了接口稳定，业务和实现解耦，具有良好的扩展性和维护性。

## 函数选项模式
以cache包为例子，为了简洁，删除了不必要的注释

    //Cache接口
    type Cache interface {
        registry.Registry
        Stop()
    }

这里应用了装饰模式，Cache接口组合了registry.Registry接口，符合go的小接口，大接口的设计原则。只要在NewCache时通过依赖注入Registry对象，那么实现了Cache接口的cache就会拥有Registry接口的功能


    //参数选项
    type Options struct {
        TTL time.Duration
    }

    //实现参数选项的函数
    type Option func(o *Options)

    //cache结构是真正的cache业务具体实现
    type cache struct {
        registry.Registry
        opts Options

        sync.RWMutex
        cache   map[string][]*registry.Service
        ttls    map[string]time.Time
        watched map[string]bool
        exit chan bool
        running bool
        status error
    }

其包含了服务发现的接口和参数选项结构，前者是用于调用服务发现注册的具体实现的接口，会在New函数注入，后者是整个cache模块的参数结构

参数选项一般会有一个结构是作为默认参数的，并且会提供所有参数的选项函数设置

    var (
        DefaultTTL = time.Minute
    )


```registry/cache/options.go```

    // TTL参数的选项函数
    func WithTTL(t time.Duration) Option {
        return func(o *Options) {
            o.TTL = t
        }
    }

当然了，如果有n个函数，那么就会有n个参数选项函数

    //cache的实例化
    func New(r registry.Registry, opts ...Option) Cache {
        rand.Seed(time.Now().UnixNano())
        options := Options{
            TTL: DefaultTTL,
        }

        for _, o := range opts {
            o(&options)
        }

        return &cache{
            Registry: r,
            opts:     options,
            watched:  make(map[string]bool),
            cache:    make(map[string][]*registry.Service),
            ttls:     make(map[string]time.Time),
            exit:     make(chan bool),
        }
    }

这里主要是New出cache，但是返回的是接口对象，注意，在go中小写是不对外暴露的，cache的可见域实质只是在cache包中，但是因为其实现了Cache接口，因此外面New出的cache对象可以调用Cache接口，达到业务依赖抽象而不是依赖实现。

对参数进行默认初始化，也可以在options.go文件中初始化一个默认的参数结构，然后替换这里的赋值方式

    options := Options{
        TTL: DefaultTTL,
    }

这里主要是对传入的所有参数选项函数进行遍历调用，注意，传入的是前面创建的options的指针，因此遍历完传入的所有参数选项函数后，就会对options各个对应的参数赋值。这就是参数选项模式的原理

    for _, o := range opts {
        o(&options)
    }

参数选项模式的原理：定义一个参数结构，定义一个参数选项函数类型，初始化一个默认参数的参数结构对象，在New的时候传入参数选项函数列表，遍历参数选项函数列表的同时，传入参数结构对象，达到对应选项参数赋值对应参数的目的。

看下面，就是一个最小的例子：

```registry/cache/cache.go```

    type Cache interface {
        registry.Registry
        Stop()
    }

    type Options struct {
        Name string
        TTL time.Duration
    }

    type Option func(o *Options)

    type cache struct {
        registry.Registry
        opts Options
        ...
    }

    var (
        DefaultOptions = Options {
            Name: "cache",
            TTL: time.Minute,
        }
    )

    func New(r registry.Registry, opts ...Option) Cache {
        rand.Seed(time.Now().UnixNano())
        options := DefaultOptions
        for _, o := range opts {
            o(&options)
        }

        return &cache{
            Registry: r,
            opts:     options,
            watched:  make(map[string]bool),
            cache:    make(map[string][]*registry.Service),
            ttls:     make(map[string]time.Time),
            exit:     make(chan bool),
        }
    }

```registry/cache/options.go```

    func WithTTL(t time.Duration) Option {
        return func(o *Options) {
            o.TTL = t
        }
    }

    func WithName(name string) Option {
        return func(o *Options) {
            o.Name = name
        }
    }


外部调用方式：

    c := cache.New(
        r,
        cache.WithTTL(time.Minute*5),
        cache.WithName("test-name"),
    )


## 插件化
```registry/registry.go```

    type Registry interface {
        Init(...Option) error
        Options() Options
        Register(*Service, ...RegisterOption) error
        Deregister(*Service) error
        GetService(string) ([]*Service, error)
        ListServices() ([]*Service, error)
        Watch(...WatchOption) (Watcher, error)
        String() string
    }

所有实现了Registry接口的插件都可以接入go-micro，作为服务发现注册中心

    DefaultRegistry = NewRegistry()

    type Option func(*Options)
    type RegisterOption func(*RegisterOptions)
    type WatchOption func(*WatchOptions)

一连串的这个，应该一下子就知道又是**函数选项模式**，在go-micro中，函数选项模式随处可见，是可以插件化的基石。

```registry/options.go```

    type Options struct {
        Addrs     []string
        Timeout   time.Duration
        Secure    bool
        TLSConfig *tls.Config
        Context context.Context
    }

    type RegisterOptions struct {
        TTL time.Duration
        Context context.Context
    }

    type WatchOptions struct {
        Service string
        Context context.Context
    }

    func Addrs(addrs ...string) Option {
        return func(o *Options) {
            o.Addrs = addrs
        }
    }

    func Timeout(t time.Duration) Option {
        return func(o *Options) {
            o.Timeout = t
        }
    }

    func Secure(b bool) Option {
        return func(o *Options) {
            o.Secure = b
        }
    }

    func TLSConfig(t *tls.Config) Option {
        return func(o *Options) {
            o.TLSConfig = t
        }
    }

    func RegisterTTL(t time.Duration) RegisterOption {
        return func(o *RegisterOptions) {
            o.TTL = t
        }
    }

    func WatchService(name string) WatchOption {
        return func(o *WatchOptions) {
            o.Service = name
        }
    }

以上是**registry模块**的所有参数结构的**参数选项函数**

## 总结
开源项目是学习大神的好入口，透过源码可以学习到优秀编码技巧和风格。写出高质量的代码，更具有拓展性，维护性，可读性。








