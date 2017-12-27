---
layout: post
title:  "rabbitmq 延迟队列实现"
date:   2017-12-27 18:46:55
tagline: "Supporting tagline"
tags : Java,Spring,rabbitmq
published: true
categories: 技术分享
---
# rabbitmq 延迟队列实现
---
message 先发送到 delayExchange，delayExchange 接收后，根据 binding 的 delayQueue 的 ttl 时间，到期后，再通过 x-dead-letter-exchange 设置的 exchange，把消息发送到 businessExchange，businessExchange 的 businessQueue 接收到消息后就可以实现具体的业务处理。   

**延迟队列定义：**   

``` xml
<!-- 延时队列 -->
<rabbit:queue name="delayQueue" > 
<rabbit:queue-arguments> 
	<!--<entry key="x-message-ttl">--> 		
	<!--<value type="java.lang.Long">10000</value>--> 	
	<!--</entry>--> 	
	<entry key="x-max-length"> 		
	<value type="java.lang.Long">10000000</value> 	
	</entry> 	
	<entry key="x-dead-letter-exchange" value="businessExchange"/> 
</rabbit:queue-arguments> 
</rabbit:queue> 
<rabbit:topic-exchange name="delayExchange" > 
	<rabbit:bindings> 
			<rabbit:binding queue="delayQueue" pattern="#" /> 
	</rabbit:bindings> 
</rabbit:topic-exchange>  

<!-- 具体逻辑处理队列 --> 
<rabbit:queue name="businessQueue"/> 
<rabbit:topic-exchange name="businessExchange" > 
	<rabbit:bindings> 		
		<rabbit:binding queue="businessQueue" pattern="#" /> 	
	</rabbit:bindings> 
</rabbit:topic-exchange> 
<rabbit:listener-container 		connection-factory="connectionFactory" concurrency="2" prefetch="2"> 
	<rabbit:listener ref="businessQueueListener" 					 method="listen" queue-names="businessQueue" /> 
</rabbit:listener-container>
```

"x-message-ttl" 为队列默认的过期时间，如果消息也设置了ttl，默认使用用两个之中最小的时间。

java 发送自定义过期时间，delayTime 默认为毫秒；由于消息队列的特性是先进先出，先进队列的消息如果没有到期或者被消费，后面的消息即使到期也是需要等待，因此该自定义时间有可能出现同时等待处理完毕队列头部长时间后，才被处理：   

**自定义延时消息发送定义**

``` java
rabbitTemplate.convertAndSend(exchange, routingKey, messageObject, new MessagePostProcessor() { 	
	@Override 	
	public Message postProcessMessage(Message message) throws AmqpException { 		
		message.getMessageProperties().setExpiration(String.valueOf(delayTime)); 
		return message; 	
	} 
});
```
