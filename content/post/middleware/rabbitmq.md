---
title: rabbitmq
date: '2025-12-28T09:38:23+08:00'

categories:
- 中间件
---

rabbitmq 三种发送信息方式 **director**，**topic**，**fanout**

**注解式**

（direct）1:1,（fanout）1：N,（topic）N:1

　　direct： 1:1类似完全匹配

　　fanout：1：N 可以把一个消息并行发布到多个队列上去，简单的说就是，当多个队列绑定到fanout的交换器,那么交换器一次性拷贝多个消息分别发送到绑定的队列上，每个队列有这个消息的副本。

　　　　　ps：这个可以在业务上实现并行处理多个任务，比如，用户上传图片功能，当消息到达交换器上，它可以同时路由到积分增加队列和其它队列上，达到并行处理的目的，并且易扩展，以后有什么并行任务的时候，直接绑定到fanout交换器不需求改动之前的代码。

　　topic  N:1 ，多个交换器可以路由消息到同一个队列。根据模糊匹配，比如一个队列的routing key 为*.test ，那么凡是到达交换器的消息中的routing key 后缀.test都被路由到这个队列上。





yml配置文件

```yml
spring:
    rabbitmq:
        host: 172.16.25.234
        username: guest
        password: guest
        virtual-host: /
mq:
    direct:
        exchange:
            direct
        queue:
            log.info
        routingKey:
            routing.key
    topic:
        exchange:
            topic
    fanout:
        exchange:
            fanout
```





==director==

```java
@Component
public class DirectSend {

    @Autowired
    private AmqpTemplate amqpTemplate;

    @Value("${mq.direct.exchange}")
    private String directExchange;

    @Value("${mq.direct.routingKey}")
    private String routingKey;

    public void send(String msg) {
        amqpTemplate.convertAndSend(directExchange, routingKey, msg);
    }

}
```

```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue(value = "${mq.direct.queue}", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.direct.exchange}", type = ExchangeTypes.DIRECT),
        key = "${mq.direct.routingKey}"))
public class DirectReceive {


    @RabbitHandler
    public void directReceive(String msg) {
        System.out.println("direct receive "+msg);
    }

}
```





==topic==

```java
@Component
public class TopicSend {


    @Autowired
    private AmqpTemplate amqpTemplate;

    @Value("${mq.topic.exchange}")
    private String exchange;

    public void topicSend(String msg){
        amqpTemplate.convertAndSend(exchange,"user.log.info",msg);
    }

}
```

```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue(value = "log.debug", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.topic.exchange}", type = ExchangeTypes.TOPIC),
        key = "*.log.debug"
))
public class TopicDebugReceive {

    @RabbitHandler
    public void topicDebugReceive(String msg) {
        System.out.println("topic debug receive " + msg);
    }

}
```

```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue(value = "log.error", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.topic.exchange}", type = ExchangeTypes.TOPIC),
        key = "*.log.error"
))
public class TopicErrorReceive {


    @RabbitHandler
    public void topicErrorReceive(String msg) {
        System.out.println("topic error receive " + msg);
    }
}
```

```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue(value = "log.info", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.topic.exchange}", type = ExchangeTypes.TOPIC),
        key = "user.log.info"
))

public class TopicInfoReceive {


        @RabbitHandler
        public void topicInfoReceive(String msg) {
            System.out.println("topic info receive " + msg);

    }
}
```





==fanout==

```java
@Component
public class FanoutSend {

    @Autowired
    private AmqpTemplate amqpTemplate;

    @Value("${mq.fanout.exchange}")
    private String exchange;

    public void fanoutSend(String msg) {
        amqpTemplate.convertAndSend(exchange, "", msg);
    }
}
```

```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue(value = "log.push", autoDelete = "true"),
        exchange = @Exchange(value = "${mq.fanout.exchange}", type = ExchangeTypes.FANOUT)

))
public class FanoutPushReceive {

    @RabbitHandler
    public void pushReceive(String msg) {
        System.out.println("fanout push receive " + msg);
    }
}
```

```java
@Component
@RabbitListener(bindings = @QueueBinding(value = @Queue(value = "log.sms",autoDelete = "true"),
        exchange = @Exchange(value = "${mq.fanout.exchange}",type = ExchangeTypes.FANOUT)

))
public class FanoutSmsReceive {


    @RabbitHandler
public void smsReceive(String msg){
    System.out.println("fanout sms receive "+msg);
}

}
```