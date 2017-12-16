---
layout: post
title:  "Jedis 订阅事件阻塞"
date:   2017-12-15 12:20:20
tagline: "Supporting tagline"
tags : Redis,Java,Spring
published: true
categories: 技术分享
---

# Jedis 订阅事件阻塞
Jedis 开启订阅事件的时候是这样的： 
  
``` java
public void psubscribe(final JedisPubSub jedisPubSub,     final String... patterns) { checkIsInMulti(); connect(); // 连接server，底层是通过socket进行连接 client.setTimeoutInfinite(); jedisPubSub.proceedWithPatterns(client, patterns);  // 主要是这里 client.rollbackTimeout();    }
```

调用JedisPubSub的代码，process() 方法里面是一个 do while 循环，所以这里肯定会阻塞，只能通过开辟另一个线程进行订阅，否则整个程序就会被阻塞在这里进行循环： 

``` java
public void proceedWithPatterns(Client client, String... patterns) { this.client = client;
// 发送订阅指令，类似上面的 psubscribe '__keyevent*__:*' client.psubscribe(patterns);  client.flush(); process(client); // 实时处理返回的指令信息 }


private void process(Client client) {  do {
		// 取得响应回来的数据     List<Object> reply = client.getRawObjectMultiBulkReply();     final Object firstObj = reply.get(0);     if (!(firstObj instanceof byte[])) { 	throw new JedisException("Unknown message type: " + firstObj);     }     final byte[] resp = (byte[]) firstObj;     if (Arrays.equals(SUBSCRIBE.raw, resp)) { 	subscribedChannels = ((Long) reply.get(2)).intValue(); 	final byte[] bchannel = (byte[]) reply.get(1); 	final String strchannel = (bchannel == null) ? null 		: SafeEncoder.encode(bchannel); 	onSubscribe(strchannel, subscribedChannels);     } else if (Arrays.equals(UNSUBSCRIBE.raw, resp)) { 	subscribedChannels = ((Long) reply.get(2)).intValue(); 	final byte[] bchannel = (byte[]) reply.get(1); 	final String strchannel = (bchannel == null) ? null 		: SafeEncoder.encode(bchannel); 	onUnsubscribe(strchannel, subscribedChannels);     } else if (Arrays.equals(MESSAGE.raw, resp)) { 	final byte[] bchannel = (byte[]) reply.get(1); 	final byte[] bmesg = (byte[]) reply.get(2); 	final String strchannel = (bchannel == null) ? null 		: SafeEncoder.encode(bchannel); 	final String strmesg = (bmesg == null) ? null : SafeEncoder 		.encode(bmesg); 	onMessage(strchannel, strmesg);     } else if (Arrays.equals(PMESSAGE.raw, resp)) { 	final byte[] bpattern = (byte[]) reply.get(1); 	final byte[] bchannel = (byte[]) reply.get(2); 	final byte[] bmesg = (byte[]) reply.get(3); 	final String strpattern = (bpattern == null) ? null 		: SafeEncoder.encode(bpattern); 	final String strchannel = (bchannel == null) ? null 		: SafeEncoder.encode(bchannel); 	final String strmesg = (bmesg == null) ? null : SafeEncoder 		.encode(bmesg); 	onPMessage(strpattern, strchannel, strmesg);     } else if (Arrays.equals(PSUBSCRIBE.raw, resp)) { 	subscribedChannels = ((Long) reply.get(2)).intValue(); 	final byte[] bpattern = (byte[]) reply.get(1); 	final String strpattern = (bpattern == null) ? null 		: SafeEncoder.encode(bpattern); 	onPSubscribe(strpattern, subscribedChannels);     } else if (Arrays.equals(PUNSUBSCRIBE.raw, resp)) { 	subscribedChannels = ((Long) reply.get(2)).intValue(); 	final byte[] bpattern = (byte[]) reply.get(1); 	final String strpattern = (bpattern == null) ? null 		: SafeEncoder.encode(bpattern); 	onPUnsubscribe(strpattern, subscribedChannels);     } else { 	throw new JedisException("Unknown message type: " + firstObj);     } } while (isSubscribed());  /* Invalidate instance since this thread is no longer listening */ this.client = null;  /*  * Reset pipeline count because subscribe() calls would have increased  * it but nothing decremented it.  */ client.resetPipelinedCount();    }
```

设置一个线程单独运行该订阅事件：   

``` java
ExecutorService executor = Executors.newSingleThreadExecutor(); Runnable subRunnable = new Runnable() { 	@Override 	public void run() { 		// TODO 添加订阅事件处理 	} }; executor.execute(subRunnable);
```


