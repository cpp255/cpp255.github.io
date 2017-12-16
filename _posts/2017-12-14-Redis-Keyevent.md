---
layout: post
title:  "Redis 事件监听"
date:   2017-12-14 12:20:20
tagline: "Supporting tagline"
tags : Redis,Java
published: true
categories: 技术分享
---
# Redis 事件监听
---

2.8 以后的版本开始支持事件监听，由于开启会消耗CPU资源，默认关闭。  
开启事件支持有两种方式：服务端配置和客户端配置；服务端的配置是永久有效的，客户端的配置重启后会失效。服务端开启事件支持后，客户端就可以开始订阅服务端推送的事件监听。   
  
**事件类型**   
```
K     Keyspace events, published with __keyspace@<db>__ prefix.
E     Keyevent events, published with __keyevent@<db>__ prefix.
g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
$     String commands
l     List commands
s     Set commands
h     Hash commands
z     Sorted set commands
x     Expired events (events generated every time a key expires)
e     Evicted events (events generated when a key is evicted for maxmemory)
A     Alias for g$lshzxe, so that the "AKE" string means all the events.
```

1.服务端在 redis.conf 中打开事件监听配置，再重启redis服务，这里使用的是接收所有类型，就是"A"，具体使用的话可以随意替换类型；注意启动的时候是否使用当前目录修改的 redis.conf     

``` 
notify-keyspace-events "EA"
```

重启 redis： 
  
``` shell
./src/redis-server redis.conf
```

2. 客户端

```
 redis-cli config set notify-keyspace-events KEA
```

如果只监听删除、过期和因为内存过大被删除的，则模式可以选择为： Egxe

**客户端接收订阅事件**   

```
psubscribe '__keyevent*__:*'  // 订阅所有事件
psubscribe __keyevent@0__:del  // 订阅数据库0的del事件
psubscribe __keyevent@0__:expired  // 订阅数据库0的expired事件
```

jedis 接收订阅事件，继承 JedisPubSub 类，在实现的方法里面接收处理的逻辑:

``` java
// 接收订阅事件
public void onPMessage(String pattern, String channel, String message) {
}
```

注册订阅事件，pattern 跟客户端订阅事件一样，可以添加不同的订阅事件，也可以添加多个事件，例如：__keyevent@0__:del；注册订阅事件是阻塞操作，建议单独生成另一个线程。       

``` java
Jedis jedis;
jedis.psubscribe(jedisPubSub, pattern);
```

参考：
1 http://redisdoc.com/topic/notification.html   
2 https://redis.io/topics/notifications

