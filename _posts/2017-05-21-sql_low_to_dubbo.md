---
layout: post
title:  "sql 引发的 dubbo 超时"
date:   2017-05-21 23:27:10
tagline: "Supporting tagline"
tags : dubbo,mysql,java
published: true
categories: 技术分享
---
# sql 引发的 dubbo 超时
---
最近在对已有项目进行分离，用了 dubbo，分离之后发现有个接口请求超时了，Google 后找到：http://www.cnblogs.com/binyue/p/5380322.html，文章说有两个解决办法：      
1. 找出耗时最多的方法，调优。
2. 加大超时时间。
 

跑代码，调试后发现，一般跑一次接口调用运行查询时间都为 4s+ 了，肯定超时。   
解决办法：
1.加大 dubbo 的超时时间：   
``` xml
<dubbo:reference id="xxDubboService"                  interface="com.xx.service.xxService" version="1.0"                  group="annotationConfig" protocol="dubbo" check="false" timeout="5000"/>

```

dubbo 默认超时时间(timeout)为1s，增加为5s.这时测试已经可以运行，只是等待时间会比较长。

2.调优   
不满足，继续优化，接口代码中有7次 sql 查询，第一次的 sql 查询出一个列表，列表长度为30，然后一个for循环，每一次循环内部有6次sql查询，打印时间，发现时间都耗在了这个for循环的sql执行上。   
查看sql代码，sql 的查询都是必要的操作，删除不了。其中，有3个sql操作是通过查询3个表，循序的通过范式获取数据，返回给java，通过在java层进行数据调用，然后再继续调用sql查询；并且每次查询都是获取所有的数据，然后只取了其中的一个属性。 
##### 优化--合并sql，消减获取列
把上面提到的3个sql合并成了一个，使用join查询，降低连接次数，并且把sql查询结果集缩小，只收集需要的数据。   
这次结果效果惊人，查询时间减到了2s以下。这种合并其实还有一个好处，spring 注入生成的dao少了，连接需要的线程少了，jvm 的内存申请也少。
##### 优化--前端把分页值调低 
分页毕竟没有硬性要求，把前端请求的分页大小改小，之前的为30改为20，时间再缩小到1.2～1.5s左右，可以接受。   

##### 优化--TODO
后期如果继续优化，可以试着对索引方向着手，估计可以再减一两百毫秒。这个业务使用场景并不高，这些工作暂时还没有继续往下深挖，有空了可以继续搞(估计就瞎扯，项目忙起来没有头).

### 总结
分拆成几个子任务在我这个例子中，已经明显消耗太多时间了，再通过这么一个for循环，查询连接，时间消耗更大。但是几个表用 join 的方式消除一些子任务，写法还是得用 EXPLAIN 查看一下，看下查询方式是否得不偿失。
