---
layout:     post                  
title:      nsq
subtitle:   nsq-go实战
date:       2019-05-27
author:     Liangjf                  
header-img: img/post-bg-coffee.jpeg
catalog: true                      
tags:                       
    - golang
---

# nsq-go实战

## nsq架构
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_nsq_jaigou.png?raw=true)


## 1.nsqd启动
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_nsq_nsqd.png?raw=true)

### nsqd的作用
接收消息，排队，转发的作用。用于生产者的解耦，消息的缓存，构建集群。

监听两个端口，默认是4150（TCP）和4151（HTTP）。

- 两个端口都提供给生产者和消费者连接。所有有两种通信方式
- http端口还提供给nsqadmin获取该nsqd本地信息（为了web监控服务）

### nsqd做两件事：
- 1、当生产生发布消息，若topic是第一次发送，那么就会在nsqd中为这个topic创建一个通道作为内部消息队列来传输生产者到nsqd的数据。起到解耦的作用。

- 2、有新topic时，向nsqlookup注册topic，待消费者通过查询nsqlookup来得到topic的更新，进一步得到topic更新对应的nsqd，然后消费者就会拿到此nsqd的地址，与此nsqd建立连接，拿到topic的数据。

## 2.nsqlookup启动
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_nsq_nsqlookup.png?raw=true)

### nsqlookup的作用
协调，发现服务，维护最终一致性。

监听两个端口，证明有两种方式通信，默认是**4160（TCP）**和**4161（HTTP）**。

- **4160**是nsqd与nsqlookup的连接端口，用于接收nsqd的广播，用于nsqd的注册；
- **4161**是接收客户端发送的管理和发现操作请求，为nsqadmin提供集群信息。

每个生产者和消费者都会随机监听一个端口来产生一条建立作为交互。

nsqd有新主题时，会在nsqlookup中注册，并且可以与消费者建立连接，消费者定时轮询nsqlookup查询订阅的topic是否有新数据。

当消费者与nsqlookup建立连接，但是对应的topic还没有消息，消费者会通过GET请求来定时查询。

## 3.nsqadmin UI
nsqadmin通过http 4161端口（消费者get请求的端口）来收集nsq数据，监控数据，通过web（4171端口）展示系统的情况。
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_nsq_adminUI.png?raw=true)

![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_nsq_adminUI2.png?raw=true)


## 4.生产者
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_nsq_product.png?raw=true)

生产者发送消息的时候，需要标明发送的**topic**，连接的**nsqd地址**。一个生产者连接一个nsqd，其实nsqd就是一个中介，生产者通过nsqd来达到消息的发送，达到解耦，可靠的目的。

## 5.消费者
![](https://github.com/liangjfblue/liangjfblue.github.io/blob/master/img/post_nsq_consumer.png?raw=true)

消费者有两种连接方式：
- 1、直连nsqd
- 2、连接nsqlookup

第一种，消费者直接从nsqd（队列中取得数据）。相当于经典的消费者生产者模型，仅仅把nsqd作为一个队列来使用。缺点是，连接断了，那就真的断了，需要自己保证可靠性。

第二种，消费者通过连接nsqlookup，从nsqlookup中发现订阅的topic的数据更新。优点是，通过nsqlookup来保证可靠性，即使连接断了。但是nsqlookup和nsqd会保证总有一条连接，即保证消息总会被消费一次（除非nsqd全都挂了）

看下面的例子：

#### 生产者
`src\myselft\product\product.go`

	package main
	
	import (
		"fmt"
		"strconv"
	
		"github.com/nsqio/go-nsq"
	)
	
	var (
		tcpNsqdAddrr = "127.0.0.1:4150"
	)
	
	func main() {
		config := nsq.NewConfig()
	
		tPro, err := nsq.NewProducer(tcpNsqdAddrr, config)
		if err != nil {
			fmt.Println(err)
		}
		topic := "test"
		tCommand := "new data!" + " " + strconv.Itoa(i)
	
		err = tPro.Publish(topic, []byte(tCommand))
		if err != nil {
			fmt.Println(err)
		}
	
	}


#### 消费者
`src\myselft\consumer\consumer.go`

	package consumer
	
	import (
		"fmt"
		"time"
	
		"github.com/nsqio/go-nsq"
	)
	
	type CallBackFunc func(msg *nsq.Message) bool
	
	type ConsumerHandle struct {
		Cb CallBackFunc
	}
	
	func (h *ConsumerHandle) HandleMessage(msg *nsq.Message) error {
		h.Cb(msg)
		return nil
	}
	
	type Consumer struct {
		Handle         ConsumerHandle
		TopicName      string
		Channel        string
		NsqLookupdAddr string
		NsqAddr        string
	}
	
	func (c *Consumer) Init() error {
		config := nsq.NewConfig()
		config.LookupdPollInterval = time.Second
		consumer, err := nsq.NewConsumer(c.TopicName, c.Channel, config)
		if err != nil {
			panic(err)
		}
	
		consumer.SetLogger(nil, 0)
	
		handle := ConsumerHandle{
			Cb: c.Handle.Cb,
		}
		consumer.AddHandler(&handle)
	
		//连接nsqlookup，自动发现nsq。
		err = consumer.ConnectToNSQLookupd(c.NsqLookupdAddr)
		if err != nil {
			fmt.Println(err)
		}
	
		//直连nsq
		// err = consumer.ConnectToNSQD(c.NsqAddr)
		// if err != nil {
		// 	fmt.Println("consumer.ConnectToNSQD error...")
		// }
	
		fmt.Println("consumer.ConnectToNSQD ok ..")
		return nil
	}
	
	func (c *Consumer) Run() {
		for range time.NewTicker(time.Second * 10).C {
			fmt.Printf("Consumer is run   %v...\n", time.Now().Format("2006-01-02 03:04:05"))
		}
	}


消费者的main.go。`src\myselft\test_nsq\main.go`

	package main
	
	import (
		"fmt"
		"myselft/consumer"
		"time"
	
		"github.com/nsqio/go-nsq"
	)
	
	func handle(msg *nsq.Message) bool {
		fmt.Printf("msg.msg=%v,  msg=%s \n", time.Unix(0, msg.Timestamp).Format("2006-01-02 03:04:05"), string(msg.Body))
		return true
	}
	
	func main() {
		handle222 := consumer.ConsumerHandle{
			Cb: handle,
		}
	
		consumer := consumer.Consumer{
			TopicName:      "test",
			Channel:        "channel_nsq",
			NsqLookupdAddr: "127.0.0.1:4161", //方式1：连接nsqlookup
			NsqAddr:        "127.0.0.1:4151", //方式2：直连nsq
			Handle:         handle222,
		}
	
		consumer.Init()
		consumer.Run()
	
		select {}
	}

