---
layout:     post
title:      为Kafka消息添加MessageID
subtitle:   在不破坏消息体的情况下为每个消息添加消息ID
date:       2018-12-06
author:     Cadmean
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - Kafka
    - Client
---

# 简介
之前提过我在公司内部做的是一个消息平台，为各个应用提供消息服务。前端时间有个领导给了个需求，能不能自动为所有的消息加上一个消息ID的属性，可以全局唯一定义一个消息。

听到时候的第一个反应是，消息体的序列化反序列化方法都可以自定义了，为什么不在自己的消息体里加MessageID？

领导的说法是因为这样所有客户端在使用的时候还要修改自己的消息体，多不方便啊

emmmmmm........

![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1544097662973&di=50d82dab0bf3598943a07e44114c733f&imgtype=0&src=http%3A%2F%2Fwww.17qq.com%2Fimg_biaoqing%2F38591974.jpeg)

然后我转念一想，在不破坏消息体的前提下，给每个消息添加一个消息ID，感觉很有趣啊，于是报着干脆玩一下的心理预期，开始干活。

# 基本思路

在Kafka客户端中，会根据用户选择的序列化方式实例化序列化器（Serializer）和反序列化器（Deserializer），要实现这个功能，基本思路是在用户定义的序列化方式上包上一层自己的序列化方法。

最终实现的效果是：
```java
//Producer Demo
public class ProducerDemo{
    @Test
    public void sendTest(){
        KafkaConfigProperties prop = new KafkaConfigProperties("producer.properties");
        ABCProducer<String,ABCMessage<String>> producer = new ABCProducer<>(prop);
        ProducerRecord<String,ABCMessage<String>> record = new ProducerRecord<String,ABCMessage<String>>("testTopic",null,new ABCMessage<String>("hello world!"));
        producer.send(record);
    }
}

```

```java
//Consumer Demo
public class ConsumerDemo{
    @Test
    public void receiveTest(){
        KafkaConfigProperties prop = new KafkaConfigProperties("consumer.properties");
        ABCConsumer<String,ABCMessage<String>> consumer = new ABCConsumer<>(prop);
        consumer.subscribe(Collections.singletonList("testTopic"));
        
        ConsumerRecords<String,ABCMessage<String>> records = consumer.poll(10000);
        for (ConsumerRecord<String,ABCMessage<String>> record:records){
            
            System.out.println(record.getMessageId()+" "+record.getvalue());
        }
        
    }
}
```

而配置的配置文件仍然可以是原生的那些
```properties
bootstrap.servers=192.168.0.1:9092
key.serializer=org.apache.kafka.common.serialization.StringSerializer
value.serializer=org.apache.kafka.common.serialization.StringSerializer
acks=all
```

究其原因，就是在通过对原生接口的修改，侵入了原生的kafka-client的api，本文只实现了在用户序列化方式外面包上一层，其实还可以实现很多其他的能力。

## 序列化位置的选择
对一个kafka消息来说，ProducerRecord里写死了很多属性：
```java
public class ProducerRecord<K, V> {
    private final String topic;
    private final Integer partition;
    private final K key;
    private final V value;
    private final Long timestamp;
}
```
这些类型都是private final的，其中topic，partition都是用户指定的，不能修改，key直接决定了patition，可能有业务含义，timestamp和value是可以稍微动一下的，但是timestamp是long类型，不好修改，最终message ID还是要加在value上。

value这个类型是用户指定的，且指定了序列化方法，在发送的时候发送的都是byte[]类型，所以考虑在用户指定的序列化器做完序列化以后，在前面加上自己的序列后的messageID，重新组合成一个新的byte[]再发往kafka。

