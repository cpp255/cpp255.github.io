---
layout: post
title:  "Thread"
date:   2017-12-27 18:46:55
tagline: "Supporting tagline"
tags : Java,源码阅读
published: true
categories: 技术分享
---

#Thread 实现
-------
需要等待前置线程处理完后，才允许当前线程执行或者结束，可以有不同的实现方式，join 方法是其中之一。

```java
// Waits for this thread to die (等待发起调用方法的这个线程死亡)
public final void join() throws InterruptedException {  
   join(0);  // 0 表示一直等待  
}
```

```java
/**
* Waits at most {@code millis} milliseconds for this thread to * die. A timeout of {@code 0} means to wait *forever.
* 等待最多 millis 时间该线程结束，否则超时则继续运行。0 表示一直运行等待线程结束。
**/
public final synchronized void join(long millis) throws InterruptedException {  
   long base = System.currentTimeMillis();     
   long now = 0;      
   if (millis < 0) {         
       throw new IllegalArgumentException("timeout value is negative");     
   }      

   if (millis == 0) {         
       while (isAlive()) { // 0 表示只要当线程满足 isAlive 则循环等待     
               wait(0);     
        }     
    } else {     
        while (isAlive()) {         
            long delay = millis - now;       
            if (delay <= 0) {   // 时间到后，结束等待，终止循环
                break;       
            }            
             wait(delay);             
             now = System.currentTimeMillis() - base;         
        }     
    }
 }
```

还有另一个方法 join(long millis, int nanos) 多了一个参数，表示join() 的时间为： millis + nanos; 最后还是根据输入值换算成 join(long millis)

```java
public final synchronized void join(long millis, int nanos) throws InterruptedException {   
   if (millis < 0) {        
        throw new IllegalArgumentException("timeout value is negative");     
    }      

    if (nanos < 0 || nanos > 999999) { // 超过了 1 ms 或者 负值不满足条件        
        throw new IllegalArgumentException( "nanosecond timeout value out of range");     
    }      

    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {  // 四舍五入或者最小单位为 1 ms         
        millis++;     
    }      

    join(millis); 
}
```

执行 sleep 会使当前线程停止执行给定的时间，但是并不会放弃锁。锁的控制应该是由具体的锁操作，而不能通过线程操作。

``` java
public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {  // 纳秒数量不对
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && millis == 0)) { // 最少是一毫秒，或者四舍五入加一毫秒
            millis++;
        }

        sleep(millis);
    }
```    

