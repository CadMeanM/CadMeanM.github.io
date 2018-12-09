---
layout:     post
title:      ActiveMQ JMX增强
subtitle:   ActiveMQ JMX添加基础方法
date:       2018-11-25
author:     Cadmean
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - ActiveMQ
    - 消息中间件
    - JMX
---


前端时间学习了一下JMX的写法，相关的学习资料可以百度一下JMX学习。
日常最常用到JMX的地方是监控采集，往往开源项目也都会内置JMX的接口，供Jconsole连接或外部监控程序连接。
JMX的常用写法其实很简单：
```java
//注册server
MBeanServer server = ManagementFactory.getPlatformMBeanServer();
//绑定地址
LocateRegistry.createRegistry(1099);
JMXServiceURL jmxURL = new JMXServiceURL("service:jmx:rmi:////jndi/rmi://localhost:1099/jmxrmi");
JMXConnectorServer jcs = jMXConnectorServerFactory.newJMXConnectorServer(jmxURL,null,server)
//注册MBean
ObjectName objectName = new ObjectName("MyMBean:name=studentobj");
server.registerMBean(new Student()),object;
```

学习完以后，看了看ActiveMQ的JMX接口，这些接口MBean其实都放在com.apache.activemq.broker.jmx包下，一般来说，可以通过修改源码的方式来改JMX里的配置，但是改源码在版本更新的时候需要每次都从源码编译，很麻烦。

前面研究过ActiveMQ的插件（Interceptor，见[ActiveMQ插件开发实例-任务日志](https://www.jianshu.com/p/c73eb072daf9)），其实在ActiveMQ的插件里也可以注册JMX。

这样可以实现的能力就是，可以将业务的统计功能整合到ActiveMQ JMX中，而后通过采集程序采集，再进行上送。更方便的是开放的统计接口可以整合到我最近做的ActiveMQ集群监控工具上，实现一些定制化的业务能力。

而修改JMX的方式也很简单，还是使用之前的例子：
```java
package com.cn.amqs;
/**
 * 实现每次任务到达MQ时自动往一个地址上送一条信息
 * @author MisterCH
 */
public class MessageLog extends BrokerFilter{
    private Log log;
    /**下行任务HashMap*/
    private ConcurrentHashMap<Object, Integer> downWards;
    /**上行任务HashMap*/
    private ConcurrentHashMap<Object, Integer> upWards;
    private boolean isRegistered=false;
    public MessageLog(Broker next,String seviceUrl,String sign) {
        super(next);
        ……
    }
 /**
  * 每当MQ收到一条生产者发送过来的消息的时候执行判断。
  */
    public void send(ProducerBrokerExchange producerExchange, Message messageSend) throws Exception {
        if (!isRegistered){
            try{
                ObjectName obn= new ObjectName("com.demo.amqtest:type=Task, name=TaskList")l
                getBrokerService().getManagementContext().getMBeanServer().registerMBean(taskList,obn);
                isRegistered=true;
            } catch (Exception e){
                //处理无法注册MBean的问题
            }
        }
        //业务逻辑
        …………
    } 
}
```

定义了一个isRegistered的变量，用于判断是否已经注册过ObjectName，其实这一步最好是放在初始化这个类的时候，调用构造函数的时候。但是我试了下发现在构造函数中使用getBrokerService()这个方法会直接导致ActiveMQ启动报错，无法正常启动，猜测应该是在初始化插件时，BrokerService还没有初始化。所以只能在初始化后，调用某个方法的时候再进行注册，这段代码里虽然每次send的时候都会调用注册检查，但是O(1)的复杂度，应该可以接受，不会影响性能。

可以愉快地在集群管理工具上增加业务的统计了~

PS：KAFKA应该也可以，需要再研究下