整个过程是这样的：
![侵入序列化方法](https://upload-images.jianshu.io/upload_images/3320837-618f32b9985e1dfc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 设计思路
我们需要自定义properties（KafkaConfigProperties），自定义producer（ABCProducer），自定义一个新的Message类型（ABCMessage），自定义一个序列化和反序列化方式（ABCMessageSerializer，ABCMessageDeserializer）。

每个组件的用途是这样的：
1. KafkaConfigProperties ：继承Properties，用于从用户给的properties中进行value.serializer的修改，此外还能加上一些强制鉴权的能力、参数校验等等。。
2. ABCProducer：继承KafkaProducer，用于在父类KafkaProducer初始化以后，将用户定义的序列化器传入ABCMessageSerializer
3. ABCMessage：包装上MessageID，给生产者和消费者对等的消息发送和接收的概念
4. ABCMessageSerializer：对数据进行拆分，并将消息的message ID进行基本序列化，将用户的value根据用户的序列化方式进行序列化。

这样设计的原因是，kafka的客户端封装得太紧密，所以找了半天没找到漏洞，最后只能通过反射来破坏KafkaProducer中的valueSerializer的private修饰，从而在子类中对其进行方法调用。

# 实现代码

### KafkaConfigProperties
根据配置文件，判断是否需要启动这个功能。如果启动，就更改一下value.serializer配置或者value.deserializer配置
```java
public class KafkaConfigProperties extends Properties {
    private boolean getBooleanProperties(String key, String defaultValue){
        return Boolean.valueOf(this.getproperty(key,defaultValue));
    }
    
    public KafkaConfigProperties(Properties prop){
        super();
        this.putAll(prop);
        configMessageId();
    }
    
    private boolean messageIdEnable;
    
    public boolean isMessageIdEnable() {
        return messageIdEnable;
    }
    
    private void configMessageId() {
        //默认不使用messageID，需要显式配置
         messageIdEnable = getBooleanProperties("messageid.enable","false");
        if (messageIdEnable){
            if (this.containsKey(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG)){
                this.put("user.value.serializer",this.get(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG));
                this.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,"com.abc.ABCMessageSerializer");
            }
            if (this.containsKey(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG)){ 
                this.put("user.value.deserializer",this.get(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG));
                this.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,"com.abc.ABCMessageDeserializer");
                
            }
        }
        
    }
}
```

### ABCMessage

每次Mesasge的生成都会通过MessageUtils来自动生成一个messageId，建议固定长度，我用的暂时是UUID，36位。

```java
public class ABCMessage<V> {
    private String messageId;
    private V value;
    public ABCMessage(String messageId, V value) {
        this.messageId = messageId;
        this.value = value;
    }
    public ABCMessage(V value) {
        this(MessageUtils.generateMessageId(),value);
    }
    public String getMessageId() {
        return messageId;
    }
    public void setMessageId(String messageId) {
        this.messageId = messageId;
    }
    public V getValue() {
        return value;
    }
    public void setValue(V value) {
        this.value = value;
    }
}

```

### ABCProducer

这里用到了反射，将父类中的valueSeriazlier获取到，然后执行里面的方法

```java
public class ABCProducer<K,V> extends KafkaProducer<K,V>{
    public ABCProducer(KafkaConfigProperties prop) {
        super(prop);
        if (prop.isMessageIdEnable()){
            Field field = this.getClass().getSuperClass().getDeclareField("valueSerializer");
            field.setAccessible(true);
            ABCMessageSerializer serializer = (ABCMessageSerializer) field.get(this);
            serializer.setUserSerializer(prop.get("user.value.serializer").toString());
        }
     }
}
```

### ABCMessageSerializer

```java
public class ABCMessageSerializer implements Serializer {
    private Serializer valueSerializer;
    private Map configs;
    
    @Override
    public void configure(Map configs,boolean isKey){
        this.configs = configs;
    }
    
    @Override
    public byte[] seriazlize(String topic, Object data){
        if (data instanceof ABCMessage){
            byte sep = 0;
            ABCMessage message = (ABCMessage) data;
            byte[] joinByte = EncryptUtils.join(sep,message.getMessageId().getBytes("UTF-8"),
            valueSerializer.serialize(topic,message.getValue()));
            return joinByte;
        }
    } 
    
    @Override
    public void close(){}
    
    public void setUserSerializer(String userValueSerializer){
        HashMap<String,Object> config = new HashMap<>();
        config.put("value.serializer",Class.forName(userValueSerializer));
        ABCMessageConfig config = new ABCMessageConfig(configProp);
        this.valueSerializer = config.getConfiguredInstance("value.serializer",Serializer.class);
        this.valueSerializer.configure(this.configs,false);
    }
}

```

这段代码里有几个需要注意的地方：
1. serialize方法里，进行了一个messageId的序列化操作，message.getMessageId().getBytes("UTF-8")，建议加上UTF-8；
2. EncryptUtils.join这个方法是我前文中对kafka的鉴权模式进行自己修改的时候用到的，作用是字符串拼接。凭借的字符串之间加上分隔符也就是sep；
3. setUserSerializer方法中用到了ABCMessageConfig，这个Config继承自AbstractConfig这个抽象类，仅仅只是进程，没有任何的重写。主要用途是模仿ProducerConfig和ConsumerConfig中对valuerSeriazlier的实例化方式。这里就不上代码了。
4. 记得构造好以后的valueSerializer需要调用一下configure方法。

我在一开始的时候想在configure方法里加上指定用户的value.serializer，这样就可以免除后面调用setUserSerializer这个方法的麻烦，但是发现传入configure方法的configs经过了一层过滤（ProducerConfig构造后过滤了），我们在properties中加入的user.value.serializer无法被识别。所以只好用后续添加的方式。

